---
description: Set up WaveMaker Studio ⇄ IDE synchronization (workspace-sync-plugin) for a WaveMaker project.
argument-hint: [path-to-wavemaker-project]
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Skill
---

Set up **WaveMaker Studio ⇄ IDE sync** for a WaveMaker project using the WorkSpace Sync Maven
plugin. First invoke the `wavemaker-migration` skill and read
`references/ide-sync.md` for the exact plugin coordinates, goals, and gotchas.

Target project: **$ARGUMENTS** (if empty, ask where the WaveMaker project lives — the folder with
`pom.xml` and `.wmproject.properties`).

Do this:

1. **Load the rules** — invoke the `wavemaker-migration` skill; read `references/ide-sync.md`.
2. **Confirm it's a WaveMaker project** — check for `pom.xml` + `.wmproject.properties`; read the
   project's display name from `.wmproject.properties` (you'll pick it during `init`).
3. **Add the plugin to `pom.xml`** (if not already present) — pin the version via a property and
   declare it with **no** build-phase binding so it never runs during a normal build:
   ```xml
   <properties>
     <wavemaker.workspace.sync.version>11.11.5</wavemaker.workspace.sync.version>
   </properties>
   <!-- build/plugins -->
   <plugin>
     <groupId>com.wavemaker.studio</groupId>
     <artifactId>wavemaker-workspace-sync-plugin</artifactId>
     <version>${wavemaker.workspace.sync.version}</version>
   </plugin>
   ```
   Confirm the latest version first:
   `curl -s "https://search.maven.org/solrsearch/select?q=a:wavemaker-workspace-sync-plugin&wt=json"`
   (use the `.ee` variant for Enterprise Studio). Validate the pom stays well-formed XML.
4. **Enable the short goal prefix** — tell the user to add `<pluginGroup>com.wavemaker.studio</pluginGroup>`
   to `~/.m2/settings.xml`, or give them the fully-qualified goal as a fallback.
5. **Write a short `SYNC.md`** in the project with the prerequisites and the exact commands:
   `mvn wavemaker-workspace:init` (one-time; host URL + auth/token + project), then
   `:pull` / `:push` / `:sync`.
6. **Do NOT run `init` yourself** — it needs the user's Studio host and credentials/token. Print the
   commands for the user to run, and remind them: **`pull` before editing, `push`/`sync` after**.

Keep changes scoped to enabling sync. Summarize what you added (pom entry, SYNC.md) and the exact
commands the user runs next.
