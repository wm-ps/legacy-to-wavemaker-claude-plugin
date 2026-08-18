---
name: wavemaker-conventions
description: >-
  Conventions and rules for migrating legacy web apps (JSP/servlet, PHP, .NET, or any
  server-rendered app) to WaveMaker AI (React/low-code), AND for hand-authoring any WaveMaker
  project files outside WaveMaker Studio. Use this skill whenever the task involves migrating a
  legacy app to WaveMaker; creating or editing WaveMaker pages/partials by hand (`.html` wm-markup,
  `.variables.json`, `.js`, `.css`); writing WaveMaker LiveVariables, LiveForms, LiveTables,
  ServiceVariables, or navigation actions; authoring custom `@ExposeToClient` Java services;
  building design-token themes or component variants (`design-tokens/overrides/**`); or configuring
  WaveMaker security (intercept-urls, roles query, auth providers). Trigger it even when the user
  just says "WaveMaker", "wm-", "LiveVariable", "design tokens", "shopbit", or names a WaveMaker
  file — hand-authored WaveMaker markup has many non-obvious rules that are easy to get wrong.
---

# WaveMaker Migration & Hand-Authoring Conventions

WaveMaker AI apps are normally built in WaveMaker Studio (a visual designer that generates the
files). When migrating a legacy app or editing a WaveMaker project **by hand**, the generated file
formats have many non-obvious rules — get one wrong and Studio rejects the file or the page renders
blank. This skill captures the rules validated against real WaveMaker Studio output.

## Step 0 — Connect Studio ⇄ local sync FIRST (do this before authoring)

You cannot preview a WaveMaker app without Studio, so hand-authored files must be verified there. The
fastest loop is WaveMaker's **`wavemaker-workspace` Maven plugin** (IDE sync, beta): edit locally,
push, the user refreshes Studio, reports back, repeat — no zip/re-import each round. Full detail and
the dedicated `/wavemaker-sync` command are in [`references/ide-sync.md`](references/ide-sync.md).

**At the very start of any WaveMaker task, ask the user whether Studio⇄local sync is connected. If
not, give them these one-time steps and let them run them — then you drive the ongoing sync.**

Prerequisites: Git, Maven, and a JDK installed; the project exported/opened once in Studio.

One-time setup (**the user runs this — it authenticates**):
1. In an active Studio session, get a token from `https://<your-studio-host>/studio/services/auth/token`
   (enterprise instances differ from `wavemakeronline.com` — use your own host). Token auth is
   preferred over username/password so no password is stored.
2. From the project root, run `mvn wavemaker-workspace:init` once and authenticate with that token.

Ongoing (**you may run these for the user after init**):
- `mvn wavemaker-workspace:pull` — fetch Studio changes into local before editing.
- `mvn wavemaker-workspace:push` — send your local edits to the Studio workspace.
- `mvn wavemaker-workspace:sync` — bidirectional (pull then push).

**Credential boundary (do not cross):** never ask for, type, or store the user's WaveMaker password or
token, and never run `init`/authenticate on their behalf. The user authenticates once themselves; you
only run `pull`/`push`/`sync` against the channel they opened. If sync isn't set up, fall back to
exporting the project and having the user import it, and still verify in Studio.

## Step 0b — Get the base project and PIN its WaveMaker version

**Always ask the user for the base WaveMaker project (a Studio-exported/opened project) before
authoring.** You can't hand-author into nothing — new pages/services are created in Studio, and you
model new files on the base project's existing pages, its imported DB data model, and its
`design-tokens/overrides/**`.

**These rules are validated against WaveMaker `1115.11` — versions drift, so read the base project's
version first and treat the base's OWN generated files as ground truth for anything version-sensitive:**
- `.wmproject.properties` → `studioProjectUpgradeVersion` (e.g. `1115.11`).
- `pom.xml` → the `ai.wavemaker.app:wavemaker-app-parent` `<version>` and the
  `wavemaker.app.runtime.ui.version` property.

If the base differs from 1115.11, don't blindly trust these references for exact file shapes
(LiveVariable/ServiceVariable JSON, page shell, security enums, service wiring). Instead grep the
base project's existing Studio-generated pages/services and copy those shapes — a real generated file
from the target version always beats a documented example. Tell the user the version you detected.

**The full ruleset lives in five topic references — open the one(s) relevant to your task before
authoring files.** The most-often-wrong rules are summarized below; consult the matching reference
for complete detail, JSON shapes, and examples.

| Reference | Covers (original §) |
|---|---|
| [`references/pages-and-markup.md`](references/pages-and-markup.md) | Page shell, `wm-list` templates, item binding, pipes, on-click/`.navigate()`, layout & sizing, static list data, plan-first workflow (§0–§4, §10–§12) |
| [`references/data-variables.md`](references/data-variables.md) | LiveVariable read=POST, column types, custom Java services, ServiceVariables & app-state, LiveForm/LiveTable CRUD, runtime filtering (§5, §7, §13–§16) |
| [`references/security.md`](references/security.md) | intercept-urls, page vs service ACLs, custom roles query, client auth state (§6) |
| [`references/design-tokens.md`](references/design-tokens.md) | Theme tokens, palette edits, component variants / "appearances" (§8–§9) |
| [`references/migration-map.md`](references/migration-map.md) | Legacy→WaveMaker mapping table + the pre-flight checklist (§17 + checklist) |
| [`references/ide-sync.md`](references/ide-sync.md) | Studio ⇄ IDE sync via the workspace-sync Maven plugin (`init`/`pull`/`push`/`sync`) — how hand-authored files get into Studio |

## Migration strategy — map to native first, custom code last

Most legacy server code becomes **native WaveMaker**, not custom code. Before writing anything, place
each legacy feature in the mapping table (reference §17). The high-value moves:

- **JSP/view page** → WaveMaker page (`.html` + `.js` + `.css` + `.variables.json`)
- **Servlet that lists/reads** → **LiveVariable** (`read`) bound to the generated DB CRUD API
- **DAO + hand-written SQL** → generated CRUD REST API (this also erases the SQL-injection surface)
- **Servlet doing CRUD** → **LiveForm/LiveTable** or the entity API `create/update/delete`
- **Search / category / `LIKE` filter** → LiveVariable `filterExpressions` + `.update()` (reference §16)
- **Non-CRUD servlet (checkout, business logic)** → custom `@ExposeToClient` Java service + a
  ServiceVariable (reference §13–§14)
- **HTTP session object (cart)** → app-scoped `wm.Variable` + `App.*` helpers (reference §14)
- **Login + session + role check** → DB security provider + `wm-login` + roles query (reference §6)

## Critical rules that are easy to get wrong

These are the mistakes that cost the most rework. Full context in the reference.

1. **Page shell** — `wm-header` and `wm-footer` are **direct children of `wm-page`**, NOT nested in
   `wm-content`; no `wm-left-panel`; nav lives in the `header` partial; wrap the body in one root
   container; page-content `columnwidth="12"`. (§0)
2. **`wm-list` item template** — use `<wm-listtemplate ...>` (Studio-native) or
   `<ng-template #listtemplate="" let-item="item" let-$index="index">` — note the reference name is
   **lowercase `#listtemplate=""`**, not camelCase. (§1)
3. **Item binding** — `bind:<Var>.dataSet[$i].field` (indexed, drop the `Variables.` prefix inside
   the template) or `bind:item.field`. (§2)
4. **No string concatenation in bind expressions** — WaveMaker has no `+`. Use pipes:
   `bind:x | prefix:'$'`, `bind:x | suffix:' items'` (chainable, static args only). (§3)
5. **Navigation uses `.navigate()`**, not `.invoke()`: `Actions.goToPage_Shop.navigate()`; params via
   `.navigate({productId: id})`, read on the target with `Page.pageParams`. (§4)
6. **A LiveVariable read is a POST** to `/{Entity}/filter` — so anonymous access needs the service
   permitted for **POST** (`"httpMethod": null` in intercept-urls; a GET-only rule 401s). Page ACL and
   service ACL are separate. (§5–§6)
7. **Roles query** (custom): placeholder is **`:username`** (lowercase), `queryType` is
   **`"NATIVE_SQL"`**, and clear `rolePropertyName`/`roleColumnName` to `""`. (§6c)
8. **Column type for double is `"double"`** (not `float`) in a LiveVariable `propertiesMap`. (§7)
9. **Styling is via component variants** — styled widgets carry **both** `variant="<name>"` and
   `class="<component>-<name>"`; variants are defined in
   `design-tokens/overrides/components/<comp>/<comp>.json` under `appearances`. Sizing uses
   `width/height = fill|hug|%|px`. Palette edits go in **both** `app.override.css` `:root` and
   `color.light.json`. (§8–§10)
10. **Custom Java service** — `@ExposeToClient` class; method name prefix sets the HTTP verb; autowire
    generated services with `@Qualifier("<dbservice>.<Entity>Service")`; wrap in
    `@Transactional("<dbservice>TransactionManager")`; Studio generates the controller +
    `javaservice-rest-patch.json`. Call it from a page via a ServiceVariable
    (`setInput`/`invoke`). (§13–§14)

## Working method

- **Register** new pages in `pages/pages-config.json` and navigation actions in `app.variables.json`.
- **Style with design tokens by default** — deliver Studio-native component variants (`variant="<v>"`
  + `class="<component>-<v>"`) backed by the full token sources (`design-tokens/overrides/global/**`
  AND `overrides/components/**/*.json` appearances) plus the compiled `app.override.css`, as part of
  the migration — NOT as a follow-up, and not ad-hoc utility CSS. If you ever fall back to plain
  utility classes, surface that as an explicit trade-off decision rather than defaulting it silently.
- **New pages/services must be created in WaveMaker Studio** (or via a full project import) before
  editing them by hand; custom Java code can be authored locally. Studio regenerates
  `design-tokens/app.override.css` from the `overrides/**` token JSON on import.
- **Cannot preview a WaveMaker app without Studio.** Author files to spec, JSON-validate everything,
  and treat a Studio import as the verification step. Model new pages on the existing samplePage /
  Login page and on entities in the imported DB's published data model.
- **Run the pre-flight checklist** in [`references/migration-map.md`](references/migration-map.md)
  before handing off any page.

When in doubt about an exact JSON shape or a widget attribute, open the matching topic reference
(see the table above) and match the documented example rather than guessing — hand-authored
WaveMaker files are unforgiving of small format errors.
