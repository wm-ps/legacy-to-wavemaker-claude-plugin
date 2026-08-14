---
description: Plan and start migrating a legacy app (or one feature/page) to WaveMaker AI, following the wavemaker-migration conventions.
argument-hint: [path-to-legacy-app-or-feature]
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Skill
---

You are migrating a legacy web app to **WaveMaker AI**. First invoke the `wavemaker-migration`
skill so the full conventions and the legacy→WaveMaker mapping are loaded, then follow them.

Target of this migration: **$ARGUMENTS** (if empty, ask the user what legacy app, feature, or page
to migrate, and where the WaveMaker project lives).

Do this:

1. **Load the rules** — invoke the `wavemaker-migration` skill and read the relevant topic
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
5. **Verify what you can** — JSON-validate every file; run the reference's pre-flight checklist.
   Remember a WaveMaker app can't be previewed without Studio, so treat a Studio import as the
   verification step and flag anything that needs Studio (new page/service creation, token
   recompile).

Keep changes scoped to the requested target; summarize the legacy→WaveMaker mapping you applied and
list anything the user must do in Studio.
