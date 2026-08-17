# Legacy → WaveMaker (Claude Code plugin)

A [Claude Code](https://claude.com/claude-code) plugin marketplace for migrating legacy web apps
(JSP/servlet, PHP, .NET, or any server-rendered app) to **WaveMaker AI** and hand-authoring
WaveMaker project files outside WaveMaker Studio, using conventions validated against real Studio
output.

## What's inside

This repo is a Claude Code **marketplace** (`legacy-to-wavemaker`) containing one plugin
(`wavemaker`):

- **Skill `wavemaker-migration`** — the full ruleset, split across five topic references the skill
  routes to on demand:
  - `pages-and-markup` — page shell, `wm-list` templates, binding/pipes, `.navigate()`, layout
    sizing, static data, plan-first workflow
  - `data-variables` — LiveVariable read=POST, column types, custom Java services, ServiceVariables
    & app-state, LiveForm/LiveTable CRUD, runtime filtering
  - `security` — intercept-urls, page vs service ACLs, custom roles query, client auth state
  - `design-tokens` — theme tokens, palette edits, component variants ("appearances")
  - `migration-map` — legacy→WaveMaker mapping table + a pre-flight checklist
  - `ide-sync` — Studio ⇄ IDE sync via the workspace-sync Maven plugin (`init`/`pull`/`push`/`sync`)
- **Command `/migrate-to-wavemaker [target]`** — plans and starts a migration with the conventions
  preloaded.
- **Command `/wavemaker-sync [project]`** — sets up Studio ⇄ IDE synchronization (adds the
  workspace-sync Maven plugin + a SYNC.md) for a WaveMaker project.
- **Agent `wavemaker-authoring`** — a subagent that hand-authors/fixes WaveMaker files to spec and
  JSON-validates them.

## Install (from GitHub)

```bash
/plugin marketplace add wm-ps/legacy-to-wavemaker-claude-plugin
/plugin install wavemaker@legacy-to-wavemaker
```

Then run `/migrate-to-wavemaker src/legacy/checkout`, delegate to the `wavemaker-authoring` agent,
or just start a WaveMaker task and the skill triggers automatically.

## Try locally without installing

```bash
claude --plugin-dir ./wavemaker
```

## Repository layout

```
.claude-plugin/marketplace.json   # marketplace manifest
wavemaker/                         # the plugin
  .claude-plugin/plugin.json
  README.md
  skills/wavemaker-migration/      # SKILL.md + references/*.md
  commands/migrate-to-wavemaker.md
  commands/wavemaker-sync.md
  agents/wavemaker-authoring.md
```

## License

MIT
