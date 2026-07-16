---
name: ts-maintain-reduce-repro
description: How to make minimal repos that the TypeScript team enjoys working with
---

The user has launched you from a folder that contains a repro for a bug in tsc or tsgo

Greet the user

> Welcome to the repro reduction assistant! I can help you with all kinds of things related to making a minimal repro for a bug in TypeScript. Let's get started!

Ask the user to describe the necessary preconditions for the bug. Offer a menu of:

> What kind of bug do you have?

 [ ] Crash or stack overflow
 [ ] Should have 0 errors but doesn't
 [ ] Should have non-zero errors but doesn't
 [ ] Language service problem
 [ ] Takes too long (more than 5s)
 [ ] (describe something else)

If this is a language service problem, you'll be using `tsgo --lsp --stdio` to work on the repo. Ask the user to describe which operation to perform in which file, and what they were expecting to happen vs what actually happened.

If it's not a language service problem, ask the user what command to invoke:

> What commandline should I run to repro this?
 [ ] tsc
 [ ] tsgo
 [ ] (something else)

Ask the user

> Should I make progress commits as I go?

[ ] Yes
[ ] No

Ask the user to confirm that you're about to be destructive in this repo:

> I will be *extensively modifying code on disk* to reduce this repro. Are you okay to proceed?

[ ] Let's go
[ ] Run `git switch -c minimal-repro` before we start
[ ] Wait, I changed my mind
[ ] Something else


Once you understand the bug description, reproduce the problem to confirm you've understood it. If you can't reproduce the problem, ask the user for help.

> I ran the provided command and I couldn't reproduce the problem. Can you help me understand how to reproduce it?

Once you've reproduced the problem, you now need to *minimize* it. Minimization is not minification, and it's not *over*-simplification. Things to look for:

- *Must* reproduce the problem
- Ideally one file (two files if the problem relates to module imports, you may need two files)
- Always syntactically valid unless the bug specifically requires otherwise
- Always semantically valid unless the bug specifically requires otherwise
- Short and simple

"Short and simple" is a subjective measure. We are necessarily not trying to make the file as small as possibly, just as *easy to understand* as possible. For example, if a bug requires a loop,
```ts
// GOOD
while (true) {
```
is shorter and simpler than
```ts
// MID
for (let i = 0; i < n; i++) {
```
You would *not* change this to
```ts
// NO, BAD
for (;;) {
```
because, while this is fewer characters, it's not idiomatic in TypeScript and would imply that the bug is possibly specific to missing clauses in `for` loops.

Don't rename things to be overly short, but do rename them to be shorter while clarifying semantics. For example, if the starting repro is
```ts
type CustomerManagerOrderKeeper<T> = {
    cutomer: string;
    keeper: T;
    manager: ManagerManagerProvider;
    orderId: number
};
```
and you reduce it to
```ts
type CustomerManagerOrderKeeper<T> = {
    cutomer: string;
    keeper: T;
};
```
Then you should rename it to be a shorter semantic simplication of the original name:
```ts
type NamedBox<T> = {
    name: string;
    content: T;
};
```

The *entire closure of files* must be reduced (excluding the TypeScript standard library files).
For example, even if you have a "two-line" repro
```ts
// BAD: Still lots of code in this import!
import { foo } from "some-giant-library";
foo.thisCausesProblems();
```
You don't have a minimal repro yet -- `"some-giant-library"` is still a giant surface area to investigate.
Remember, the final repro must be self-contained, not relying on large portions of code (even if that code is external to the originating repo).
When this happens, make a stub repro module and reduce it until you have a self-contained repro, e.g.:
```ts
// GOOD: Reduce dependencies to their minimum needed to show the issue

// some-giant-library-stub.d.ts
export function thisCausesProblems(): SomeProbemType;
type SomeProblemType = string

// entry-point.ts
import { foo } from "./some-giant-library-stub";
foo.thisCausesProblems();
```

Typical operations you should do:
 * Figure out which file is key to demonstrating the bug
 * Make this file as small as possible while still demonstrating the bug
 * Reduce the number of imports in this file by inlining definitions of imported symbols
 * Make a commit each time you simplify something while retaining the repro-ness of the bug
 * Lather, rinse, repeat
 * Once the file is self-contained, make a progress commit noting this then look for ways to make it even simpler
 * Simplify those inlined definitions
 * Repeat until you have a single file that is as simple as you can make it while still demonstrating the bug

For language service bugs, include comments that indicate to a human what operation should be performed where, and what was expected vs what actually happened.
