# Cloud Platform ERP — Deployment Problem Statement

> See [overview.md](./overview.md) for the current CI/CD setup. This document ignores the `beta` environment, which is being removed soon.

## The problem

The release flow is a **branch-driven promotion ladder**:

```
main (dev) ──► release ──► release-qa (qa) ──► release-prod (prod)
```

The unit of promotion is *everything on the branch* — not a single feature. This couples unrelated changes together, so you cannot release one feature without releasing everything batched with it.

### Concrete scenario

1. **Feature A** and **Feature B** are both promoted to **QA** at the same time. The QA branch now holds A + B.
2. **Feature A passes UAT** and is approved for production.
3. **Feature B is still pending approval.**
4. The business needs **Feature A in PROD now**.

There is no way to promote A alone. Promoting QA → PROD ships the entire branch — so **Feature B (unapproved) goes to PROD too**.

The team is then forced into one of three bad options:

- **Block A** — wait until B also passes UAT. The approved feature is held hostage by an unrelated, unfinished one.
- **Ship both** — push unapproved code to PROD anyway, defeating the point of UAT.
- **Cherry-pick by hand** — revert B out of the release branch manually. Error-prone, slow, and risks dropping or duplicating commits.

### Root cause

The pipeline conflates two things:

- **Deployment** — getting code onto an environment.
- **Release** — making a feature active for users.

The only release boundary is the branch, and branches batch many features. So features cannot be approved, promoted, or rolled back on their own. Approval happens per feature (UAT), but promotion happens per branch. That mismatch is the problem.

### Impact

- Approved, time-sensitive features are delayed by unrelated work.
- Pressure builds to ship unapproved code, eroding the value of UAT.
- Hotfixes are risky — any fix promoted to PROD drags along whatever else is on the branch.
- Manual cherry-picking adds human error to the release path.

---

## Potential solutions

There is no single fix; the realistic answer is a combination. Listed roughly in order of recommended adoption.

### 1. Feature flags — decouple *deploy* from *release*

Ship A and B to every environment together, but wrap each feature in a flag. B stays dark (flag off) in PROD until it passes UAT; A is turned on.

**This is not a silver bullet.** The unapproved code (B) is still physically shipped to the PROD artifact — the flag only gates whether it *runs*. So the safety is entirely as strong as the discipline around the flag, not a property of the pipeline. The flag is a layer of control, not a wall.

It fails to protect you when the flag is used wrong, for example:
- The off-state isn't truly inert — B's code still touches shared state, runs migrations, or alters startup/DI, so "flag off" ≠ "feature absent."
- The flag is wired incompletely (covers the API but not a background job, or the frontend but not the backend), leaving a half-active feature.
- The flag is flipped on by mistake, or its default in PROD is wrong.
- Only the on-path is tested; the off-path (the state that actually ships) is assumed to work.
- Flags accumulate and are never cleaned up, so the codebase fills with stale, interacting toggles.

In short, feature flags move the risk from "wrong code is promoted" to "flag is misconfigured." That's a better, more controllable failure mode — but only if the off-state is genuinely inert and the flag is tested, owned, and removed once the feature is permanent.

- **Pros:** Directly addresses the coupling. Deploy is always "all of main"; release becomes a per-feature toggle. Enables gradual rollout and instant rollback (flip the flag, no redeploy). Keeps the branch model simple.
- **Cons:** Unapproved code still ships to PROD — protection depends entirely on flag discipline (see above). Needs a flag system (a provider or a lightweight in-house config). Flagged paths must be tested in both states, and flags must be cleaned up.

### 2. Shift UAT left — onto the PR environment (not viable here)

In theory, the platform already builds ephemeral PR environments (`pr-deploy`, the `deploy` label, `<pr#>.pr.erp.dev.jtl-cloud.com`), so UAT could be done per PR before merge, keeping only approved features on `main`.

In practice this does **not** work for us. PR environments run on **Dev infra, which changes often**. UAT exists precisely to validate a feature against stable, production-like conditions — signing off against a shifting Dev environment defeats that purpose. It would be UAT in name only. So PR-environment UAT is more of a gimmick than a real substitute and is **not** a path we should rely on.

### 3. Trunk-based development (the end state of (1))

This is not a separate mechanism — it's where feature flags lead. Everyone works off one main branch and merges small changes often, instead of maintaining the `release` → `release-qa` → `release-prod` chain. Every merge deploys, but unfinished features stay off behind flags. Moving code between environments is automated rather than a manual branch merge, and what's *live* is decided by flags, not by which branch the code sits on. The separate release branches go away.

- **Pros:** Removes branch sprawl and the batching problem at the root. Smaller, safer, more frequent releases.
- **Cons:** Biggest process shift. Depends on solid test coverage and flag discipline.

### 4. Dedicated Jira workflow — make the release state explicit

A process/visibility layer rather than a technical fix. Today nothing tracks *which* feature is approved versus pending at each gate, so the batch is treated as one lump. A dedicated workflow adds explicit per-ticket gates and two "parking-lot" statuses — one for code waiting to go to QA, one for approved work waiting for the prod release window:

| Desired Status | Current Availability | Description | Environment | Requested Change / Action |
|---|---|---|---|---|
| Backlog | Available | Ticket parking lot for unprioritized/unrefined work. | n/a | Rename "Inbox" to "Backlog"; apply to CP, CERP, PRO. |
| To Do | Available | Work is defined and prioritized for the current cycle. | n/a | No action required. |
| In Progress | Available | Active development is happening. | Local / Dev | No action required. |
| In Review | Partially available | Peer review of the code (Pull Request). | GitHub | Rename "Code Review" to "In Review"; apply to CP, CERP, PRO. |
| Testing in Dev | Missing | Internal gate: developer verifies the feature works in Dev. | Dev Env | Create status; apply to CP, CERP, PRO. |
| Ready for QA | Partially available | Code is in Dev, waiting to be deployed to QA. | Dev Env | Rename "QA" to "Ready for QA"; apply to CP, CERP, PRO. |
| Testing in QA | Missing | QA gate / UAT: final validation in QA before Production. | QA Env | Create status; apply to CP, CERP, PRO. |
| Ready for Prod | Partially available | Approved work awaiting the final production release window. | QA Env | Rename "Awaiting Deployment" to "Ready for Prod"; apply to CP, CERP, PRO. |

- **Pros:** Cheap, no code changes. Makes the approved-vs-pending state of every feature visible, so the team stops reasoning about the release as one undifferentiated lump. Gives a shared vocabulary for the gates.
- **Cons:** Does **not** fix the underlying coupling. By itself it just **shifts the congestion from the Prod gate to the UAT gate** — features still pile up, only earlier. It only pays off when paired with a mechanism that actually lets features move independently (i.e. feature flags). Think of it as the governance layer *on top of* (1), not an alternative to it.

### 5. EPIC branches + per-epic squash + cherry-pick to QA

A branch-based answer that gets *real* isolation instead of runtime gating. Each EPIC gets a branch (`epic/pro-1234`); small `feature/*` branches merge into it and the epic branch can be PR-deployed for integrated testing. When the epic is done it is **squash-merged into `main` as one clean commit representing the whole epic**. Because each epic is now a *single* commit, you can **cherry-pick that one commit onto the QA path** (`release-qa`) independently — promoting one approved epic at a time without dragging unapproved ones along.

```
feature/abc ┐
            ├─► epic/pro-1234 ──(PR deploy → dev)──┐
feature/xyz ┘                                       │
                          squash-merge (one clean commit per epic)
                                                    ▼
                          main (dev) ──merge──► release
                                                    │
                          cherry-pick the approved epic's squash commit
                                                    ▼
                          release-qa (QA) ──merge──► release-beta ──merge──► release-prod
```

The squash is the trick that makes cherry-pick viable: you move one atomic, self-contained commit, not a tangle of interleaved small commits. Unlike feature flags, the unapproved epic's code **never enters the QA or PROD artifact** — so there is no "code still ships to prod" risk and no off-state to get wrong.

- **Pros:** True code isolation — unapproved work is physically absent from QA/PROD, not just flagged off. No runtime flag system or flag discipline needed. Granularity finally matches approval: one epic = one promotable unit. Works within today's branch model. Epic branches give a natural integration-test surface (PR deploy).
- **Cons:** Cherry-pick still breaks if epics aren't independent — promoting epic B that depends on epic A produces a broken build, so dependency order must be managed by hand. Long-lived epic branches reintroduce the merge-hell / integration-drift that trunk-based development tries to avoid. The QA branch becomes a hand-curated set of cherry-picks that can drift from `main` over time. One squash commit per epic loses granular history and makes partial revert *within* an epic harder. Relies on discipline that epics are genuinely self-contained.

> **Flags vs. cherry-pick — the core trade-off:** feature flags (1) ship all code and gate *execution* (risk: misconfigured flag, but easy integration). EPIC cherry-pick (5) ships only approved code and gates *inclusion* (risk: dependency/merge drift, but no runtime exposure). Flags favour integration simplicity; cherry-pick favours hard isolation.

### 6. Curated cherry-pick release branch (stopgap)

Keep the branch model but cut `release-prod` from specific approved commits instead of promoting the whole QA branch — *without* the epic-squash discipline of (5).

- **Pros:** No new tooling; works within today's structure.
- **Cons:** Manual and error-prone — exactly the untangling described above, because you're picking many small interleaved commits. (5) is the disciplined version of this; raw cherry-picking is a temporary bridge only.

---

## Recommendation

Lead with **feature flags (1)** — with eyes open. They are the best available mechanism to decouple deploy from release: A and B ship to QA together, UAT happens **on the stable QA environment** per feature via its flag, and only the approved feature is turned on in PROD while the unapproved one stays dark. But they are not a wall — the unapproved code still ships to PROD, and the protection holds only if the off-state is genuinely inert and the flag is tested, owned, and cleaned up. Flags trade "wrong code promoted" for "flag misconfigured"; adopt them only alongside the discipline that makes that trade worthwhile.

Pair this with a **dedicated Jira workflow (4)** as the governance layer, so the approved-vs-pending state of every feature is explicit. On its own the Jira workflow only shifts congestion from the Prod gate to the UAT gate, but on top of feature flags it gives the visibility to manage independent releases cleanly.

The main alternative to consider is **EPIC branches + per-epic cherry-pick (5)**. It trades flags' runtime risk for hard code-level isolation — unapproved work never reaches the PROD artifact — at the cost of dependency management and long-lived-branch drift. If the team is uneasy about flag discipline (the "code still ships to prod" concern), (5) is the stronger fit; if frequent independent releases and gradual rollout matter more, (1) is. The two can also coexist: cherry-pick for hard epic isolation, flags for finer rollout control within an epic.

PR-environment UAT (2) is **not** a viable substitute, since PR environments run on frequently-changing Dev infra. Raw curated cherry-pick (6) is just (5) without the epic-squash discipline — a manual stopgap at best. Over time, flags are the natural path toward trunk-based development (3).
