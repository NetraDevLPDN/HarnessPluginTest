<p align="center">
  <img src="public/logo/dsh-proof-icon.png" width="120" alt="dsh-proof logo">
</p>

<h1 align="center">dsh-proof</h1>

<p align="center">
  <strong>A verification gate for DeepSeek Harness.</strong><br>
  Run typecheck, lint, and tests after every edit — and keep the agent working until they pass.
</p>

<p align="center">
  <a href="#features">Features</a> •
  <a href="#quick-start">Quick start</a> •
  <a href="#usage">Usage</a> •
  <a href="#configuration">Configuration</a> •
  <a href="#security">Security</a> •
  <a href="#documentation">Docs</a> •
  <a href="#contributing">Contributing</a>
</p>

<p align="center">
  <a href="https://github.com/NetraDevLPDN/dsh-proof/actions/workflows/ci.yml"><img src="https://github.com/NetraDevLPDN/dsh-proof/actions/workflows/ci.yml/badge.svg?branch=main" alt="CI"></a>
  <a href="https://github.com/NetraDevLPDN/dsh-proof/releases"><img src="https://img.shields.io/github/v/release/NetraDevLPDN/dsh-proof?style=flat-square&label=release" alt="Release"></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/NetraDevLPDN/dsh-proof?style=flat-square" alt="License: MIT"></a>
  <a href="https://nodejs.org/"><img src="https://img.shields.io/badge/node-%3E%3D22.19-339933?style=flat-square&logo=node.js&logoColor=white" alt="Node >=22.19"></a>
  <a href="https://github.com/deepseek-ai/deepseek-harness"><img src="https://img.shields.io/badge/DSH-%3E%3D0.2.1--rc.2-4D6BFE?style=flat-square" alt="DSH >=0.2.1-rc.2"></a>
  <a href="https://www.typescriptlang.org/"><img src="https://img.shields.io/badge/TypeScript-5.9-3178C6?style=flat-square&logo=typescript&logoColor=white" alt="TypeScript 5.9"></a>
  <a href="https://github.com/NetraDevLPDN/dsh-proof/stargazers"><img src="https://img.shields.io/github/stars/NetraDevLPDN/dsh-proof?style=flat-square&label=stars" alt="GitHub stars"></a>
  <a href="https://github.com/NetraDevLPDN/dsh-proof/pulls"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square" alt="PRs welcome"></a>
</p>

---

## Features

The most expensive failure mode of a coding agent is not bad syntax — it is **premature completion**. The agent edits three files, writes “done”, and you discover by hand that typechecking fails and two tests are broken.

`dsh-proof` targets the exact moment the agent tries to stop: it turns the project’s checks into a **turn-exit condition**.

- **Zero configuration** on a Node project whose package manager is available.
- **Bounded correction loop** — `maxReruns` limits retries, so there is no infinite loop.
- **`verify` tool** — the agent can run the same checks itself before claiming completion.
- **Prompt section** — the agent knows which checks are active and that they block the turn.
- **Host-only** — no browser bundle and no UI dependency.

## Quick start

### Prerequisites

- DSH `>=0.2.1-rc.2 <0.3.0` (declared in `engines.dsh`).
- Node.js `>=22.19`.
- The project’s package manager (`npm`, `pnpm`, `yarn`, or `bun`) available on `PATH`.
- A `package.json` with at least one of `typecheck`, `lint`, `test`, or `check`, or explicit `checks`.

### Desktop app

1. Open the left sidebar (**Ctrl+B** if it is collapsed).
2. Open the **Plugins** panel (the pinwheel icon below “New session”).
3. Click **Add plugin**.
4. Paste `dsh-proof` (npm), or the latest `.tgz` asset from [Releases](https://github.com/NetraDevLPDN/dsh-proof/releases/latest).
5. Accept the trust checkbox, install, and restart DSH if prompted.

> The `desktop` profile is managed by the Electron app; `dsh plugin --profile desktop` is intentionally refused by the CLI.

### CLI profile (`web`, `headless`)

```sh
dsh plugin --profile web add dsh-proof
```

Or, from this repository:

```sh
pnpm install
pnpm test
pnpm pack
dsh plugin --profile web add ./dsh-proof-0.1.3.tgz
```

Restart the profile afterwards: manifest and export changes are not hot-reloaded.

## Usage

### Automatic

After a `write`, `edit`, `apply_patch`, or similar call, the gate activates at the end of the turn:

1. it runs the checks in the session’s working directory;
2. if everything passes, the turn ends normally;
3. if a check fails, the output is sent back to the agent via `agent.steer(...)`, which **prevents the turn from closing** and opens a correction step;
4. `maxReruns` bounds the retries (2 by default).

Success is silent. A failure produces a `[Proof gate]` message in the conversation.

### `verify` tool

```text
verify()             # run every check
verify(only="test")  # run only the named check
```

It returns `{ passed, skipped, cwd, checks[] }` plus a compact text rendering: overall status, check name, exit code, and output on failure.

### What you will see

- **During a successful run** — nothing in the transcript; the turn simply ends.
- **On failure** — a synthetic `[Proof gate]` user message containing the failing command output, followed by another agent step.
- **On demand** — the `verify` tool result with `PASS` or `FAIL`.

## Configuration

The defaults work on a Node project. For explicit settings, add a **top-level override** in `~/.dsh/profiles/<profile>/cordis.patch.yml`:

```yaml
- id: proof
  config:
    checks:
      - name: typecheck
        command: npm run typecheck
        timeoutMs: 120000
      - name: test
        command: npm test
        timeoutMs: 180000
```

> The bundle already inserts the `proof` row. Use `- id: proof` to override it; a second `insert` with the same `id` would create a duplicate at startup.

| Key | Default | Purpose |
|---|---|---|
| `enabled` | `true` | Master switch. |
| `autoDetect` | `true` | Detect `typecheck`, `lint`, `test` (or `check`) in `package.json`. |
| `checks` | `[]` | List of `{ name, command, timeoutMs }`; takes precedence over auto-detection. |
| `cwd` | `''` | Directory for the checks; relative to the session directory. |
| `maxReruns` | `2` | Maximum corrective retries per turn (`0`: verify without retrying). |
| `outputChars` | `6000` | Maximum output size returned to the model. |
| `watchShell` | `false` | Treat mutating shell commands as edits too. |
| `mutatingShellPattern` | (provided regex) | Pattern for shell commands considered edits. |
| `includeSubagents` | `false` | Apply the gate to subagent sessions. |
| `stopOnFirstFailure` | `true` | Stop after the first failing check. |

### Auto-detection

The package manager is chosen from the lockfile (`pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`, otherwise npm), **but only if it is actually available on `PATH`**. Otherwise the plugin falls back to `npm`. The following scripts become commands:

- `typecheck` → `npm run typecheck` (or the detected manager);
- `lint` → …
- `test` → …
- `check` (only when none of the three above exists).

## How it works

1. The plugin observes session events. A call to `write`, `edit`, `str_replace_editor`, `apply_patch`, and similar tools marks the workspace as edited.
2. At `agent/turn-stopping`, if files changed, the checks run in the session’s working directory.
3. On failure, the output is sent back to the agent, which must fix it before finishing.
4. The turn only ends without verification when the checks pass, or when `maxReruns` is exhausted (a warning is logged).

## Security

The checks are **shell commands executed with the user’s privileges, outside the DSH sandbox** — like an `npm test` you run yourself. Only configure commands you trust.

The plugin does not modify files itself. However, the commands it runs may do so (formatters, snapshot updates). Their output is visible to the model: avoid printing secrets from a check.

## Troubleshooting

- **`'pnpm' is not recognized`** — the lockfile manager is not on `PATH`. The plugin now falls back to `npm`; otherwise install pnpm or use explicit `checks` with `npm`.
- **`Verification skipped`** — there is no `package.json` and no configured check. Add a `test` script or explicit `checks`.
- **The gate does not trigger after an edit** — the edit may have been made through `pwsh`/`bash`. Enable `watchShell: true`, or use the editing tools.
- **Checks are too slow** — adjust `timeoutMs` or reduce the list of checks.

## Known limitations

- Only known editing tools trigger the gate; shell command tracking is an optional heuristic.
- No automatic rollback: the plugin retries the correction, it does not restore files.
- Checks run sequentially; a test suite that exceeds the timeout must raise `timeoutMs`.
- Tracking is per session; subagents are ignored by default.

## Development

```sh
pnpm install
pnpm build      # tsc → lib/
pnpm test       # build + tests (node:test)
pnpm pack       # installable archive
```

Structure:

```text
src/index.ts        Cordis entry: verify tool, prompt section, hooks
src/gate.ts         the gate state machine (testable without DSH)
src/runner.ts       bounded command execution
src/types.ts        configuration + the “proof” message source
src/__tests__/      unit tests
cordis.patch.yml    bundle configuration layer
```

## Contributing

Issues and pull requests are welcome. Read [CONTRIBUTING.md](CONTRIBUTING.md) for the full engineering, testing, security, and release rules.

1. Fork the repository and create a topic branch.
2. Run `pnpm install` and `pnpm test`.
3. Keep commits in [Conventional Commits](https://www.conventionalcommits.org/) format.
4. Open a pull request describing the behavior change and the tests that cover it.

CI runs on Ubuntu and Windows. Release steps are documented in [PUBLISHING.md](PUBLISHING.md).

## Documentation

- [Contributing guide](CONTRIBUTING.md)
- [Changelog](CHANGELOG.md)
- [Publishing guide](PUBLISHING.md)
- [Security policy](SECURITY.md)
- [DSH plugin packaging guide](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md)

## Release

GitHub releases and npm publication are automated: pushing a `v*` tag triggers [release.yml](.github/workflows/release.yml), which runs the tests, checks that the tag matches `package.json`, packs the plugin, publishes to npm through OIDC trusted publishing with provenance, and creates the release with the `.tgz` asset.

## License

MIT.
