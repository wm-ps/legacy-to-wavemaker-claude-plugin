# IDE Sync — WaveMaker Studio ⇄ local IDE (workspace-sync-plugin)

How to round-trip a WaveMaker project between **Studio** and a **local IDE** (IntelliJ / Eclipse /
VS Code) so hand-authored changes (the ones this skill produces) can be pushed back into Studio —
and the reverse, pulling Studio's changes down. This is the companion to the "you can't preview
without Studio" caveat: sync is how hand-authored files get *into* Studio to be verified/previewed.

Official docs: https://docs.wavemaker.com/learn/how-tos/synchronizing-wavemaker-apps-ides/

Companion files: [`pages-and-markup.md`](pages-and-markup.md), [`data-variables.md`](data-variables.md),
[`security.md`](security.md), [`design-tokens.md`](design-tokens.md), [`migration-map.md`](migration-map.md).

---

## What it is
The **WaveMaker WorkSpace Sync Plugin** is a Maven plugin
(`com.wavemaker.studio:wavemaker-workspace-sync-plugin`) that talks to a Studio workspace over HTTP.
It gives four goals:

| Goal | Direction | Use |
|---|---|---|
| `wavemaker-workspace:init` | — | one-time: set Studio host, auth, and which project |
| `wavemaker-workspace:pull` | Studio → IDE | get the latest Studio changes locally |
| `wavemaker-workspace:push` | IDE → Studio | send your local edits up to Studio |
| `wavemaker-workspace:sync` | both | `pull` then `push`, in order |

## Add it to a project's `pom.xml`
Pin the version and declare the plugin (no build-phase binding — it only runs when you invoke a
goal, so it never fires during a normal `mvn package`):
```xml
<properties>
  <wavemaker.workspace.sync.version>11.11.5</wavemaker.workspace.sync.version>
</properties>
...
<build>
  <plugins>
    <plugin>
      <groupId>com.wavemaker.studio</groupId>
      <artifactId>wavemaker-workspace-sync-plugin</artifactId>
      <version>${wavemaker.workspace.sync.version}</version>
    </plugin>
  </plugins>
</build>
```
- Latest published version: **`11.11.5`** (Maven Central). Enterprise Studio: use the **`11.11.5.ee`**
  artifact instead.
- Confirm the newest version:
  `curl -s "https://search.maven.org/solrsearch/select?q=a:wavemaker-workspace-sync-plugin&wt=json"`.

## Make the short `wavemaker-workspace:` prefix resolve
Maven resolves a goal prefix from configured **pluginGroups**, not from the pom. Add WaveMaker's
group once to **`~/.m2/settings.xml`** (create the file if absent):
```xml
<settings>
  <pluginGroups>
    <pluginGroup>com.wavemaker.studio</pluginGroup>
  </pluginGroups>
</settings>
```
Without it, use the **fully-qualified** goal instead:
```bash
mvn com.wavemaker.studio:wavemaker-workspace-sync-plugin:11.11.5:init
```

## Prerequisites
- **Maven** and **Git** on PATH.
- A Studio account with the project in your workspace.
- **Windows:** set the IDE line separator to Unix (`\n`) or Studio round-trips will show spurious
  whole-file diffs.

## Flow
```bash
# one-time, from the project directory
mvn wavemaker-workspace:init      # prompts: host URL, auth (user/pass OR token), project

# day to day
mvn wavemaker-workspace:pull      # before you start editing
mvn wavemaker-workspace:push      # after you finish (or `sync` for both)
```
`init` auth accepts username/password **or** a token from
`https://<studio-host>/studio/services/auth/token` (log into Studio first, then open that URL).

## How this fits the migration workflow
1. Hand-author the WaveMaker files per this skill (pages, variables, services, tokens).
2. `wavemaker-workspace:push` (or import the zip) to get them into Studio.
3. In Studio, do the verification steps the skill flags — LiveVariable relation regen, custom Java
   service creation, security provider via the Security wizard.
4. `wavemaker-workspace:pull` to bring Studio's normalized versions back into your IDE/repo.

> Don't hand-author what Studio round-trips better: new **Java services** (needs
> `service_<Service>.spring.xml` wiring) and the **security auth provider block** are best created in
> Studio, then pulled down — see [`data-variables.md`](data-variables.md) §13 and
> [`security.md`](security.md) §6.

## Verifying in a run-preview — the URL is EPHEMERAL
A Studio run-preview URL looks like `https://<host>/run-<id>/ent<...>/<Project>_master/react-preview/<Page>`.
The `run-<id>` segment is tied to a **live run session and expires** — the same URL returns **HTTP 404**
once that session ends, and (with security enforced) redirects to Login for an anonymous browser. So:
- Don't treat a pasted run URL as a durable link; re-run the app in Studio to get a fresh one before
  each verification pass.
- After a `push`, reload/re-run in Studio so it redeploys and recompiles (`app.override.css`
  regeneration, `font.config.js` fonts) — the previous preview instance will not reflect the push.
