# AI-First Design System — Concept

> Source: [Confluence — AI-First Design System Concept](https://jtl-software.atlassian.net/wiki/spaces/AICP/pages/1212022823/AI-First+Design+System+Concept)
> Space: AICP (AI and Cloud Platform) · Author: Tobias Lengsholz · Last modified: Mar 20, 2026

This document proposes a new architecture for our component library: primitives from the shadcn registry, a design system documentation platform, and an MCP server that makes it all machine-readable for AI coding tools.

---

## Problem

Building UI components today requires a multi-step pipeline — tickets, requirement engineering, specification, implementation, review — to produce components that are over-engineered for the actual problem. Components get made generic enough to justify their existence in a shared library, even when the use case is specific.

### Process overhead

Every new component or change goes through a heavyweight pipeline (ticket → spec → build → review → release) before a developer can use it. Frontend teams discover issues late, file bugs, wait for fixes that sometimes land as breaking changes — a cycle that could be shorter.

### Over-engineering by design

Components accumulate complexity (form validation, application logic, configuration options) that most consumers don't need. Our DatePicker was rebuilt to look like shadcn but still lacks a11y features that shadcn ships by default. Command requires extra state to hold a value. Effort goes into reimplementing what the ecosystem already solves.

### Composition lock-in

Developers consume opaque, pre-composed components instead of assembling what they actually need. Repeated requests for extensibility (like injectable classNames) have been slow to land because the library's architecture doesn't prioritize consumer flexibility.

### Design-first without constraints

Designs are created without regard for what shadcn/ui supports natively, leading to expensive workarounds. Components that exist in design but aren't finished in code only get discovered by frontend teams at integration time.

### Visibility gap

When components ship, teams find out through version bumps and ticket statuses rather than clear communication of what changed, why, and how to use it. Storybook documentation often lacks code examples in sub-pages.

### No AI integration

The component library isn't machine-readable. Developers already use AI with the library (pointing it at the npm package or Storybook), but there's no structured access — it works by luck, not by design.

---

## Current State

Two repositories form the component library today:

* [jtl-platform-ui-react](https://github.com/jtl-software/jtl-platform-ui-react) (public) — 59 components: 46 primitives + 13 composed
* [jtl-platform-internal-react](https://github.com/jtl-software/jtl-platform-internal-react) (internal) — 18 composed + re-exports from public + business modules

Both use React 19, Tailwind CSS 4, and the shadcn/ui pattern (Radix UI + CVA + tailwind-merge). Storybook 9 for documentation.

### Component Inventory

These mirror shadcn/ui with minimal customization. Sourcing them from the registry means they stay available, gain community improvements (a11y, value handling, className support), and we stop maintaining code that adds no value over the upstream.

`Accordion, Alert, AlertDialog, Avatar, Badge, Button, Calendar, Card, Checkbox, Collapsible, Command, ContextMenu, Dialog, Dropdown, Input, InputOTP, Label, Popover, Progress, Radio, ScrollArea, Select, Separator, Sheet, Skeleton, Switch, Textarea, Toggle, ToggleGroup, Tooltip`

Components that don't exist in the shadcn registry:

`AnnotatedSection, Box, Breadcrumb, ButtonGroup, ErrorMessage, FormGroup, Grid, Icon, JTLLogo, Layout, LayoutSection, Link, Stack, StyledIcon, Tag, Text`

Higher-complexity components with significant dependencies. Remain available as-is. Over time, evaluate which are better served as documented composition patterns.

| Component | Dependencies | Notes |
| --- | --- | --- |
| **DataTable** | TanStack Table + Virtual | Most complex. 15+ sub-components. |
| **Form** | React Hook Form + Zod | Validation framework coupling |
| **ComboBox** | cmdk + Radix Popover | Searchable multi-select |
| **CodeEditor** | Monaco Editor | Specialized — few consumers |
| **FileUpload / HtmlEditor** | react-dropzone / TipTap | File handling / Rich text |
| **DatePicker / DateRangePicker** | react-day-picker + date-fns | Available in shadcn registry |
| **Chart / Stepper / Pagination** | Recharts / — / — | Chart: 5 sub-types. Pagination: in shadcn registry. |
| **JTLDropdown** | Radix Dropdown | JTL-branded variant |

`AppAvatar, AppCard, AppShell, CenteredLayout, DragNDropList, ErrorPage, GlobalLoadingScreen, InnerSidebar, Navbar, PageHeader, PageTab, PhotoUpload, Sidebar, StyledCard, Toast, Tree, TreeSidebar, Workspace`

| Area | Today | Opportunity |
| --- | --- | --- |
| **Sidebar** | 4 implementations across 2 repos | One primitive + composition patterns |
| **Card** | Card + StyledCard + AppCard | Clarify or merge variants |
| **Toast** | Only in internal repo | Promote to shared primitive |
| **Tab** | Tab + PageTab | Clarify relationship |

---

## Proposed Architecture

Three components, each with a distinct responsibility:

### 1. Storybook

Component Dev Platform

* Custom JTL components
* Custom hooks
* Composed components
* Visual regression (Chromatic)

Primitives from shadcn registry, themed via tokens.

### 2. Documentation

Design System Docs

* Design tokens (from Figma)
* Blessed patterns
* Composition rules
* UX principles + assets

The "cookbook" — recipes with intent and reasoning.

### 3. MCP Server

AI Gateway

* Component catalog
* Resolved design tokens
* Composition patterns
* Guardrails

Custom build or Supernova's built-in. Feeds Claude Code, Copilot, Cursor.

### Documentation Platform Candidates

| Platform | Cost | Figma tokens | MCP | Notes |
| --- | --- | --- | --- | --- |
| **Docusaurus** | Free | Tokens Studio + Style Dictionary | Custom build | Already running. Best prose. DIY pipeline. |
| **Storybook alone** | Free | design-token addon | Custom build | Already in use. Prose is second-class. |
| **zeroheight** | ~$16/user/mo | Native Figma sync | Built-in | Purpose-built. Lowest effort. Per-user pricing. |
| **Supernova.io** | ~$25/user/mo | Native Figma, TW v4 exporter | Built-in | Full hub. MCP could replace custom build. |

**Token pipeline (for free options):** [Tokens Studio](https://tokens.studio/) → [Style Dictionary](https://styledictionary.com/) → CSS / Tailwind config.

---

### Alternative: Google Stitch as design tool

[Google Stitch](https://stitch.withgoogle.com) (Gemini 2.5 Pro, free, 350 gens/month) could replace Figma in the design pipeline and eliminate the token handoff problem entirely.

* **Design-first without constraints disappears** — Stitch outputs Tailwind, so designers work within what's implementable.
* **Token pipeline collapses** — Stitch auto-generates [DESIGN.md](https://github.com/google-labs-code/stitch-skills/tree/main/skills/design-md), an agent-friendly markdown format. Can be hand-edited, git-versioned, imported across projects.
* **Prototyping is instant** — working interactive prototypes, not static mockups.
* **Design/code gap closes** — no handoff step where things get lost.

**Doesn't replace:** Storybook (custom components, hooks, testing), design system docs (DESIGN.md only covers tokens/styling — no composition patterns, recipes, assets), or MCP server (Stitch MCP exposes project data, not DESIGN.md).

**Risks:** Google Labs product (could be shut down). V2 just launched March 2026. Unclear how well it handles complex enterprise UI. No native Figma import. Worth a serious pilot — if it works for our complexity level, it solves the design constraint and token pipeline problems in one move.

---

## What Changes

| Today | Where we're heading |
| --- | --- |
| Full pipeline for every component | Primitives from shadcn registry; teams compose what they need |
| ~30 maintained primitives that mirror shadcn | Community maintains those; we focus on what's uniquely ours |
| Our primitives lag behind shadcn (a11y, value state) | Registry primitives ship with full feature parity |
| Designs exceed what the framework supports | Design and engineering align on shadcn's capabilities |
| Frontend discovers design/code gaps late | Design system docs are the shared source of truth |
| New UI needs wait for the pipeline | Teams self-serve with primitives + patterns + AI |
| Changes via ticket statuses and version bumps | Automated changelogs via changesets |
| Component library not machine-readable | MCP gives AI tools structured access — by design, not by luck |

---

## Required Qualifications

### Design System Lead

Owns the design system as a product. Bridges design and engineering.

* Deep shadcn/ui, Radix, Tailwind CSS knowledge
* Design tokens, composition patterns, style guidelines
* Can say "no" to unjustified design deviations
* Figma token workflows and design-to-code pipelines

### Senior Frontend Engineer

Builds and maintains the technical infrastructure.

* Strong React + TypeScript + Tailwind
* Storybook, Chromatic, CI/CD for libraries
* shadcn registry model and consumption pipelines
* Can build an MCP server (Node.js, MCP protocol)

### Documentation Writer

Writes the composition patterns and guidelines that make the MCP server useful.

* Clear, example-driven technical documentation
* API reference vs. usage guidance
* Translates design decisions into documented patterns

### Design Team Alignment

Not a hire, but a prerequisite: the design team needs to commit to working within shadcn's constraints, with deviations treated as exceptions that require justification and an explicit effort/reward assessment.

---

## Implementation Plan

### Phase 1: Foundation

- [ ] Set up shadcn registry consumption — replace ~30 primitives, validate no breaking changes
- [ ] Roll out injectable classNames across remaining custom components
- [ ] Roll out changesets to component library repos
- [ ] Choose design system documentation platform
- [ ] Consolidate sidebar, card, toast, tab overlap
- [ ] Stitch pilot — test with 2-3 real UI tasks to evaluate complexity handling

### Phase 2: Documentation

- [ ] Import design tokens from Figma (or Stitch DESIGN.md) into chosen platform
- [ ] Document first 10-15 composition patterns (prioritized by team demand)
- [ ] Establish design constraint agreement — shadcn capabilities vs. custom work
- [ ] Migrate useful Storybook content into design system docs

### Phase 3: AI Integration

- [ ] Set up MCP server (or configure platform's built-in MCP)
- [ ] Connect component catalog + tokens + patterns
- [ ] Pilot with 2-3 frontend teams using Claude Code / Copilot
- [ ] Iterate on pattern quality based on AI output quality

### Phase 4: Steady State

* Composition patterns grow organically as teams encounter new needs
* Composed components evaluated for decomposition into patterns
* Visual regression testing via Chromatic on custom components
* Design system docs become the default entry point for building UI

---

## Already In Progress

* **Injectable classNames** — agreed and being implemented soon
* **Changesets for changelogs** — piloted in [jtl-platform-configs](https://github.com/jtl-software/jtl-platform-configs), generating human-readable per-release changelogs from PRs

---

## Open Questions

- [ ] Docusaurus (free) vs zeroheight ($16/user) vs Supernova ($25/user, replaces custom MCP)?
- [ ] Who fills the design system lead role?
- [ ] Which Figma files are the token source of truth — and does Stitch change this question?
- [ ] Which composed components should be offered as patterns alongside the composed version?
- [ ] Timeline for shadcn registry switch?
- [ ] How to involve the design team in defining shadcn constraints?
- [ ] Can Stitch handle our enterprise UI complexity, or is it better suited for marketing/landing pages?

---

## To Be Explored

Ideas and tools that could complement this concept but need further evaluation.

| Tool | What it does | Why it's interesting |
| --- | --- | --- |
| [UI UX Pro Max](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill) | Open-source AI skill with 161 design rules, 67 UI styles, palettes, typography | Generic design intelligence for prototyping, landing pages, hackathons |
| [v0.dev](https://v0.dev) | Vercel's AI UI generator — shadcn/ui + Tailwind from prompts | Rapid prototyping. Custom token support unclear. |
| **DESIGN.md format** | Stitch's natural language token format (e.g. "Ocean-deep Cerulean (#0077B6) — Primary action color") | Good model for structuring MCP token content. See Stitch section above. |
| **shadcn custom registry** | Publish JTL components as private registry — npx shadcn add for all | Unifies consumption model across all components |
| **Tokens Studio Pro → GitHub sync** | Figma tokens → GitHub PR → Style Dictionary → Tailwind config | Fully automated Figma → code pipeline |
| **Figma Dev Mode** | Developer handoff view — code snippets, measurements, tokens | Makes design/code gaps visible earlier |
| **AI-assisted pattern generation** | Use MCP context to have AI draft composition patterns from existing usage | Bootstrap pattern library faster than manual docs |
