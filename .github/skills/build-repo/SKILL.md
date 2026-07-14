---
name: build-repo
description: Get an unseen (possibly old or abandoned) TypeScript/JavaScript GitHub repo building in a disposable sandbox. Detects the era-appropriate toolchain, runs the documented setup, then drives Setup/Compile/Test/Runtime to green with a progress-based convergence loop. Use when you need to bring up a repo you've never seen and honestly report what it took. Scope is TS/JS (Node + package manager + tsc) only.
---

# Build Repo Skill

Bring an unseen TS/JS repository up from nothing: clone it, install the right
era-appropriate toolchain, run its documented setup, and drive each phase to green —
honestly. The repo may be old, abandoned, large, or under-documented. Your job is to make
it build *and* tell the truth about what that took.

**Environment assumption:** you run in a disposable sandbox (a throwaway machine or
container). You may install system packages (`apt`), switch Node versions, and start
backing services freely — but you MUST record every tool and version you install (see
[extended-plan-schema.md](./extended-plan-schema.md)) so the result is reproducible.

**Scope:** TS/JS only — Node, a JS package manager (npm/pnpm/yarn), and `tsc`. Other
languages are out of scope.

## Output

When you finish, produce:

1. **The working checkout** — the repo cloned and built in the sandbox.
2. **An extended EvalPlan JSON** — the four-phase plan (the exact commands that worked),
   extended with a **tools manifest** and a **convergence log**. Schema and example are in
   [extended-plan-schema.md](./extended-plan-schema.md). Write it to your session/output
   folder; do **not** commit it into the target repo.

## Phases

The build uses four canonical phases:

| Phase | Meaning | Default verification |
|-------|---------|----------------------|
| **Setup** | Install dependencies + run the project's build pipeline (turbo/webpack/etc.) | exit code 0 |
| **Compile** | Typecheck only, via the project's own `tsc` (its typecheck script, or `npx tsc --noEmit`) | exit code 0, parse error count |
| **Test** | The project's real test command | exit code 0, parse failure count |
| **Runtime** | Verify it actually runs — module loads, CLI runs, server responds | per-step (`output-contains`, `http-status`, etc.) |

Each phase ends in one of four **terminal states**:

- **PASS** — ran green.
- **FAIL** — ran and failed (after the convergence loop plateaued).
- **ABSENT** — proven not to exist (e.g. the repo genuinely has no tests). Only valid with
  evidence — never as a shortcut for "I couldn't find it."
- **BLOCKED-ON-EXTERNAL** — needs real credentials or external services the sandbox can't
  provide. Reported honestly; not a failure.

**Definition of done:** the loop has plateaued (or hit the cap) on every phase, every
terminal state is justified, and **at least one of Compile/Test/Runtime is PASS**. A repo
where nothing can be compiled, tested, or run is a failed build, not a pass.

## Workflow

### 1. Clone and orient

1. Build whatever commit you're given — an upstream process selects the ref to check
   out, so don't pick a branch or commit yourself. Note the **commit date of the ref
   you're building** (`git show -s --format=%cI HEAD`) — you'll need it for toolchain
   selection, and it may be far older than the repo's latest commit.
2. Read documentation **prose first**: `README.md`, then `CONTRIBUTING.md`, then
   `docs/`. Well-maintained repos put setup in `CONTRIBUTING.md`; smaller repos use
   `README.md`. Extract the documented setup, build, test, and run commands verbatim.
   You are partly here to **audit whether these documented instructions actually work**,
   so start from them rather than from CI.
3. Only if prose is missing or its steps fail, fall back to **executable** sources of
   truth: `.github/workflows/*.yml`, `Dockerfile`, `.devcontainer/`, `docker-compose.yml`,
   `Makefile`/`justfile`, and the `scripts` section of every `package.json`.
4. Find every `package.json` to populate `packages` (monorepos have many).

### 2. Select and install the toolchain

Pick the **Node version** from these sources, in priority order:

1. `engines.node` in the root `package.json`
2. `.nvmrc`
3. `.node-version`
4. `.tool-versions` (asdf) / Volta config in `package.json`
5. `Dockerfile` / `.devcontainer/` base image
6. CI workflow `node-version`
7. **If none of the above:** default to the Node **LTS that was current at the
   checked-out ref's commit date** — *not* the latest. Abandoned repos build with
   era-appropriate
   tools. (This is where naïvely defaulting to "latest" breaks old repos.)

Obtain it via a version manager (`nvm`/`fnm`). For the **package manager**, respect the
`packageManager` field / lockfile and install the matching version (corepack). Record
every install in the tools manifest.

### 3. Install dependencies — frozen first

Reproduce the original dependency graph: `npm ci` / `pnpm i --frozen-lockfile` /
`yarn install --frozen-lockfile`. If the frozen install **fails**, you may fall back to a
non-frozen install (`npm install`) or regenerate the lockfile — but only as a **recorded
fidelity compromise** in the manifest, because you've now changed which code you're
building. A frozen-install failure is itself a meaningful signal that the repo may be
broken as published (yanked packages, integrity mismatches).

Infer undocumented system tools and services as you go:

- **Reactively:** parse install/build errors for missing-tool signatures and install them —
  `node-gyp`→python/make/gcc (`build-essential`), `sharp`→libvips, `canvas`→cairo/pango,
  `puppeteer`/`playwright`→browser binaries, `node-sass`→python, etc.
- **For Runtime:** stand up backing services you can infer — `docker-compose` services, a
  local Postgres/Redis/sqlite — and synthesize a dev environment from `.env.example` or
  sensible defaults. **Never fabricate real third-party credentials.** If a phase truly
  needs a real secret or external service, mark it **BLOCKED-ON-EXTERNAL**.

### 4. Drive each phase to green — the convergence loop

For each phase, run its command(s), then:

1. Compute the **outcome signature**: per phase, `{ passed, errorCount, messageSet }`
   (pass/fail + failure count + the set of distinct error messages). Append it to the
   convergence log.
2. If the phase passed → move on.
3. If it failed → propose a **fix** (a change to setup/compile/test/runtime commands, a
   tool to install, a service to start, a config to set) and re-run.
4. **Termination:** stop the loop for this run when the current signature matches **any**
   signature already seen this run (a plateau or an A→B→A cycle), or when you reach the
   **hard cap of 10 total fix attempts** across all phases. Otherwise keep looping — a
   changed signature (status flip, count move, or a different error set) is progress, even
   if it's still failing.

The point of progress-based termination (vs. a fixed retry count) is to keep unblocking
error-after-error until you genuinely can't make the result move — so you don't quit early
and accept a broken setup as if it were merely unlucky.

A downstream failure may be caused upstream: a Compile or Test failure can legitimately
send you back to fix Setup (e.g. a missing dependency). Fixes are not confined to the
phase that failed.

### 5. Faithfulness rules (a run is INVALID if violated)

Never game a phase to green. These are inherited from discovery and are non-negotiable:

1. **No exit-code masking.** Never append `|| true`, `|| :`, `|| exit 0`, `|| echo ...`,
   `; true`, or `; exit 0`. Commands must report their real exit code. A non-green result
   is a valid outcome to report.
2. **No toolchain-pin swaps.** Compile MUST use the project's own configured TypeScript
   (its typecheck script, or `npx tsc`/`tsc` resolving the project's installed version).
   Never substitute a different pinned version (e.g. `typescript@5.9.2`) to force success.
3. **No narrowing.** Keep Test and Runtime representative of the whole project. Don't
   shrink the test command to one trivial suite, and don't reduce Runtime to a trivial
   check like `--version`.

Record dependency drift and any non-frozen install as a **fidelity compromise** in the
manifest rather than hiding it.

### 6. Emit the result

Write the extended EvalPlan JSON (working commands per phase + tools manifest +
convergence log + per-phase terminal states) to your output folder, and summarize for the
user: what built, what's ABSENT or BLOCKED-ON-EXTERNAL and why, every tool you had to
install, any fidelity compromises, and — if the build ultimately failed — your diagnosis
of whether the repo is broken as published or simply needs something the sandbox lacks.
