# multi-persona-review-panel

**Run four independent reviewers on your own diff before anyone else sees it.**

> A Claude Code skill, Apache-2.0. The procedure is in
> [`skills/multi-persona-review-panel/SKILL.md`](skills/multi-persona-review-panel/SKILL.md); a
> worked run on a sample diff is in [`examples/example-run.md`](examples/example-run.md). This is
> the pre-PR gate FortunaTerra runs on its own products, cut down to the mechanism. The rest of
> this page is why it exists and how to install it.

---

The usual failure of self-review is not laziness. One reviewer, human or agent, reading a diff
anchors on the first thing it notices and reads everything after through that. The architectural
smell gets a paragraph; the missing concurrency test three functions later gets nothing, because
the reviewer already has a story about the change.

This skill spawns a small set of reviewer lenses (architecture, security, quality, performance) as
separate agents against the real diff and the real files it touches, not against the PR
description. Each returns findings with file and line and one verdict: SIGN, SIGN-WITH-CHANGE, or
BLOCK. Each also checks four absences (the undo step, the test, the single owner of any new list,
the bad-day input), because a diff cannot show what was left out. You fold what comes back before
you push.

The worked example runs the panel on a rate-limiting middleware: two lenses find the same hot-path
cost from different directions, one blocks on a contract violation, one blocks on a customer tier
the code never checks, and the fold makes the PR mergeable.

## Install

Copy the skill into your project:

```bash
git clone https://github.com/FortunaTerra-Group/multi-persona-review-panel-skill
cp -r multi-persona-review-panel-skill/skills/multi-persona-review-panel .claude/skills/
```

Or load the repository as a plugin for one session:

```bash
claude --plugin-dir ./multi-persona-review-panel-skill
```

Then invoke `/multi-persona-review-panel` on a branch before opening the PR.

## Where it fits

The panel is the review step of a governed build loop. Upstream of it, a
[goal contract](https://github.com/FortunaTerra-Group/goal-contract) says what done means and who
checks; the lenses here are how the author checks their own diff before anyone else does. The
Architecture lens is where a [CHOP](https://github.com/FortunaTerra-Group/chop) violation (a second
owner for a piece of state, a transition that bypasses the contract) gets caught before it is
written into the history. The two vocabularies are deliberately different: a panel lens returns
BLOCK for a named defect in the diff; a goal contract's gate returns BLOCKED when the evidence
could not be produced at all.

## Provenance and license

This is the public form of the review panel FortunaTerra runs as its pre-PR gate. The internal
version routes to a larger cabinet of named personas and adds two further absence checks to
the four here; the mechanism is the part that transfers. First drafted 2026-09-05 under MIT by
the same author; relicensed to Apache-2.0 and assigned to FortunaTerra Technologies Inc.
2026-09-12. Copyright 2026 FortunaTerra Technologies Inc. Written and maintained by Vivek Iyer
([FortunaTerra-Group](https://github.com/FortunaTerra-Group)). Released under
[Apache-2.0](./LICENSE). The bar for changing the procedure is a defect that a panel run this way
let through; open an issue with the diff shape and the lens that should have caught it.
