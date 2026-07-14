# TypeScript Maintainer Skills

This repository contains [Agent Skills](https://docs.github.com/en/copilot/customizing-copilot/copilot-skills) that help with TypeScript maintenance tasks.

## Reduce Repro

```
/ts-maintain-reduce-repro
```

An interactive Copilot skill that helps you take a bug reproduction for `tsc` or `tsgo` and minimize it into a clean, easy-to-understand repro that the TypeScript team can work with quickly.

### What It Does

The skill walks you through an interactive workflow to **reduce a bug repro** down to the smallest, simplest form that still demonstrates the problem. This means:

- Stripping away unnecessary files, imports, and code
- Inlining imported symbols so the repro is self-contained
- Renaming overly verbose identifiers to shorter, semantically clear names
- Keeping the code syntactically and semantically valid
- Targeting a single file when possible (two files if the bug involves module imports)
- Making progress commits along the way so you can track each simplification step

### Supported Bug Types

- Crash or stack overflow
- Unexpected or missing errors
- Language service problems (e.g. completions, go-to-definition, hover), supported only for TS7+
- Performance issues

### Prerequisites

1. **A folder with a bug repro** — You should already have a project or set of files that reproduces a bug in `tsc` or `tsgo`.
2. **`tsc` or `tsgo`** — Whichever compiler the bug applies to should be available on your PATH.

### How to Use

1. **Install** the skill
 * For VS Code, run `/skills add microsoft/TypeScript-Maintainer-Skills` in Copilot Chat.
 * For Copilot CLI, run `/skills add microsoft/TypeScript-Maintainer-Skills`.
 * For Claude Code, clone this repo and copy the skill to your personal skills directory:
   ```
   git clone https://github.com/microsoft/TypeScript-Maintainer-Skills /tmp/ts-skills
   cp -r /tmp/ts-skills/.github/skills/ts-maintain-reduce-repro ~/.claude/skills/
   ```

2. **Invoke** the skill, usually with 
   ```
   /ts-maintain-reduce-repro
   ```

3. **Answer** the prompts. There are 3-6 questions depending on the bug report.
4. **Let it work.** This typically takes at least 2 minutes and might be up to an hour for very complex repros.
5. **Get your minimal repro!** When it's done, you'll have a clean, minimal reproduction that's ready to attach to a TypeScript issue.

## Build Repo

```
/build-repo
```

A Copilot skill that gets an unseen (possibly old or abandoned) TypeScript/JavaScript GitHub repo building in a disposable sandbox. It detects the era-appropriate toolchain, runs the documented setup, then drives Setup/Compile/Test/Runtime to green with a progress-based convergence loop.

### What It Does

- Clones and orients in the repo, reading documentation prose first
- Selects and installs the correct Node version (era-appropriate for old repos)
- Installs dependencies with frozen lockfiles when possible
- Drives each phase (Setup → Compile → Test → Runtime) to green via a convergence loop
- Honestly reports what worked, what failed, and what it took to get there

### Use Cases

- Bringing up an unfamiliar repo for the first time
- Auditing whether a repo's documented setup instructions actually work
- Building old/abandoned repos with era-appropriate tooling

### Output

The skill produces the working checkout plus an extended EvalPlan JSON with:
- The exact commands that worked for each phase
- A tools manifest (every tool/version installed)
- A convergence log showing the fix attempts
- Terminal states per phase (PASS / FAIL / ABSENT / BLOCKED-ON-EXTERNAL)
