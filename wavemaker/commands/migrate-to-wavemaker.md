---
description: Plan and start migrating a legacy app (or one feature/page) to WaveMaker AI, following the wavemaker-conventions conventions.
argument-hint: [path-to-legacy-app-or-feature]
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Skill
---

You are migrating a legacy web app to **WaveMaker AI**. First invoke the `wavemaker-conventions`
skill so the full conventions and the legacy→WaveMaker mapping are loaded, then follow them.

Target of this migration: **$ARGUMENTS** (if empty, ask the user what legacy app, feature, or page
to migrate, and where the WaveMaker project lives).

Do this:

0. **Connect Studio ⇄ local sync FIRST** — before authoring anything, ask the user whether the
   `wavemaker-workspace` IDE sync is set up (see the skill's "Step 0"). If not, give them the one-time
   `mvn wavemaker-workspace:init` steps and let them authenticate; then you drive `pull`/`push`/`sync`
   each round so they can watch output in Studio and chat corrections. Never handle their password/token
   or run `init` yourself. If they decline, fall back to export → user imports, and verify in Studio.
0b. **Get the base project and PIN its version** — ask for the base WaveMaker project, then read its
   version (`.wmproject.properties` `studioProjectUpgradeVersion`, and the `wavemaker-app-parent`
   version in `pom.xml`). The conventions are validated against `1115.11`; if the base differs, grep
   the base's own Studio-generated files for exact shapes instead of trusting the references. Tell the
   user the detected version. (See the skill's "Step 0b".)
1. **Load the rules** — invoke the `wavemaker-conventions` skill and read the relevant topic
   reference(s) under `references/` (pages-and-markup, data-variables, security, design-tokens,
   migration-map) before authoring any file.
2. **Map, don't rewrite** — for each legacy feature, place it in the mapping table in
   `references/migration-map.md`: servlet-read
   → LiveVariable, DAO+SQL → generated CRUD API, non-CRUD servlet → custom `@ExposeToClient` Java
   service, session object → app-scoped variable, login/roles → DB security provider + roles query.
   Prefer native WaveMaker over custom code.
3. **Inspect both sides** — read the legacy source for the feature AND the WaveMaker project's
   existing pages, imported DB published data model, and `design-tokens/overrides/**` so new work
   matches existing conventions and entity/column names.
4. **Author to spec** — produce the WaveMaker files (`.html` wm-markup, `.variables.json`, `.js`,
   `.css`, Java services, token variants), register pages in `pages/pages-config.json` and nav
   actions in `app.variables.json`, and configure security/anonymous access as needed.
   - **LiveVariables**: emit Studio's **full JSON envelope** (data-variables.md §7) — `"type"` is the
     **entity simple name** (not the FQN, which goes in `"package"`), with `tableName`, `entityName`,
     `matchMode`, `orderBy`, and full per-column metadata copied from the published data model. A
     trimmed envelope binds wrong.
   - **Theme**: derive the palette/fonts **from the legacy app's CSS/SCSS** (design-tokens.md §8·0) so
     it matches — grep the legacy static assets for the brand/accent colors; state the mapping. Don't
     invent a palette.
   - **Custom Java service**: author the non-CRUD backend (checkout/business logic) with its §13a
     wiring (pom source-root, `service_<Service>.spring.xml`, `@Service` stereotype). Don't ship a
     dead button. Docs: https://docs.wavemaker.com/learn/app-development/services/java-services/java-service/
   - **Security**: actually **enable** it — `enforceSecurity: true` + a DATABASE provider (security.md
     §6e) plus the prepared `intercept-urls`/`roles`. Prepared ACLs alone do nothing. Docs:
     https://docs.wavemaker.com/learn/app-development/app-security/
5. **Verify what you can** — JSON-validate every file; run the reference's pre-flight checklist.
   Remember a WaveMaker app can't be previewed without Studio, so treat a Studio import as the
   verification step and flag anything that needs Studio (new page/service creation, token
   recompile).
6. **On a "migrate everything at once" full run, do NOT silently defer** the theme match, the Java
   service, or enabling security — those three are what full runs tend to drop. If a real constraint
   forces staging one (e.g. a read-only DB blocks write wiring), say so explicitly and keep the task
   open; never let a skipped task read as done.

Keep changes scoped to the requested target; summarize the legacy→WaveMaker mapping you applied and
list anything the user must do in Studio.
