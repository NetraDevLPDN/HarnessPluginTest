# Contributing to dsh-proof

This document is the project handbook: architecture, coding rules, tests, security, packaging, and release process. Read it before changing the plugin.

## 1. Scope and principles

`dsh-proof` is a **host-only** plugin for DeepSeek Harness (DSH). It turns a project’s typecheck, lint, and test commands into a condition for ending an agent turn.

Non-negotiable principles:

- **Proof before completion.** The gate must never report success when a configured check failed or timed out.
- **Fail open, not silent.** If the plugin cannot inspect a workspace or run a check, log the reason and let the turn close; do not block the agent on plugin bugs.
- **No hidden magic.** Checks come only from explicit configuration or from scripts deliberately named in `package.json`. Never derive commands from model input.
- **Small surface.** Keep the plugin host-only, dependency-light, and free of client/UI code.
- **Public seams only.** Use documented DSH APIs (`tools`, `systemPrompt`, `session/event`, `agent/turn-stopping`, `agent.steer`). Do not import private package internals.
- **Bounded work.** Every command has a timeout and a bounded output. Every retry loop has a maximum.

## 2. Repository map

| Path | Responsibility |
|---|---|
| `src/index.ts` | Cordis entry: config schema, `verify` tool, prompt section, event wiring. |
| `src/gate.ts` | Session state machine, check resolution, turn-stop enforcement. No Cordis imports. |
| `src/runner.ts` | Bounded shell execution, timeout/abort handling, output truncation. |
| `src/types.ts` | Public config types, defaults, and the `proof` message-source identity. |
| `src/__tests__/` | `node:test` unit and wiring tests. |
| `cordis.patch.yml` | Bundle patch that inserts the `proof` row. |
| `public/logo/` | Source logo and the optimized 512×512 plugin icon. |
| `.github/workflows/ci.yml` | Ubuntu + Windows test matrix. |
| `.github/workflows/release.yml` | Tag-triggered tests, npm publication, GitHub release. |
| `README.md` | User-facing documentation. |
| `CHANGELOG.md` | User-visible changes, one section per version. |
| `PUBLISHING.md` | Release checklist and trusted-publishing details. |
| `SECURITY.md` | Threat model and vulnerability reporting. |

## 3. Toolchain and setup

Requirements:

- Node.js `>=22.19`.
- pnpm `11`.
- DSH `>=0.2.1-rc.2 <0.3.0`.
- TypeScript 5.x and `@types/node` 22.

```sh
pnpm install
pnpm test
pnpm build
pnpm pack
```

Rules:

- Do not add a runtime dependency unless it is strictly necessary; prefer Node built-ins.
- Keep `dependencies` empty. Runtime DSH packages are peers.
- Keep `peerDependencies` and `devDependencies` aligned and on the same release channel.
- Do not change `engines.dsh` or peer ranges without testing against a real DSH profile.

## 4. Architecture rules

### `src/index.ts`

- Wiring only: config schema, tool registration, prompt section, event subscriptions.
- Do not put check logic or state transitions here.
- Every registration must be owned by the plugin fiber: `ctx.effect(() => ctx.tools.register(...))`, `ctx.on(...)`, or a disposable returned by the API.
- Keep the Cordis plugin `name` (`proof`) identical to the patch row `id`.

### `src/gate.ts`

- Must remain independent of Cordis so it can be unit-tested directly.
- Inject the command runner and logger through `GateOptions`; never spawn processes directly.
- State is keyed by session id and reset on `turn/start`.
- Delete state on `session/disposed`.
- Resolve checks at most once per `package.json` mtime; do not stat/read on every token.
- Skip sessions without a working directory; never fall back to the DSH installation directory for automatic checks.
- Treat `timedOut` as failure even when the exit code is `0`.
- On check failure, return a `UserMessage` with `source.kind = 'proof'` and a bounded summary.
- Never throw out of `onTurnStopping`; catch, log, and allow the turn to close.

### `src/runner.ts`

- `runCommand` never rejects: spawn errors become non-zero outcomes.
- Always enforce a timeout and observe the caller’s `AbortSignal`.
- Strip ANSI escape codes, normalize line endings, and truncate with head + tail.
- On Windows, kill the process tree with `taskkill`; on POSIX, kill the process group.
- Do not inherit a shell through an unsafe path; commands are operator-authored.
- Never log raw environment variables or credentials.

### Event flow

1. `session/event` → `tool/call` for `write`, `edit`, `str_replace_editor`, `apply_patch`, `multiedit`, `notebook_edit`.
2. Optional `watchShell` → treat matching `pwsh`/`bash` commands as edits.
3. `agent/turn-stopping` → run checks if dirty or a previous verification is pending.
4. On failure → `agent.steer(...)` with the failing output, up to `maxReruns`.
5. `session/disposed` → drop state.

Do not add work to per-token or per-message hot paths.

## 5. Coding conventions

- TypeScript strict mode; ESM with `module: NodeNext`.
- Relative imports in emitted code use explicit `.js` specifiers.
- Prefer `interface` for object shapes and `type` for unions.
- No `any` in production code. Tests may use `as unknown as Agent` / `as any` only for minimal fakes.
- No unused locals, parameters, or imports (`noUnusedLocals`, `noUnusedParameters` are on).
- Keep `src/gate.ts` free of `@deepseek-ai/cordis` imports.
- Use small pure helpers; keep side effects at the edges.
- Comments explain **why**, not what.
- Every timer, listener, child process, and registration must have an owner and a cleanup path.
- Handle cancellation at every await boundary in command execution.
- Prefer explicit error messages with the check name and exit status; never include secrets.

### Naming

- Tools: lowercase verb (`verify`).
- Config keys: camelCase (`maxReruns`, `watchShell`).
- Check names: short, stable, user-facing (`typecheck`, `test`).
- Session message source: `proof`.

## 6. Configuration rules

- Add every user-facing knob to the `Config` schema with a safe default.
- New keys must be optional at runtime and documented in the README table.
- Explicit `checks` always win over auto-detection.
- Auto-detection may use a lockfile’s package manager **only if that command exists on `PATH`**; otherwise fall back to `npm`.
- Clamp `timeoutMs` to at least 1000 ms.
- `cwd` is relative to the session directory unless absolute.
- Subagents are skipped unless `includeSubagents` is true.
- Shell mutation tracking is opt-in; keep it heuristic, not authoritative.

### Profile patch rule

When the bundle is installed, the `proof` row already exists. To override it, use a top-level entry:

```yaml
- id: proof
  config:
    # ...
```

Never add a second `insert` with `id: proof`; it creates a duplicate at startup. Config replacement is whole-value, not a deep merge.

## 7. Testing rules

Run:

```sh
pnpm test
```

Requirements:

- Use `node:test` and `node:assert/strict`.
- Test pure logic and wiring separately: `gate.ts` with a fake runner/agent, `index.ts` with a fake Cordis context.
- Use temporary directories (`mkdtempSync`) and remove them in `finally`.
- No network, no real DSH instance, no global state changes in tests.
- Fake agents expose only `session.id`, `session.header`, and `steer`.
- Cover failure paths, not just happy paths: timeout, abort, missing cwd, subagents, rerun exhaustion, duplicate/spurious events.
- Every bug fix gets a regression test.
- Keep the suite fast (currently under 2 seconds locally) and cross-platform (Ubuntu + Windows in CI).
- Do not reduce the number of tests; currently 16.

Manual verification for behavior changes:

```powershell
$env:DSH_HOME = "C:\Users\DELL\AppData\Local\Temp\dsh-proof-check"
$env:PATH = "<pnpm-bin>;$env:PATH"
node <dsh>/lib/bin.js plugin --profile check add <tarball-or-package>
node <dsh>/lib/bin.js --profile check --dump-config
node <dsh>/lib/bin.js --profile check --help
```

Expect a `# == dsh-proof` layer, `id: proof`, and a clean boot.

## 8. Commit conventions

Use Conventional Commits, imperative mood, English:

| Type | Use for |
|---|---|
| `feat` | User-visible capability |
| `fix` | Bug fix |
| `test` | Tests only |
| `docs` | README, comments, guides |
| `refactor` | Internal change without behavior change |
| `perf` | Performance |
| `ci` | Workflows |
| `chore` | Version bumps, metadata, tooling |

Examples:

```text
feat: expose verify tool and enforce checks at turn stop
fix: fall back to npm when pnpm is unavailable
test: cover session disposal and timeout handling
docs: document configuration and trusted publishing
ci: publish npm package from release workflow
chore: prepare 0.1.3 for automated npm publishing
```

Rules:

- One logical change per commit.
- Do not commit `node_modules/`, `lib/`, `*.tgz`, logs, or local profile files.
- Do not commit generated build output; the release build produces it.
- Do not use `wip` or vague messages on `main`.

## 9. Branches and pull requests

- `main` is the release branch.
- Feature and fix work goes on a topic branch, then a pull request.
- Direct pushes to `main` are reserved for release chores and urgent CI fixes.
- Do not force-push `main`.
- A PR must pass CI on Ubuntu and Windows.
- Describe the behavior change, the user-visible impact, and the tests that cover it.
- Never include credentials, tokens, or personal DSH profile data in a PR.

## 10. Versioning and releases

Versioning is SemVer. Pre-1.0: breaking changes may ship in a minor, but prefer a patch for fixes and documentation.

Release procedure:

1. Update `package.json` version.
2. Add a `CHANGELOG.md` section for that exact version.
3. Update version references in README/PUBLISHING when they are user-facing.
4. Run `pnpm test` and `npm pack --dry-run`; inspect the file list.
5. Verify installation in a **fresh** `DSH_HOME` profile.
6. Commit the release preparation.
7. Create an annotated tag `vX.Y.Z` and push it.
8. Let `release.yml` run tests, check the tag/version, pack, publish to npm via OIDC, and create the GitHub release.

Hard rules:

- The tag **must** equal `v` + `package.json.version`.
- Never publish the same version twice; bump before tagging.
- Never move or delete a tag after a release exists. If a release is wrong, publish a new patch version.
- Never publish from a dirty working tree.
- Never commit an npm token; publication uses OIDC Trusted Publishing.
- The first OIDC publish validates the Trusted Publisher configuration; it must happen before npm’s displayed deadline.
- Manual `npm publish` is a fallback and requires 2FA OTP.
- Do not attach a locally built tarball to a release unless it was built from the tagged commit.

## 11. Packaging rules

`package.json` must keep:

- `type: "module"`, `main: "lib/index.js"`, `types: "lib/types/index.d.ts"`.
- `exports` for `.`, `./cordis.patch.yml`, and `./package.json`.
- `dsh.bundle.patch: "./cordis.patch.yml"`.
- `engines.node` and `engines.dsh`.
- `files` whitelist: `lib`, the plugin icon, patch, README, CHANGELOG, SECURITY.
- `icon` pointing to a PNG/JPEG/WebP/SVG **inside the package**, at most 256 KiB. The current icon is 512×512 and about 126 KiB.
- `prepare` and `prepublishOnly` build scripts.
- No tests in the published tarball.

Before tagging:

```sh
pnpm pack
tar -tf dsh-proof-*.tgz
```

The archive must contain `lib/`, `cordis.patch.yml`, `README.md`, `CHANGELOG.md`, `SECURITY.md`, `LICENSE`, `package.json`, and `public/logo/dsh-proof-icon.png`. It must not contain `node_modules`, `src/__tests__`, or stray `.tgz` files.

## 12. Documentation rules

- `README.md` is English and user-facing. Keep the header, badges, navigation, install commands, config table, and security warnings current.
- `CHANGELOG.md` records every user-visible change; do not rewrite released sections.
- `PUBLISHING.md` is the release runbook; keep the trusted-publisher and tag instructions accurate.
- `SECURITY.md` describes the unsandboxed command model and the reporting path.
- Code examples must be runnable; do not document options that do not exist.
- When behavior changes, update the README in the same PR.

## 13. Security rules

- Checks run with the user’s privileges, **outside** the DSH sandbox. Treat every configured command as trusted code.
- Never build a shell command from model-provided text. The model may only choose a configured check name.
- Do not print environment variables, tokens, or credentials in check output.
- Treat check output as untrusted diagnostic data; the steering message says so.
- Do not add telemetry, network calls, or remote execution.
- Review dependency changes; keep the runtime dependency set empty.
- Report vulnerabilities privately via the repository’s security advisory.

## 14. Compatibility rules

- Target DSH `>=0.2.1-rc.2 <0.3.0`.
- Use public APIs only; no private fields or internal imports.
- Re-check `defineTool`, event payloads, and message types against the installed DSH version when bumping.
- Keep peer ranges on the same release channel as the tested DSH build.
- Test `--dump-config` and a real boot in a fresh profile, not only typechecking.
- When DSH changes, update the README compatibility note and the changelog.

## 15. Performance rules

- Verification runs at turn end only; never per token or per message.
- Cache package.json detection by mtime.
- Run checks sequentially; do not parallelize them by default.
- Cap output with `outputChars` and per-check `timeoutMs`.
- Bound reruns with `maxReruns`; never add an unbounded loop.
- Keep the prompt section short and static enough to preserve cache stability.

## 16. Rules for AI agents

- Read this file and the relevant source before editing.
- Never modify the user’s real `~/.dsh` profile or credentials unless explicitly asked.
- Use an isolated `DSH_HOME` for integration tests.
- Never commit, push, tag, publish, or move a tag without explicit instruction.
- Never force-push `main` or rewrite published history.
- Never print secrets, tokens, or full credential values.
- Keep changes scoped; do not reformat unrelated files.
- Run `pnpm test` and inspect `pnpm pack` output before claiming completion.
- Report what changed, what was verified, and what remains uncertain.

## 17. Definition of done

- [ ] `pnpm test` passes.
- [ ] New behavior has a regression test.
- [ ] `pnpm build` succeeds.
- [ ] `pnpm pack` contains the expected files and no tests.
- [ ] Installation in a fresh DSH profile succeeds.
- [ ] `--dump-config` shows `# == dsh-proof` and `id: proof`.
- [ ] README, CHANGELOG, and config table are updated.
- [ ] No secrets, credentials, or local profile data are committed.
- [ ] Version, tag, release, and npm package agree.
- [ ] CI is green on Ubuntu and Windows.

## 18. Useful commands

| Command | Purpose |
|---|---|
| `pnpm install` | Install dev dependencies. |
| `pnpm test` | Build and run the 16 tests. |
| `pnpm build` | Compile `src/` to `lib/`. |
| `pnpm pack` | Create the installable tarball. |
| `npm publish --dry-run` | Inspect what npm would publish. |
| `gh run list` | Check CI and release runs. |
| `npm trust list dsh-proof` | List trusted publishers (OTP required). |
| `npm view dsh-proof version` | Check the published version. |

## 19. References

- [README](README.md)
- [Changelog](CHANGELOG.md)
- [Publishing guide](PUBLISHING.md)
- [Security policy](SECURITY.md)
- [DSH plugin packaging guide](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md)
- [Conventional Commits](https://www.conventionalcommits.org/)
