# Extended EvalPlan schema (build-repo output)

The skill emits the repo's standard `EvalPlan` JSON shape, **extended** with two fields
the build process needs: a `toolsManifest` and a `convergence` log. Each phase's steps are
the exact commands that ended the run, and each phase carries a terminal `state`.

The base fields match a standard EvalPlan shape, so the artifact stays mechanically
convertible to a harness plan. The extra fields are additive — a strict harness parser
ignores them.

## Shape

```jsonc
{
  // ── base EvalPlan fields ──────────────────────────────────────────────
  "url": "https://github.com/owner/repo",
  "branch": "main",
  "commit_id": "<HEAD sha at build time>",
  "projectType": "cli | webapp | api | library | other",
  "description": "one-line description",
  "nodeVersion": "18",
  "packageManager": "npm | pnpm | yarn",
  "packages": ["package.json", "packages/core/package.json"],

  "setup":   [{ "name": "...", "command": "...", "verificationMethod": "exit-code", "expected": "0" }],
  "compile": [{ "name": "...", "command": "..." }],
  "test":    [{ "name": "...", "command": "..." }],
  "runtime": [{ "name": "...", "command": "...", "verificationMethod": "output-contains", "expected": "..." }],

  // ── build-repo extensions ─────────────────────────────────────────────

  // Terminal state of each phase: PASS | FAIL | ABSENT | BLOCKED-ON-EXTERNAL.
  // ABSENT and BLOCKED-ON-EXTERNAL MUST carry a reason.
  "phaseStates": {
    "setup":   { "state": "PASS" },
    "compile": { "state": "PASS" },
    "test":    { "state": "ABSENT", "reason": "no test script and no test files in the repo" },
    "runtime": { "state": "BLOCKED-ON-EXTERNAL", "reason": "server requires a real STRIPE_SECRET_KEY" }
  },

  // Every toolchain / system package installed, so the build is reproducible.
  // `how` is the acquisition method; `why` is the trigger.
  "toolsManifest": [
    { "name": "node", "version": "18.20.4", "how": "nvm", "why": "engines.node \">=18\"" },
    { "name": "pnpm", "version": "8.15.4", "how": "corepack", "why": "packageManager field" },
    { "name": "build-essential", "version": "12.9", "how": "apt", "why": "node-gyp build of better-sqlite3" },
    { "name": "libvips-dev", "version": "8.12.1", "how": "apt", "why": "sharp native build" }
  ],

  // Recorded deviations from building the repo exactly as published.
  "fidelityCompromises": [
    { "what": "regenerated package-lock.json", "why": "npm ci failed: yanked left-pad@1.3.1 integrity mismatch" }
  ],

  // One entry per attempt, in order. The `signature` is what the loop compares
  // to detect a plateau/cycle; the loop stops when a signature repeats or after
  // 10 total attempts.
  "convergence": [
    {
      "attempt": 1,
      "phase": "setup",
      "signature": { "passed": false, "errorCount": 1, "messages": ["gyp ERR! find Python"] },
      "fix": "apt-get install -y python3 make g++",
      "changed": true
    },
    {
      "attempt": 2,
      "phase": "setup",
      "signature": { "passed": true, "errorCount": 0, "messages": [] },
      "fix": null,
      "changed": true
    },
    {
      "attempt": 3,
      "phase": "compile",
      "signature": { "passed": false, "errorCount": 12, "messages": ["TS2307: Cannot find module 'foo'"] },
      "fix": "add missing @types/foo via the project's own install",
      "changed": true
    },
    {
      "attempt": 4,
      "phase": "compile",
      "signature": { "passed": false, "errorCount": 12, "messages": ["TS2307: Cannot find module 'foo'"] },
      "fix": "retried with cleared tsc cache",
      "changed": false
    }
    // attempt 4 produced a signature identical to attempt 3 → plateau → stop compile.
  ]
}
```

## Field notes

- **`signature`** is the comparison key. Two attempts are the "same outcome" when, per
  phase, `passed`, `errorCount`, and the *set* of `messages` all match. Normalize messages
  (strip absolute paths, timestamps, line/col noise) before comparing so cosmetic churn
  isn't mistaken for progress.
- **`changed: false`** marks a plateau (the signature equals the previous attempt's). A
  signature that equals any *earlier* (non-adjacent) attempt marks a cycle — also a stop.
- **`fix: null`** when an attempt records a green result with no further action.
- **Hard cap:** at most 10 entries in `convergence` across all phases. Reaching it stops
  the run regardless of state.
- **Faithfulness:** no step command may contain exit-code masking (`|| true`, `; true`,
  …), a swapped pinned TypeScript version, or a narrowed test/runtime. Such a plan is
  invalid and must not be emitted.
