# Component Library Planning

This folder tracks the bi-weekly planning cycles for the Component Library team.

## Files

- `ticket-list.md` — the source of truth for each cycle's spillover and committed tickets
- `README.md` — this file

---

## How a cycle works

Each cycle has three sections in `ticket-list.md`:

| Section | Who fills it | When |
|---------|-------------|------|
| **Spillover** | AI (`/sync-complib-planning`) | Start of cycle |
| **Committed Tickets** (key list) | Planner manually | Before running the command |
| **Committed Tickets** (AI table) | AI (`/sync-complib-planning`) | Start of cycle |
| **Releases** | Planner manually | Before running the command |

---

## Setup (one-time)

Add your Jira API token to your shell profile:

```bash
echo 'export JIRA_TOKEN=your_api_token_here' >> ~/.zshrc && source ~/.zshrc
```

Get your token at: https://id.atlassian.com/manage-profile/security/api-tokens

---

## Starting a new cycle

### 1. Add a new cycle block to `ticket-list.md`

Copy the template from Cycle 0 and fill in the cycle number, then add the committed ticket keys and release tickets manually:

```markdown
# Cycle N

## Spillover

**This is filled by AI**

## Committed Tickets

**This is filled manually**

Key:
* CP-XXXX
* CP-YYYY

**This is filled by AI**

Total Story Point:

## Releases

**This is filled manually**
- Internal React: CP-XXXX
- UI-React: CP-YYYY
```

### 2. Run the command

```
/sync-complib-planning
```

This will automatically:
- Fetch all active spillover tickets from Jira and populate the Spillover table (Awaiting Deployment first)
- Fetch story points for all tickets via the Jira REST API
- Fill in the Committed Tickets AI table with categories, story points, and a total row
- Update the Confluence page [Component Library Planning Cycle Commitments](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1269235716/Component+Library+Planning+Cycle+Commitments)
- Link each qualifying ticket (non-Design/Docs/Concept/Operation/Process) to its release ticket in Jira based on its component (Internal or Public Component Library)

### 3. The command will stop if…

The `**This is filled manually**` key list under Committed Tickets is missing or empty. Fill it in first, then re-run.

---

## Category exclusion rules for release linking

The following categories are **not** linked to a release ticket:

| Excluded | Reason |
|----------|--------|
| Design | No code shipped |
| Docs | No code shipped |
| Concept | No code shipped |
| Operation | Infrastructure / process work |
| Process | Infrastructure / process work |

All other categories (Implement, Fix, Enhancement, GitHub, etc.) are linked to their corresponding release ticket based on component.
