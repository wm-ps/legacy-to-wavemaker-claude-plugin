# WaveMaker Migration & Authoring (Claude Code plugin)

Everything needed to migrate a legacy web app (JSP/servlet, or other) to **WaveMaker 11** and to
hand-author WaveMaker project files outside WaveMaker Studio, using conventions validated against
real Studio output.

## What's inside
- **Skill `wavemaker-migration`** — the full ruleset, split across five topic references the skill
  routes to on demand: `pages-and-markup` (page shell, `wm-list` templates, binding/pipes,
  `.navigate()`, layout sizing, static data, plan-first workflow), `data-variables` (LiveVariable
  read=POST, column types, custom Java services, ServiceVariables & app-state, LiveForm/LiveTable
  CRUD, runtime filtering), `security` (intercept-urls, roles query, auth state), `design-tokens`
  (theme tokens, component variants), and `migration-map` (legacy→WaveMaker mapping table +
  pre-flight checklist). Auto-triggers on WaveMaker/migration tasks.
- **Command `/migrate-to-wavemaker [target]`** — plans and starts a migration with the conventions
  preloaded.
- **Agent `wavemaker-authoring`** — a subagent that hand-authors/fixes WaveMaker files to spec and
  JSON-validates them.

## Install
```bash
/plugin marketplace add ./legacy-to-wavemaker
/plugin install wavemaker@legacy-to-wavemaker
```
Then: run `/migrate-to-wavemaker src/legacy/checkout`, delegate to the `wavemaker-authoring` agent,
or just start a WaveMaker task and the skill triggers automatically.

## Try locally without installing
```bash
claude --plugin-dir ./legacy-to-wavemaker/wavemaker
```
Invoke `/wavemaker:migrate-to-wavemaker` or the `wavemaker-migration` skill.
