# AGENTS.md

DeepSeek Harness is an all-plugin Cordis agent harness. Read [docs/architecture.md](docs/architecture.md) before changing `packages/`; follow [docs/AGENTS.md](docs/AGENTS.md) for documentation.

## Pre-stable APIs and released Session data

Public APIs are pre-stable; update every consumer. Follow [version/status](docs/session-format-status.md) and [type acknowledgements](docs/cookbook/reviewing-persistence-type-changes.md). [Adjacent migration](.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md) may add a version-named successor but never move, overwrite, or delete committed generations; predecessors imply neither fallback nor downgrade support. SQLite uses monotonic `SCHEMA_VERSION`.

Acknowledge [declared persistence-type changes](docs/cookbook/reviewing-persistence-type-changes.md). Record each externally perceptible breaking change immediately in an [upgrade guide](.agents/skills/dsh-create-upgrade-guide/SKILL.md).

**Application launch.** Only `dsh` profiles launch supported Node apps; package bins, demos, and public SDK argv escapes are forbidden ([rule](docs/architecture.md#application-launch)).

## Repository layout

- `packages/<group>/<pkg>/`: `@deepseek-ai/dsh-<pkg>` workspaces. The group map is in [packages/README.md](packages/README.md); [experimental status](packages/experimental/README.md#status) is independent of optional delivery.
- `vendor/`: pinned Cordis source copies; sync procedure in [README](vendor/README.md). After a sync, rerun `pnpm run test && pnpm run build`.
- `native/`: node-addon-system source. `python/`: Python SDK/runtime. `docs/`: documentation ([AGENTS.md](docs/AGENTS.md)). `.agents/`: agent workflows and notes. `scripts/`: gates and generators. `website/`: VitePress projection. `benchmarks/`: performance gates.

## Commands

```sh
pnpm install            # pnpm workspaces, node ^22.19 || >=24
pnpm run test           # unit tests
pnpm run test:coverage  # CI coverage gate: per-file 100% on packages/*/*/src
pnpm run test:snapshot  # keyless recorded-session replay; filter: -t <name>
pnpm run typecheck
pnpm run lint
pnpm run build:artifacts  # develop with this: tsc types, tsdown bundles, Web
pnpm run build:check    # typecheck against the current build:artifacts output
pnpm run build          # build:artifacts + build:check
pnpm run doc-sync       # documentation gates
pnpm dsh --profile headless "task"  # one task from source (needs DEEPSEEK_API_KEY)
```

The full script list is in [package.json](package.json). Build modes are in [docs/development.md](docs/development.md#daily-commands); test lanes are in [docs/testing.md](docs/testing.md).

## Run relevant checks locally

- Before pushing, follow [dsh-pre-push-checks](.agents/skills/dsh-pre-push-checks/SKILL.md) and report only commands run. After `gh stack sync`, validate immediately; do not merge before checks pass.
- Match evidence to the surface: focused behavior tests, snapshots for model or user-visible output, `doc-sync` for docs, built smokes for published paths, real-API e2e for providers. `test:coverage`, not `test`, is the CI coverage gate ([why](docs/testing.md)).
- Never default to the full suite or repeat a passing check for commit or push. CI owns exhaustive coverage and the platform matrix; run everything locally only on explicit request, for CI diagnosis, or for an irreducibly repo-wide change.
- Before claiming TypeScript work complete, run `build:check` after the latest `build:artifacts`, or `build` ([layout](docs/development.md#typescript-project-layout)).
- Web browser automation and GIF recording: launch with `pnpm dsh web --patch apps/web/tests/pin-browse-picker.overlay.yml` unless testing native picker behavior ([overlay](apps/web/tests/pin-browse-picker.overlay.yml)).
- Host sandbox failures: if a required `gh`, `pnpm`, build, test, or generator command fails because the sandbox blocks credentials, network, IPC, watching, or nested `sandbox-exec`, retry unchanged with the narrowest host escalation. Require sandbox evidence; never bypass test failures or the product sandbox.

## Secrets / .env

Real-API tests and demos read `DEEPSEEK_API_KEY`, optional `DEEPSEEK_BASE_URL`, and root `.env`. Never commit credentials; CI e2e skips without a key, and [testing.md](docs/testing.md) owns key policy. Windows packaging and signing: [required reading](apps/desktop/windows-signing/README.md). cordis.yml allows `!!js` (never `!js`) under plugin `config` and entry `disabled`; other metadata stays literal, so conditional composition uses overlays ([primer](docs/cordis-primer.md#loader-configuration)).

## Conventions

- Packages use `@deepseek-ai/dsh-<name>`. Vendor is [rescoped](docs/rescope.md) and `private: true`. Harness packages declare `@deepseek-ai/cordis` as a peer and dev dependency. Workspace deps use `workspace:*` (DSH) or `workspace:~` (vendor, native) ([rules](.agents/notes/implemented/process/2026-08-10-npm-release-sequences.md)).
- ESM everywhere (`"type": "module"`). Use package names across packages and `.ts` in local relative imports. Config subprocesses run built `lib/` under plain Node ([testing policy](docs/testing.md#test-subprocess-launch-modes)). The `dsh` source launch uses tsx's ESM-only hook (`node --import tsx/esm`), so modules it reaches must not use CJS-only exports ([contract](.agents/notes/implemented/architecture/2026-07-29-dsh-source-launch-tsx-esm.md)). Raw/Web `cordis.yml` bare plugins must appear in their resolver manifest's `dependencies`; `verify-cordis-config` enforces it.
- **Registrations are effects.** Every contribution goes through `ctx.effect()` or `ctx.on()`; a registry's `register()` returns the disposer.
- **Typed events use declaration merging** and merge-extensible maps. Event JSDoc needs `@mode` and payload `@param`; public service methods document parameters and non-void returns. `SessionEventMap` members are required-on-read unless the envelope carries `ignorable: true`. Bump `SESSION_FORMAT_VERSION` only when existing discriminators cannot prevent unsafe interpretation; same-version changes need safe reader behavior ([review](.agents/notes/implemented/process/2026-10-08-session-reader-compatibility-review.md)).
- **Switch on discriminant tags.** Closed unions end in `assertNever`; merge-extensible unions fall through a documented default. **Waterfall listeners MUST call `next()`**, or the chain short-circuits ([semantics](docs/cordis-primer.md#cordis-waterfall-semantics)).
- **Model-visible ⟺ logged.** Anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event.
- **Plugins, not loop changes.** New behavior goes on documented extension points; changing `agent-loop` requires updating docs/architecture.md.
- **A capability seam has Service Definition, Service Provider, and Consumer roles.** It is complete, never one role ([glossary](docs/glossary.md#capability-seam)).
- **Prefer maintained dependencies** over hand-rolling when they genuinely delete owned code and tests ([policy](.agents/notes/implemented/process/2026-07-26-dependencies-over-hand-rolling.md)).
- **Explicit over implicit at package boundaries.** Defaulting is an explicit `resolve(request): Spec` step, never a hidden `?? default` inside `run()` (`dsh-shell` is the template).
- **No hardcoded tunables in plugins.** Deployment-varying choices are validated `Config` fields; a `DEFAULT_*` constant is not configurability. Protocol constants, external specs, and security invariants stay fixed.
- **Misconfiguration fails loud** at load, or at the earliest resolvable point. Never silently skip a missing referent.
- **Opaque cross-boundary ids are branded** (`Branded<B>` from `dsh-brand`), never bare `string`.
- **Trust TypeScript at typed same-process boundaries.** Validate only at parser, config, queued, model or tool JSON, durable or file, worker, process, and wire boundaries.
- **No new assertions to `unknown`** (`as unknown` or `<unknown>`). Preserve or reduce the legacy baseline ([rule](.agents/notes/implemented/process/2026-09-19-no-unknown-casts.md)).
- **Source plane and artifact plane are never mixed.** Static gates and tests resolve workspace imports through tsconfig `paths` to `src`; gates that consume built `lib/` declare that dependency ([layout](docs/development.md#typescript-project-layout)).
- **Keep compiler faces explicit.** Packages with both Host and Client programs expose face-specific leaf configs and a solution-only root ([layout](docs/development.md#typescript-project-layout)).
- **Error handling** uses typed, structured failures and localized, actionable reports. Stop dependent work and visibly report untrusted or unknown state. Follow [dsh-error-handling](.agents/skills/dsh-error-handling/SKILL.md).
- **An empty `catch` names the error** and why; keep its `try` to one statement.
- **Keep comments local.** Do not restate code or explain distant behavior without local need ([rationale](.agents/skills/dsh-prose-standard/SKILL.md)). Ban the words `prove` and `nance` ([rule](.agents/notes/implemented/process/2026-08-26-ban-ambiguous-origin-label.md)).
- **Prefer symmetry for parallel values.** Unexplained asymmetry usually signals a missed extraction.
- **Tests describe behavior, not correctness.** Change obsolete behavior with its tests and explain why in the PR.
- **Agent Notes are for durable decision rationale only**; mechanical and local UI edits are exempt ([scope](.agents/notes/README.md#when-to-write-one)). Archived notes are frozen ([archive policy](.agents/notes/README.md#archiving-and-deletion)).
- **Official plugin artwork** follows [dsh-official-plugin-artwork](.agents/skills/dsh-official-plugin-artwork/SKILL.md); obtain designer approval before landing new glyphs.
- **Client UI copy is locale-owned.** Route product text through typed dictionaries and `t`; `verify-client-ui-i18n` rejects hardcoded copy ([decision](.agents/notes/implemented/architecture/2026-08-23-locale-owned-client-ui-copy.md)).
- **Testing** follows [docs/testing.md](docs/testing.md). Every non-trivial model- or user-visible change updates a keyless recorded-session snapshot ([ownership](snapshots/AGENTS.md)). Plan unit, e2e, and snapshot coverage for capability seams, lifecycle paths, and transcript output. Fix fixtures, not normalizers.
- **Design each tool's UI presentation up front.** Host presenters stay pure; Web cards derive from raw events and persisted result metadata ([cookbook](docs/cookbook/adding-a-tool.md)).
- **Both SDKs project the loop.** Agent-loop, session-lifecycle, and `SessionEventMap` changes update the TypeScript and Python SDK expected outputs in the same PR ([surfaces](docs/testing.md#when-a-snapshot-test-is-required)).
- **Choose PR history deliberately.** Split independent changes. Rewrites use `--force-with-lease` and never raw `--force` ([rationale](.agents/notes/implemented/process/2026-08-02-native-github-stacks-and-optional-rebases.md)).
- **Labels:** one PR `kind/*`, all material `area/*`, and a native Issue Type ([taxonomy](.agents/notes/implemented/process/2026-08-08-unified-github-label-taxonomy.md)).
- TODO markers are `FIXME`, `TODO`, or `XXX` by urgency ([semantics](docs/development.md)). Files end with exactly one trailing newline; `git diff --cached --check` gates it.

## Defensive patterns

Read [docs/defensive-patterns/README.md](docs/defensive-patterns/README.md) before changing a package surface, remote service, client component or state, Conversation or Session behavior, or lifecycle, concurrency, subprocess, or teardown code.

## Type safety and documentation

Everything compiles under `strict: true` with `noImplicitAny`; each remaining `any` explains why narrowing is infeasible. Every module and export has concise JSDoc for its non-obvious contract, and function-like exports include `@param` and `@returns`.

Comments and docs state complete contracts and context, in direct, concrete terms, without metaphors or reasoning transcripts. Write `response fields`, `JSON validation`, or `ESM exports` rather than `response shape`, `validation boundary`, or `module shape`. Keep `contract` for preconditions, postconditions, invariants, and compatibility promises ([dsh-prose-standard](.agents/skills/dsh-prose-standard/SKILL.md)).

Docs accompany every code change: update affected READMEs and JSDoc together. Bilingual work follows [docs/AGENTS.md](docs/AGENTS.md); only explicit user invocation may run `dsh-translate-docs`.

## Editing these instructions

`CLAUDE.md` symlinks `AGENTS.md` at root and `packages/`; edit the real file. Keep each rule self-contained, link high-level docs, and condense when clarity survives. Raise a `verify-doc-budgets` ceiling only when the required content needs the space.
