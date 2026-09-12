---
name: multi-persona-review-panel
description: >
  Convene a small panel of independent reviewer lenses on a diff before opening a PR. Each
  lens reads the change against the repository as it is, not as the PR description says it
  is, and returns findings with file and line plus one verdict. Use before opening a PR for
  anything beyond a trivial edit. Complements a correctness-focused code-review pass: this
  skill covers the lenses a pure bug hunt misses (architecture, security, test and quality
  coverage, performance).
---

# Multi-persona review panel: run the panel yourself, before the PR

The premise: a defect the reviewer would have found is a defect the author can find first.
Instead of pushing a change and waiting for a human reviewer to notice the architectural smell
or the missing test, convene a small panel of independent reviewer lenses yourself and let each
one read the change against the real repository. The cheap catches move to authoring time; the
human reviewer's time goes to judgment calls.

**When:** before opening any PR for a non-trivial change. The same procedure works on a written
design or a goal contract before any code exists; the document is then the artifact under review.
**Not worth a panel:** typos, renames, pure questions.

## Procedure

### 1. Scope the change

Scope is the **whole diff the reviewer will actually see**: every commit on the branch against
its base (for example `git diff main...HEAD`), not just your most recent commit. A file added
earlier in the same branch is in the reviewer's scope, so it is in yours. A panel run on the last
commit alone can return four clean verdicts on a branch whose second commit added an untested
script.

### 2. Pick the lenses the change touches, typically three or four

A minimal, generically useful lens set:

| If the diff touches | Bring in |
|---|---|
| architecture, module boundaries, a shared data model | the **Architecture** lens |
| auth, secrets, external input, trust boundaries | the **Security** lens |
| tests, edge cases, acceptance criteria | the **Quality** lens |
| a hot path, a resource cap, a cost-sensitive operation | the **Performance** lens |

Scale the panel to the blast radius of the change. A one-file bugfix might only need one lens;
a new service boundary might need all four. If the diff touches something none of the four
cover, add a lens for it rather than stretching one that does not fit.

### 3. Spawn each lens as an independent parallel reviewer

Each reviewer gets:

- (a) the lens to adopt: a short description of what that lens cares about and what a bad
  example looks like;
- (b) the actual diff under review;
- (c) the real source files the diff touches, with the instruction to check every claim against
  those files and to treat the PR description as a claim, not as evidence;
- (d) a fixed return contract: **concrete findings** (issue, file and line, and why it matters;
  no impressions) and **one verdict**, which is **SIGN** (no blocking issues),
  **SIGN-WITH-CHANGE** (named, specific changes required), or **BLOCK** (a named blocking
  defect);
- (e) the standing instruction to be rigorous and independent: neither wave the change through
  nor manufacture objections. If the change is sound, sign it and give the reason.

A diff shows what was added; it cannot show what was left out. So each lens also checks four
absences before it votes:

1. **Rollback.** For each forward step in the diff (a migration, a flag, a deploy change), name
   the step that undoes it. If no such step exists, say so.
2. **Coverage.** For each new code path, name the test that exercises it. Code that rewrites
   files on disk with no test around it is the riskiest shape here.
3. **One owner.** For each list or set the diff introduces (allowed values, tenants, routes),
   name the place it is derived from. A second copy maintained by hand is a defect even when
   both copies currently agree.
4. **The bad day.** For each input the diff handles, name what happens when it is empty,
   malformed, duplicated, or arrives twice at once, and name the test for the case the
   contract calls out (the exempt tenant, the legacy record).

A lens may defer any of the four with a stated reason. It may not skip them; a SIGN with an
unexamined absence behind it is a guess.

Independence is what makes the panel worth running. A single reviewer wearing every lens in
sequence anchors on the first thing it noticed and reads the rest of the diff through it. Four
separate reviewers do not share that anchor, which is why each lens runs as its own isolated
reviewer and not as one pass through a checklist.

### 4. Synthesize and fold

Deduplicate near-identical findings raised by more than one lens; two lenses landing on the same
line independently is the strongest signal the panel produces, not noise. Rank by severity:
BLOCK, then SIGN-WITH-CHANGE, then notes. Fold the required changes into the diff **before**
pushing. Record which lenses were convened and their verdicts somewhere durable (the PR
description is enough) so the review is on the record before a human opens it.

## Anti-patterns

- Running the panel after the PR is already up and treating it as a formality instead of a gate.
- One agent adopting every lens in a single pass, which removes the independence that does the
  catching.
- Scoping the panel to your latest commit instead of the whole diff against the base branch.
- Reading a SIGN from every lens as proof of completeness. It is proof that the lenses you picked
  found nothing; the panel only checks for what its lenses are looking for.
- Letting an absence pass because no lens brought it up. The four checks in step 3 are asked, not assumed.
