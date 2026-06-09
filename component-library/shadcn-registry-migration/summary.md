# AI-First Design System — Summary

> Source: [Confluence — AI-First Design System Concept](https://jtl-software.atlassian.net/wiki/spaces/AICP/pages/1212022823/AI-First+Design+System+Concept) · Space AICP · Author Tobias Lengsholz · Last modified Mar 20, 2026
> Full scan: [scanned-content.md](./scanned-content.md)

## TL;DR

A proposal to re-architect JTL's component library around three pillars: **shadcn registry primitives** (instead of maintaining ~30 in-house clones), a **design system documentation platform** (the "cookbook" of patterns and tokens), and an **MCP server** that makes the whole system machine-readable for AI coding tools (Claude Code, Copilot, Cursor). The goal is to drop the heavyweight ticket→spec→build→review pipeline in favor of teams self-serving with primitives + patterns + AI.

## The problem it solves

- **Process overhead** — every component change runs through a slow pipeline; issues found late, fixes sometimes ship as breaking changes.
- **Over-engineering** — components carry complexity most consumers don't need, while in-house reimplementations (e.g. DatePicker) lag shadcn on a11y and features.
- **Composition lock-in** — consumers get opaque pre-composed components; flexibility requests (injectable classNames) land slowly.
- **Design without constraints** — designs ignore what shadcn supports natively, creating expensive workarounds discovered at integration time.
- **Visibility gap** — changes communicated via version bumps/ticket statuses, not clear changelogs; Storybook docs lack code examples.
- **No AI integration** — library isn't machine-readable; AI usage works "by luck, not by design."

## Current state

- Two repos: **jtl-platform-ui-react** (public, 59 components) and **jtl-platform-internal-react** (internal composed + business modules).
- Stack: React 19, Tailwind CSS 4, shadcn/ui pattern (Radix + CVA + tailwind-merge), Storybook 9.
- ~30 primitives mirror shadcn (candidates to source from registry); a set of custom non-registry components; and high-complexity components (DataTable, Form, ComboBox, CodeEditor, etc.) that stay as-is.
- Known duplication to consolidate: Sidebar (4 impls), Card/StyledCard/AppCard, Toast (internal only), Tab/PageTab.

## Proposed architecture (3 parts)

1. **Storybook** — component dev platform for custom JTL components, hooks, composed components, visual regression (Chromatic). Primitives come from shadcn registry, themed via tokens.
2. **Documentation** — design system docs: tokens (from Figma), blessed patterns, composition rules, UX principles. The "cookbook."
3. **MCP Server** — AI gateway exposing component catalog, resolved tokens, composition patterns, and guardrails to AI tools.

**Platform candidates:** Docusaurus (free, DIY pipeline), Storybook alone (free, weak prose), zeroheight (~$16/user/mo, built-in MCP), Supernova.io (~$25/user/mo, full hub, built-in MCP). Free options use a Tokens Studio → Style Dictionary → Tailwind pipeline.

**Wildcard — Google Stitch:** could replace Figma, output Tailwind directly, and auto-generate an agent-friendly `DESIGN.md`, collapsing the token handoff. Risks: Google Labs product, just launched V2 (Mar 2026), unproven on complex enterprise UI, no Figma import. Flagged for a serious pilot.

## Rollout

- **Phase 1 Foundation** — adopt shadcn registry, injectable classNames, changesets, pick docs platform, consolidate duplicated components, Stitch pilot.
- **Phase 2 Documentation** — import tokens, document 10–15 composition patterns, agree design constraints, migrate Storybook content.
- **Phase 3 AI Integration** — stand up MCP server, connect catalog/tokens/patterns, pilot with 2–3 teams.
- **Phase 4 Steady State** — patterns grow organically; docs become the default entry point for building UI.

## Roles needed

Design System Lead (owns it as a product), Senior Frontend Engineer (infra + MCP server), Documentation Writer (composition patterns), plus **Design Team Alignment** — a commitment to work within shadcn constraints, deviations as justified exceptions.

## Already in progress

Injectable classNames (agreed, implementing soon); changesets for human-readable changelogs (piloted in jtl-platform-configs).

## Key open questions

Which docs platform (Docusaurus vs zeroheight vs Supernova)? Who is the design system lead? Token source of truth in Figma — and does Stitch change that? Timeline for the shadcn registry switch? Can Stitch handle enterprise UI complexity?
