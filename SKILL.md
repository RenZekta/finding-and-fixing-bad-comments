---
name: finding-and-fixing-bad-comments
description: Search a codebase for low-quality comments left behind by AI-assisted or multi-session development (chat-transcript narration, "fixed a bug" changelogs, restated code, stale/outdated claims) and rewrite them to describe current behavior. Use this whenever the user asks to "clean up comments", "find bad comments", "remove AI slop from comments", audit a codebase's comment quality, or after a long multi-turn coding session where many small comments may have accumulated across edits.
---

# Finding and fixing bad comments

Codebases that were built up over many AI-assisted edits (or many human PRs
without review) tend to accumulate comments that narrate the *process* of
writing the code instead of describing the code itself. This skill is a
repeatable method for finding those comments and rewriting them well.

## The core distinction

A good comment describes something true about the code **right now**, for a
reader who has no access to the conversation, ticket, or PR that produced it.

A bad comment describes the **history** of the code, or the process that
created it: what it used to do, that a bug existed, who asked for a change,
which numbered task this was. That information has a natural home — commit
messages and PR descriptions — and it stops being true or useful the moment
the surrounding code changes again.

If you have to ask "does this comment reference something outside the file
itself (a conversation, a ticket, a prior version)?" and the answer is yes,
it's very likely worth rewriting.

## Step 1: search with grep, don't rely on skimming

Skimming misses most of these because they're scattered across a large
codebase in small quantities. Use grep (or the local equivalent) with the
keyword groups below. Search comments in every language present in the repo
(`//`, `#`, `/* */`, `<!-- -->`, etc.) — don't assume it's all one language.

### Group 1: explicit transcript/task numbering
The highest-confidence signal. These almost never contain information a
future reader can use.

```
Task [0-9]
Item [0-9]
Fix [0-9]
Feature [0-9]
Bug fix
BUG FIX
```

### Group 2: referring to the conversation or the person
```
per user
per the user
as requested
as discussed
the user (reported|noticed|asked|wants)
this round
per your (request|ask)
```

### Group 3: narrating history instead of stating current behavior
```
previously
used to be
used to (crash|fail|show|throw|hardcode|return)
no longer needed
was broken
now (correctly|fixed|works)
this used to
originally
legacy
we (fixed|changed|removed|added)
I (fixed|changed|removed|added)
```

### Group 4: changelog-in-a-comment
Look for comments containing phrases like "before/after", "old behavior",
side-by-side descriptions of two versions of the same logic, or a comment
that reads like a diff description ("X now does Y instead of Z").

Run each keyword as its own grep pass (combining too many into one regex
buries the signal). Record file:line for every hit before editing anything —
triaging first prevents partial, inconsistent cleanup.

## Step 2: triage each hit

For every match, decide which of these it is:

1. **Pure narration, zero remaining content** — the comment says nothing
   about the code that isn't already obvious from reading it, once you strip
   the "we changed X" framing. **Delete it entirely.** Don't leave a
   watered-down version just to have "something" there.
2. **Narration wrapping a real, useful fact** — buried inside the history
   ("this used to crash because...") is a real invariant or constraint
   worth keeping ("X must happen before Y because Z"). **Rewrite it** to
   state that fact directly, present tense, with the history stripped out.
3. **Stale/outdated claim** — the comment describes behavior, a code path,
   or a feature that has since been removed or changed elsewhere, and now
   the comment is simply *wrong*, not just badly framed. This is the most
   dangerous category because it actively misleads. Verify against the
   current code before rewriting or deleting — check whether the thing it
   describes still exists at all.
4. **Fine as-is** — a numbered list describing sequential *algorithm steps*
   (not task-tracking) or a comment that already describes an invariant
   is not a violation just because it contains a number or the word "now".
   Don't over-apply this skill to things that were never a problem.

## Step 3: rewrite, don't just delete the flagged phrase

The lazy fix is deleting the word "previously" and leaving the rest of the
sentence — that usually leaves an awkward half-sentence or still-narrated
tense. Rewrite the whole comment as if it had always been there, describing
what's true now:

```
BAD (narrates history, references people/process):
// Bug fix: previously this crashed when the user turned off mmap while
// mlock was on. Now we combine them into one flag as requested.

GOOD (states the invariant directly):
// mlock requires the file to already be memory-mapped; llama.cpp asserts
// on this internally, so --load-mode enforces it as one combined option.
```

```
BAD (changelog disguised as a comment):
// Previously this used a flat 10% estimate for all models. Now it uses
// the model's real attention geometry from the GGUF header instead,
// which is far more accurate.

GOOD (just states what it does and why):
// Uses the model's real attention geometry from the GGUF header (not a
// flat percentage) because per-model KV size varies enough that a flat
// estimate is off by 2-4x on common models.
```

```
BAD (pure narration, no remaining content — delete this one entirely):
// Item 6: Overrides tab — positioned after Monitoring, per the plan.
<button onClick={...}>Overrides</button>
```

## Step 4: don't stop at the comment — check the code it describes

A stale comment is often a symptom of stale logic nearby: if a comment says
"X option is disabled when Y" and Y no longer exists in the code, check
whether the disabling logic itself is also dead code that should be removed,
not just the sentence describing it. Fixing the comment without checking the
code it references only hides the smell.

## Step 5: don't invent scope

Only touch comments you actually found violating the above — don't take this
as license to rewrite comments that are already fine, restructure code, or
"improve" prose style beyond removing the narration problem. If a sweep turns
up comments outside the area the user is actively working in, it's worth
asking whether they want the full-codebase sweep or just the files touched in
this session, since a large diff of pure comment changes can be noisy to
review.

## General rules for writing comments well (apply going forward, not just when cleaning up)

- A comment should tell a future reader something the code itself doesn't
  already say: a non-obvious invariant, a constraint imposed by an external
  system, or the reason behind a non-obvious choice. If the code is
  self-explanatory, don't add a comment just to restate it in English.
- Never reference the conversation that produced the change, a task/ticket
  number, or "the user" — write as if the comment has always been there.
- When explaining a bug fix, describe the invariant or failure mode itself,
  not the fact that a bug existed and was fixed.
- When editing existing code, replace the old comment rather than leaving it
  next to a new one saying something similar — a function should have one
  coherent explanation, not a changelog of every pass someone made over it.
- If asked to add this guidance to a project's own contributor docs (e.g. an
  AGENTS.md, CONTRIBUTING.md, or CLAUDE.md), keep it concise: the
  distinction in "The core distinction" above, one or two good/bad examples,
  and the "replace, don't stack" rule are usually enough — the full keyword
  list here is meant for an active grep sweep, not for embedding wholesale
  into every project's contributor guide.
