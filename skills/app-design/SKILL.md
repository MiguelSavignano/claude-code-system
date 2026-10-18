---
name: app-design
description: How screen designs for your team's apps are made with claude.ai artifacts — one design system per product (even within the same company) and one design canvas per repo (an admin/backoffice living in the backend repo uses its product's design system on its own canvas), the canvas organized into pages by domain ("Base design" first), each screen in a "today | proposal" row with a note, and the registry of where each design system and canvas lives and which domains it has. Use before designing or redesigning a screen, when creating an app's design system, when adding a domain, or when someone asks "where is the design for X?". Invoke as /app-design [app] [screen].
argument-hint: "[app] [screen]"
---

# App designs: one design system and one canvas per app

Record here when the registry was last checked against the artifacts (e.g. "Checked YYYY-MM-DD").

## The rule

**One canvas per repo with a UI; one design system per product.** Even when products belong to the
same company, they don't share styles: different products (e.g. a React SPA, a Django app, a Rails
app, a Next.js app) have different tokens. Never design a screen with another product's design
system, and never put screens from two repos on the same canvas.

The exception that proves the rule: an **admin/backoffice that lives in the backend repo** (`api`)
belongs to the same product as the SPA (`web`), so it uses **that product's design system**, but it
gets **its own canvas** because it lives in a different repo from `web`.

**Before creating anything, look the app up in the registry.** If it already has a design system or
canvas, work on that one (by its `url`); creating another leaves two sources that age separately.

## Registry

Fill this in for your organization.

| App (repo) | Design system | Design canvas | Pages (domains) |
|---|---|---|---|
| `<product>` (`web`) | [`<product>` Design System](https://claude.ai/artifact/<id>) | [`<product>` design](https://claude.ai/artifact/<id>) | Base design · Auth / Company / Profile · Orders · Invoices · Reports |
| `<product>` backoffice (`api`, admin panel) | the `<product>` one (row above) | [`<product>` backoffice design](https://claude.ai/artifact/<id>) | Base design · Home · Catalog · Users · Organizations · Access · System |
| `<app>` | [`<app>` Design System](https://claude.ai/artifact/<id>) | [`<app>` design](https://claude.ai/artifact/<id>) | Base design · … |

A new domain or a new app **is added to this table in the same change** (with `/skill-design`).

Artifact types they are created from (claude.ai; list them with the Artifact tool's `scope: "types"`
and record the URLs here):

- Design System: `https://claude.ai/artifact/<design-system-type-id>`
- Design (canvas): `https://claude.ai/artifact/<design-type-id>`

Each is created only once with `type_url` + `title`; afterwards always by its `url`. The type's own
instructions (returned by a `read` of the artifact) define the file format.

## An app's design system

- It comes **from the app's code**, with exact values: colors, typography, spacing, radii,
  components, logos and icons copied (never approximated). Each token cites its source file.
- If the repo already has its own kit (e.g. `src/lib/ui/tokens.ts` + `UI_KIT.md`), **the repo
  rules**: the design system mirrors it, it doesn't replace it. If the repo's kit changes, resync
  the design system.
- The app's canvas has it installed (`designSystems` in `project/canvas.json`).

## An app's canvas

- **Pages = the app's domains**, taken from its routes (not invented). The first one is always
  **"Base design"**: the sheet of tokens and patterns built with the design system.
- On each page, one row per screen: **left "today"** (a screenshot of the current screen, or a
  faithful recreation from the code) and **right "proposal"**, built only from "Base design"
  patterns. Above it, a `title1` note with the name; to the right, a note with what changes and why
  (plus route + source file).
- Desktop screens at 1440 wide as a fluid page; if field users use it on the go (e.g. chat, mobile
  checklists), also a 390×844 phone version.
- **Fake data only**: no real customers, license plates, device IDs (IMEI/ICCID) or serial numbers.
  Documented test fixtures/units from your team wiki are fine.

## New screen or redesign

1. Find the app in the registry and open its canvas; the screen goes on its domain's page.
2. If there's no domain for it, create the page and add it to the table above.
3. "Today | proposal" row + note. The proposal follows the app repo's rules (e.g. never disable the
   submit button, all copy through i18n — whatever that repo's `AGENTS.md` says).
4. If it's for a ticket, link the page and rows in its UI design field or heading, with what changes
   in each (`/task-tracker`). The design lives in the canvas; the ticket only links to it.
5. Sharing: the canvas is shared with the organization from its Share menu (the owner does it;
   Claude can't change it).
