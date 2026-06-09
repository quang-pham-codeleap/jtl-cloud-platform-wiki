# Component Library — Problem Review & Diagnosis

> A critical review of the [AI-First Design System Concept](./scanned-content.md).
> Purpose: separate the *symptoms* the proposal reacts to from their *actual source*, so the fix is proportional to the problem rather than to the proposed architecture.
>
> Scope note: the original concept bundles three initiatives (shadcn registry pivot, documentation platform, MCP server) and credits the bundle with solving six problems. This review collapses those six down to the **two that actually hurt**, locates the source of each, and only then discusses mechanism (npm vs. copy-paste, customization, Atomic tiers).

> **Bottom line.** Don't decide yet — run a short audit first (Part V). shadcn might earns a place, but for AI legibility and team-owned higher-level components, not the maintenance savings the proposal sells. And if the real pain is slow releases or hard assembly, changing how components ship won't fix it.

---

## Part I — Framing

### The two problems that actually matter

Strip the concept down and two things genuinely hurt:

1. **The library blocks Application Developers** — teams wait on the central team, work around components, or reimplement locally.
2. **The library has no AI-first mentality** — no MCP, no machine-readable catalog, no AI effort. AI usage "works by luck, not by design."

Both are **symptoms**, not causes. Each can come from several different sources, and **each source has a different fix** — so doing the most visible work (re-sourcing 30 primitives) can leave the real bottleneck untouched. The concept jumps from symptom to a four-phase platform; this review inserts the missing step: **diagnose first.**

### The shadcn registry is an answer in search of a problem

The proposal is built backwards — **mechanism → problem** instead of **problem → diagnosis → mechanism.** It opens with the registry as the headline move, then recruits six problems to justify it. The tells:

- **It's credited for problems it doesn't solve.** Only *over-engineering* (partly *composition lock-in*) actually touches the registry; *process overhead*, *visibility*, and *AI integration* belong to the other two pillars (changesets, docs, MCP).
- **It targets the lowest-friction tier.** Developers don't tend to stall on `Separator` — they stall on organisms (the Sidebar-×4 pain, §2.3). The pivot re-sources the ~30 *stable* primitives, i.e. the components least implicated in the complaints.
- **Its headline benefit rests on an unmeasured 1:1 assumption that collapses either way.** If our primitives *are* 1:1 with shadcn → it's a one-afternoon re-pull, not a program; if they *differ* → `shadcn add` is a snapshot, so we re-fork and upstream fixes stop flowing — the maintenance we claimed to escape. Never both high-impact *and* as-advertised.
- **Copy-paste fixes a problem we haven't confirmed.** Its real mechanism is *source ownership* (buckets A/B), but the likely-dominant complaints are latency (C), knowledge (D), or assembly (E) — which copy-paste doesn't touch, and E it worsens.
- **It was chosen for being cheap and fashionable, not derived.** That's what makes it risky as the headline: approving "the pivot" reads as approving the program it rides with (paid platform, custom MCP, replacing Figma with a just-launched Google Labs product).

### It misreads what shadcn is *for*

Beneath all the tells above is one root mistake: the proposal wants shadcn for the exact benefit shadcn is designed to give up. shadcn says so itself:

> **"This is not a component library. It is how you build your component library."**

shadcn's model is a deliberate **trade — ownership in exchange for centralized maintenance.** You run the CLI once, the code lands in your repo, and it's yours to change; but in taking that control you give up what a versioned package gives you, a single upstream that pushes fixes down to every consumer. This is the same npm-vs-copy-source tradeoff spelled out in §3.6 — *own the source and copies drift.* That trade isn't a flaw in shadcn; it *is* shadcn.

The proposal tries to bank both halves at once. It sells the pivot as **maintenance relief** — adopt shadcn primitives and inherit the ecosystem's a11y and bug fixes for free — which is precisely the half the trade gives away. You *can* keep clean upstream pulls, but only by never diverging, i.e. by treating shadcn as the closed dependency we already have and gaining nothing from the switch. The moment you use the ownership shadcn exists to provide, the free fixes stop. So this isn't a flaw separate from the 1:1 assumption above — it's *why* that assumption is decisive: shadcn forces the very choice the proposal tries to dodge. (§3.4's "stable workflow" is an attempt to engineer a narrow path through this with pristine vendoring + wrappers — real discipline we impose, not a benefit shadcn ships.)

Once maintenance-relief collapses, the justification leans entirely on **"AI-first" — which can't hold it up.** Being AI-first is a *content* problem (Part IV); none of that content needs us to re-source primitives. shadcn conventions do buy AI legibility nearly for free — but only on atoms that are *already* 1:1, where adopting costs nothing anyway; it is no reason to re-fork the ones that differ. Invoking "AI-first" to carry the migration is answer-in-search-of-a-problem, one level up.

None of this is anti-shadcn — it's anti-*misuse*. shadcn's intent fits us at the **opposite end of the hierarchy from where the proposal points**: owning and freely diverging *organism/template* source (§3.6) is shadcn exactly as designed — copy a block, make it yours, never look back. That, plus free legibility on already-1:1 atoms, is its honest value here. "Re-pull pristine primitives for free upstream fixes" is the one thing shadcn was built *not* to be.

---

## Part II — Problem 1: the library blocks Application Developers

This is the most easily *mis-diagnosed* problem: "the components block us" can mean five very different things, which look identical from the outside but need opposite fixes.

### 2.1 The possible sources

| # | Bucket | What the developer says | Where the problem actually lives | Fix direction |
|---|---|---|---|---|
| A | **Capability** | "The component can't do what I need" | The artifact is too **rigid** | composition / tokens / slots |
| B | **Ownership** | "I knew the fix but couldn't make it — had to ask the central team" | Central team is a **bottleneck** | self-serve (own the source) |
| C | **Latency** | "The change was trivial but took two weeks to ship" | The **release / CI pipeline** is slow | *not a component problem* — fix release process |
| D | **Knowledge** | "It already did this — I just didn't know, so I rebuilt it" | **Documentation / discoverability** gap | examples + docs |
| E | **Assembly** | "I have the bricks, but stitching them into a screen is slow and hard" | A **missing higher tier** — no assembled starting points | move *up* the hierarchy (organisms, templates, patterns, AI) |

The concept assumes the problem is **A + B** and prescribes shadcn accordingly. But if it's **C**, no amount of shadcn helps (it's a process/CD problem); if **D**, the fix is docs; if **E** (§3.5), the cure is assembled higher tiers, and most "obvious" fixes make E *worse*. **We don't yet have the data to know which dominates — that's the first thing to try and find out.**

### 2.2 The diagnostic

Pull the last ~20–30 times a developer was blocked — via Github Issues, "add a prop/variant" requests, Teams Message, and most tellingly **local re-implementations**:

- Mostly **A + B** → composition / self-serve direction is right (§3.3–3.4).
- Mostly **C** → fix release process; the architecture is a red herring.
- Mostly **D** → docs + code examples (doubles as the AI fix).
- Mostly **E** → build the missing organism/template tier + patterns + AI (§3.5).

Without this, any large investment is a bet placed without looking at the table.

### 2.3 The most likely source (hypothesis, to confirm)

The concept's own evidence points to **A/B and E**, in one place:

> **Developers rarely stall on primitives. They stall on organisms** — opaque/rigid, un-adaptable without a ticket, or simply missing so every team rebuilds them.

- "opaque, pre-composed components instead of assembling what they actually need" (A/B)
- injectable-className requests "have been slow to land" (B)
- **Sidebar exists 4×** — nobody rebuilds a Button 4 times; they rebuild the *organism* the library stops short of (E)

So the friction lives one layer above the ~30 stable primitives the pivot targets.

---

## Part III — Solution Space: if we're actually blocking the App Developer

> This whole part applies **only if the diagnosis (Part II) lands on bucket A/B (rigidity / ownership) or E (assembly).** If it lands on C (latency) or D (knowledge), stop here — the fix is the release pipeline or the docs, and none of the tier/delivery questions below are relevant.
>
> We haven't actually run that diagnosis yet — that's the audit in Part V. This part lays out the full solution space *in advance*, so the audit has somewhere to point and we can see what fix each bucket would call for before committing to anything.
>
> Assuming we *are* blocking developers at the component level, the work runs in order: **(1) sort the components (§3.1), (2) per atom, decide whether to source it from shadcn or build our own — this is where the pivot lives or dies (§3.2), (3) decide how flexible each tier should be (§3.3), (4) apply that customization safely (§3.4), (5) handle the assembly case (§3.5), (6) decide how each tier is delivered (§3.6).** Every later decision falls out of the first two.

### 3.1 First, sort the components by Atomic tier

The library's basis is Atomic Design (Atoms → Molecules → Organisms → Templates → Pages). Before any "should we adopt shadcn" question, sort what we actually have — because the answer is different in every row, and the proposal's headline move only touches one of them.

| Tier | What's here (from the inventory) | shadcn coverage | JTL's unique value |
|---|---|---|---|
| **Atoms** | Button, Input, Label, Badge, Checkbox, Separator, Avatar — *plus layout/type atoms shadcn has no opinion on* (Box, Stack, Grid, Text) | **High** — most of the ~30 "stable primitives" | Low — mostly theming |
| **Molecules** | Select, Dropdown, Dialog, Popover, Tooltip, Tabs, Accordion + JTL form-fields | **Partial** — compound, stateful Radix parts (`Trigger`/`Content`/`Item`) | Moderate — convenience wiring |
| **Organisms** | **Sidebar (×4), AppShell, PageHeader, Card/StyledCard/AppCard, Toast, PageTab** | **Little to none** | **High — this is the product** |
| **Logic-heavy** | DataTable, Form, ComboBox, CodeEditor | Partial (Command, Form) | High — and churns |
| **Templates / Pages** | app territory | None | Recipes only |

One thing jumps straight out of this table: **shadcn is not simply "our atoms."** Its catalog spans atoms *and* most of the molecule tier — so "swap our atoms for shadcn primitives" both undersells what shadcn actually covers and, more tellingly, leaves the **organism** row — where our real value lives — completely untouched. Keep that in mind; it's the thread running through everything below.

### 3.2 The per-atom gate: source from shadcn, or build our own

Now that the components are sorted, we walk the atoms one at a time — because this is the step that decides whether the headline pivot even applies. The proposal's premise is that *every* atom should come straight from shadcn, so the burden is on us to justify any atom we'd build or keep ourselves instead. In other words: **adopting shadcn is the default; building our own has to earn it.**

The principle that sets the bar:

> **Divergence cost is inversely proportional to the Atomic tier. Diverge freely at the top, reluctantly at the bottom.**

Atoms have the highest blast radius (everything inherits them) and benefit most from shadcn's ecosystem a11y investment, so the bar for "build our own" is **high**. (Organisms/templates are app-specific where shadcn offers little — there, divergence needs no defending; see §3.5.)

**What counts as a valid justification to build our own atom:** (1) it doesn't exist in shadcn — Box, Stack, Grid, Text, Link, Icon, Tag (layout/typography shadcn has no opinion on); (2) a different API contract / mental model (polymorphic `as`, token-only spacing) where wrapping fights the grain; (3) cross-cutting concerns shadcn doesn't carry (i18n/RTL, stricter a11y, analytics hooks).

**The test:** building our own is justified only when the divergence is **structural / behavioral / contractual** and can't be expressed as a token or a thin wrapper. **Visual divergence is a token, not a new atom** — rebuilding an atom just to *look* different, while inheriting its maintenance and falling behind on a11y, is the DatePicker mistake.

**And here is where the pivot lives or dies.** Read the test as a gate on the proposal itself:

- **No foundational divergence** (the atom is identical, or any difference is reconcilable by token/wrapper) → it's a genuine pivot candidate. Source it from shadcn and carry it forward.
- **A justified foundational divergence exists** (passes the test above) → **adopting shadcn as the primitive for that atom simply fails here.** There is no token-and-wrapper path back to the shadcn source; re-sourcing it would mean either abandoning a divergence we decided we needed, or re-applying it on every `shadcn add` — a permanent fork, i.e. the exact maintenance the pivot promised to remove.

> So "adopt shadcn as our primitive layer" is **not** library-wide true or false — it holds **per atom**, and it **breaks at every atom carrying a justified foundational divergence.** Each of those is a conscious *keep-our-own*, not a swap.

Running this as step two pays off immediately: the pivot is only as valid as the share of atoms that *survive* it. If many of our atoms carry justified divergences, the headline "replace our primitives with shadcn" is already mostly dead — and we learn that before spending a word on tokens, wrappers, or delivery. This is exactly what the audit in Part V sets out to measure; everything that follows applies only to the atoms that clear this step.

### 3.3 How flexible each tier should be

The gate told us *which* components we keep ownership of. The next question is how much each of them should let a developer change — and here the intuitive answer is the wrong one.

The naive reaction to "the component blocks me" is "give it more knobs." It's usually wrong, and the reason is a tradeoff most knob-adding ignores:

> **Customizability and ease-of-assembly pull against each other at the low tiers.** Every prop / slot / variant on an atom is one more decision the assembler is forced to make. A maximally customizable atom is maximally *hard to use.*

So the design rule is the inverse of "make everything flexible":

> **Atoms should be dumb; organisms should be smart. Customization rises UP the hierarchy — and the mechanism shifts with it: tokens at the bottom, composition in the middle, own-the-source at the top.**

| Tier | Dumb ↔ Smart | How customizable | Mechanism | Why |
|---|---|---|---|---|
| **Atoms** | **Dumb** — strong defaults, near-zero decisions | **Least** | **Tokens** + a few structural props | Used everywhere; prop-explosion makes them unusable |
| **Molecules** | Mixed | Moderate | **Composition / slots** + tokens — offer *both* a simple opinionated form *and* a composable one (progressive disclosure) | Common case needs zero wiring; exception case drops to the parts |
| **Organisms / Templates** | **Smart** — carry product identity and assembly decisions | **Most** | **Own the source** (copy-paste / blocks) | This is where divergence and decisions legitimately live |
| **Logic-heavy** (DataTable, Form) | **Smart, but closed** — rich behavior, not a customization surface | Low–moderate, via props/config | **Versioned npm**, not source-owned | Behavior is the value; 50 forked copies is the nightmare, not the goal |

**The reassessment this forces.** When a dev says *"the atom can't do what I need,"* the instinct is to add a prop to the atom. Re-read against the table, the right answer is almost always one tier away:

- want it to *look* different → **token**, not a prop;
- want a different *structure* → **slot/composition**, not a prop;
- want a whole assembled thing → *"that's an organism's job — here's the organism,"* not a fatter atom.

In other words, most "make the atom more customizable" requests are mis-filed: they're really token gaps, missing molecules, or a missing organism tier (bucket E). **Adding knobs to atoms would make the library measurably worse** — harder to assemble — while leaving the real complaint unfixed. That reassessment is the whole point of sorting first.

### 3.4 Customizing safely: the "stable workflow"

§3.3 settled *how much* each tier should bend. This section is the mechanics of actually bending it — applying that customization to a shadcn component without losing the ability to pull in upstream updates later.

> **One precondition.** This only applies to the components we've **accepted sourcing from shadcn** — the atoms and molecules that cleared the §3.2 gate. For anything we deliberately kept our own, there's no upstream to re-pull from, so none of this matters; it's just a normally-maintained component. §3.2 decides *whether* shadcn is our primitive; this section only asks *how to live on it safely* once we've said yes.

shadcn gives you no automatic updates by design (Part I) — so "upgrade-safe" here means a *disciplined manual* path, not a free one. It is achievable, but how safe it stays depends entirely on *where* the customization lives:

| Layer | Example | Lives where | Re-pull safe? |
|---|---|---|---|
| **1. Tokens / theme** | brand colors, radius, spacing | `globals.css` + tailwind config (CSS vars) | ✅ Fully decoupled |
| **2. Variants** | a JTL `size="compact"` via CVA | co-located variants file | ✅ Mostly safe |
| **3. Composition wrapper** | `JtlDatePicker` = calendar + popover + defaults | separate file that *imports* the primitive | ✅ Safe — primitive stays pristine |
| **4. Source edit** | change internal JSX, rewire Radix | inside the vendored file | ❌ This is a **fork** |

> **Wrap, don't edit.** Tokens + wrappers + variants are decoupled, so upstream updates flow in. Source edits are a fork, and the stable workflow is gone.

**The workflow:** vendor primitives into a *pristine*, never-edited directory (like a lockfile) → theme via tokens → when you need more, write a wrapper that *imports* the vendor primitive → publish wrappers via a private registry. **Update** = re-run `shadcn add` into vendor/, diff, review, run Chromatic. Because wrappers depend only on the public API, breaking changes surface as **TypeScript errors in the wrapper** — localized and detectable.

**The line:** once a customization can't be expressed as token + wrapper and forces a source edit, that component becomes a maintained fork — accept it consciously and quarantine it. (The DatePicker got stranded exactly this way: a fork nobody re-pulled.)

### 3.5 When the real problem is assembly, not rigidity

Everything so far has been about making individual components more adaptable. But there's a different complaint hiding inside "the library blocks us," and it needs the opposite move. Some developers don't struggle because a component is too rigid — they struggle because they have all the parts and still find building a whole screen slow and fiddly (bucket E).

When that's the pain, the fix is to go **up the hierarchy** and hand them bigger, ready-made pieces. It is emphatically *not* more knobs on the atoms — every extra knob is one more decision to make — and *not* a change in how things are shipped. Three actions, in order of impact:

1. **Ship the missing tier** — deliver organisms/templates as **ready-to-use starting points**: a dashboard layout, a settings page, the *one* canonical Sidebar. Biggest single lever.
2. **Write down the common patterns** — recipes for how to compose things, so devs reuse a decision instead of re-making it each time.
3. **Add convenience molecules** that pre-wire the fiddly bits — e.g. a `FormField` bundling Label + Input + Error + a11y — while still letting them drop down to the parts when needed.

### 3.6 How it ships: when copy-paste actually helps

We've settled how flexible each component should be and how to keep it upgrade-safe. The last question is distribution — how a component reaches an app team. There are really only two answers: an opaque npm import they can't touch, or source code they own and can edit. The shadcn pivot is, at bottom, an argument for the second one.

> **A word of caution first.** This is the *last* decision, not the first. The proposal frames the entire pivot *as* this delivery swap — but changing how a component ships only helps if the diagnosed pain was **ownership** (bucket B). If the pain was latency, missing knowledge, or assembly, switching from npm to copy-paste answers a question nobody asked.

**Step 1 — Pin down what "ship it in the shadcn registry" even means.** The phrase blurs three different moves, and only one is worth discussing:

| Meaning | Verdict |
|---|---|
| **(a) Consume** the public shadcn registry into our repo | the original "pivot" — fine for stable primitives, but doesn't touch the bottleneck |
| **(b) Contribute** our components to the *public* shadcn.com registry | **No** — that open-sources our business UI to the world |
| **(c) Publish our own *private*, shadcn-CLI-compatible registry** (`npx shadcn add <our-url>`) | **The one worth considering** — distribute JTL components through the registry mechanism |

So the live question isn't "shadcn registry: yes or no" — it's whether to distribute a given component as **(c) copy-source** or keep it as a versioned **npm package.**

**Step 2 — Understand the tradeoff between those two models.** They are mirror images; one's strength is the other's weakness:

| | **npm package** (current) | **Registry / copy-source** |
|---|---|---|
| Consumer owns the code? | No — opaque import | **Yes — source in their repo** |
| Adapt without a ticket? | No → **the A/B bottleneck** | **Yes → unblocks A/B** |
| Centralized fix propagation? | **Yes** — bump the version | **No** — copies drift |
| Fragmentation | Low | **High** |
| AI-legibility | Weak | Strong |

The takeaway: copy-source **buys ownership at the cost of drift.** That trade is worth making only where ownership is the actual pain — which is *not* uniform across the library.

**Step 3 — So match the model to the tier; don't switch the whole library at once.**

| Tier | Delivery | Why |
|---|---|---|
| **Stable primitives** | registry (consume shadcn) or private-registry wrapper | low churn; self-serve is safe |
| **Composed / business UI** (the bottleneck) | **private registry, as `blocks`** — copy & own | needs per-app adaptation; this is the real self-serve win |
| **Logic-heavy** (DataTable, Form) | **npm, closed + versioned** | we don't want 50 drifting copies; centralized fixes matter most here |

**Two refinements that keep this from over-reaching:**

- **It's even per-component, not just per-tier.** Only a deep *restructure* needs copy-source ownership; a mere *restyle/extend* is better served by **npm + injectable classNames + slots** — no fork, no drift. That single feature dissolves much of the A/B bottleneck *without touching the registry at all*, which is why it's so high-leverage.
- **Copy-paste is a *flexibility* tool, not an *ease* tool.** For atoms/molecules it adds ownership burden while doing nothing for assembly (bucket E). It earns its place only at **organisms/templates**, where the value is the *pre-assembled starting point* — the copy-paste is incidental.

---

## Part IV — Problem 2: no AI-first mentality

Problem 2 isn't really separate from Problem 1. They're the same root wearing two faces: **we ship closed artifacts and tribal knowledge, so nobody can self-serve.** A human hits that wall and waits or forks; an AI hits the same wall and guesses. That's why what follows is mostly the Problem 1 fixes — composable source, written-down patterns, worked examples — pointed at a second reader.

So the bulk of this review has been Problem 1, where the proposal's reasoning is shakiest. Problem 2 needs far less space: the diagnosis is obvious, and the fix is largely work we've already described.

### 4.1 Assessment

Unlike Problem 1, this one needs no diagnostic — the gap is total and beyond dispute: **no MCP, no machine-readable catalog, no AI effort at all; AI usage "works by luck, not by design."** But "total gap" is not the same as "big project," and the proposal mis-reads which part is actually hard.

The reframe that keeps the fix proportional:

> **AI-first is mostly a *content* problem, not a *transport* one.** An MCP server, an `llms.txt`, a catalog endpoint — these are all just *transports* that expose knowledge an agent can consume. They are worthless if the knowledge doesn't yet exist as structured, example-rich content. **Write the content first; the transport is the last mile.**

What this implies for the assessment:

- **The expensive, headline transport (a custom MCP server) is the *last* thing to build, not the first.** Building it before the content exists produces a server that returns thin, example-poor data — luck, with extra infrastructure.
- **Most of the AI-first work is the *same work* as Problem 1.** Code examples and composition patterns fix the Knowledge (D) and Assembly (E) buckets for humans *and* are exactly what an agent reads. One body of content, two consumers — every artifact is twice-useful.
- **shadcn conventions buy AI legibility nearly for free.** Every frontier model is trained on shadcn's structure and naming, so the atoms that legitimately stay 1:1 (Part III §3.2) are already maximally legible. *This — not maintenance — is the strongest honest argument for the pivot.*
- **The bar for "AI-first" is concrete:** an agent, given only our published content, can pick the right component, compose it the blessed way, and respect our constraints — *without guessing.* Today it guesses.

### 4.2 Immediate actions (cheapest first, transport last)

These are ordered so each rung delivers value alone and feeds the next; you can stop at any rung that closes the gap. The first three are content the agents (and humans) read; the last two are transport.

| # | Action | Cost | Why it's first / what it unlocks |
|---|---|---|---|
| 1 | **Write an `AGENTS.md` / guardrails file** at each repo root — "how to build UI here": which components exist, the composition rules, the do/don'ts, the design constraints. | Hours | The single highest-leverage move. Every coding agent (Claude Code, Copilot, Cursor) reads it today, no infrastructure. Turns "by luck" into "by rule" immediately. |
| 2 | **Add a copy-paste usage example to every component** (Storybook story + a code snippet block). | Days | Closes bucket D for humans *and* gives agents grounded examples instead of hallucinated APIs. The proposal already flags Storybook lacks code examples — this is owed regardless. |
| 3 | **Document the 10–15 blessed composition patterns** as example-rich markdown (the "cookbook" recipes). | Days–weeks | This is the Assembly (E) fix and the AI-assembly fuel at once — an agent that knows the patterns can wire screens up. |
| 4 | **Emit a machine-readable catalog** — a generated manifest of components, props, variants, and resolved tokens (a private `registry.json` gives this *by construction*). | Days | The structured index transports read from. Falls out of the registry work if the pivot proceeds; build standalone if not. |
| 5 | **Stand up the MCP server** — only now, as the last mile, pointing at the content from 1–4. | Weeks | Worth it only once 1–4 exist; otherwise it serves thin data. Pilot with 2–3 teams before generalizing. |

### 4.3 The sequencing rule

**Ship rungs 1–3 first, then measure whether the gap is already closed** — a well-fed coding agent with a guardrails file, examples, and patterns may be "AI-first enough" without any custom infrastructure. Build the catalog (4) and MCP server (5) **only if those cheaper transports demonstrably fall short.** This keeps the AI investment proportional and front-loads the work that also fixes Problem 1.

---

## Part V — Recommended next action: the component inventory matrix

Everything above is reasoning; this is the one thing to actually *do*. The whole review collapses into a single deliverable that turns the two gates we've leaned on — the demand diagnostic (§2.2) and the per-atom pivot gate (§3.2) — into one table, so the migration decision falls out of evidence instead of opinion.

The recommendation, then, is **not** to approve or reject the migration. It's to commission a bounded audit — **roughly one week, one owner** — that produces a single artifact: a **component inventory matrix, one row per component**, from which the decision falls out mechanically.

### 5.1 The artifact

One row per component, carrying it from *what is it* on the left to *what we do with it* on the right. The last two columns (→) are **conclusions**, derived from the ones before them — not opinions filled in up front:

| Column | Values |
|---|---|
| **Component** | Button, Sidebar, … |
| **Atomic tier** | atom / molecule / organism / template |
| **shadcn equivalent** | exists / partial / none |
| **Demand signal** *(fill first — it gates everything)* | blocked-often / occasional / never — **+ bucket A/B/C/D/E** |
| **Divergence from shadcn** | none / visual-only / structural-behavioral |
| **Divergence expressible as** | token / wrapper / **source-edit (fork)** |
| **Divergence justified?** | deliberate / legacy-NIH / unknown |
| **Fitness as a building brick** | good / too-rigid / too-opaque / missing-tier |
| **→ Recommended distribution** | adopt-vanilla / npm+wrapper / registry-block / npm-closed |
| **→ Recommended action** | adopt / wrap / keep-fork / decompose-to-pattern / build-organism |

### 5.2 The order to fill it in

Don't fill the columns left-to-right. Two of them are gates that can end the exercise early, so they go first:

1. **Classify** every component into its Atomic tier — settle the purist/pragmatic line (§3.1) up front.
2. **Pull the demand signal — from evidence, not memory.** *This is §2.2's diagnostic, run for real; it gates everything else.*
   - **Re-triage the GitHub issues** across both repos — re-tag each by **(a) component targeted** and **(b) bucket (A/B/C/D/E)**. ("New Sidebar variant" = organism/E; "injectable className on Button" = atom/A.) The *distribution* of tags is the demand signal.
   - **Scan the application repos for workaround code** — local re-implementations, copied-and-modified components, `// TODO: library doesn't support…`. **Developers vote with their workarounds**; this is the strongest, least-biased signal. Categorize each by the component/tier it routes around.
   - Roll both up into the **Demand signal** column — turning "developers are blocked" into a map of *which* component, tier, and bucket.
3. **Run the per-atom gate** from §3.2 for real — compare each atom/molecule against the *current* shadcn primitive and record the delta as **1:1 / token-only / structural-or-behavioral**. This is the make-or-break number for the pivot: the share that comes back 1:1 (or token-reconcilable) *is* the share the pivot can actually cover. Until this column exists, "remove our atoms, use shadcn primitives" is an assertion, not a plan.
4. **Assess fitness + divergence** for the remaining tiers (token / wrapper / fork).
5. **Fill the two → columns**, closing with a one-paragraph go/no-go per tier.

### 5.3 The decision is per-tier, not one binary yes/no

Once the demand signal is in, the dominant bucket dictates the answer — and for three of the five buckets, the distribution question never even arises:

| Dominant bucket | Verdict |
|---|---|
| **C** (latency) | Fix the release process. **npm-vs-copy-paste is irrelevant — stop here.** |
| **D** (knowledge) | Docs + examples. Distribution unchanged. |
| **E** (assembly) | Build the missing organism/template tier + patterns + AI. Distribution is secondary. |
| **A/B** (rigidity / ownership) | The distribution question is real — but answered *per tier*: atoms/molecules stay npm + injectable classNames + wrappers; **organisms** pilot registry blocks; logic-heavy stays npm-closed. |

### 5.4 Scope and ownership

- **Days, not quarters.** The deliverable is a go/no-go, not a report.
- **The would-be design-system lead is the natural owner** — this matrix *is* the audition for that role.
- **Build it once, use it twice** — the same structured inventory is exactly the machine-readable catalog the AI/MCP work needs (Part IV, rung 4).
- **Everything else is unbundled and deferred** — paid docs platform, Figma → Google Stitch. Neither addresses the two problems that actually hurt.

---

## One-line summary

Both problems share one root — **we ship closed artifacts and undocumented knowledge, so nobody (human or AI) can self-serve.** The cure: composable bricks + assembled higher tiers + written-down knowledge, customization rising up the Atomic hierarchy, copy-paste reserved for where pre-assembly is the point. But **measure before you build** — run the matrix with the demand signal first, because if the complaint is latency or assembly, an npm→copy-paste migration answers the wrong question. Treat the shadcn pivot as a cheap path to AI legibility, not a maintenance silver bullet.
