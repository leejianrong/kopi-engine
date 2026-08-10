# kopi-engine

The shared agent engine behind [kopicode](https://github.com/leejianrong/kopicode)
(a terminal coding agent) and sotong (an always-on assistant, not yet started).

Nothing is built yet. This README records the decisions already taken and the
constraints that already bind, so the first commit of real code starts from them.

## What this owns, and what it doesn't

```
kopicode                     sotong
  terminal coding agent        always-on assistant
  interactive, supervised      unattended, triggered
              \               /
             kopi-engine (this repo)
             agent loop · tool dispatch · permission model
             session state · model routing · context management
                       |
                     satay
             durable execution · journal · replay · fork · compare
```

**Satay owns durability.** The journal, replay, forking, run comparison, retries
and idempotency keys are all its job. This repo does not reimplement any of them.

**The surfaces own UX.** Rendering, keybindings, notifications, and how a human is
asked for permission belong to kopicode and sotong. This repo decides *that*
permission is required and what the decision means, not how it is presented.

## Why two products on one engine

Something that acts while nobody is watching needs a stricter safety model than
something you are supervising. A coding agent runs in a repo with you present and
its worst case is a bad diff, caught in review, reversible with `git revert`. An
always-on assistant holds long-lived credentials to personal accounts, acts on
triggers, and its worst case has no equivalent rollback.

That difference is an architecture, not a config flag, which is why they are two
products. Making them two *engines* would be the same mistake sibei-flow's
[ADR-0012](https://github.com/leejianrong/sibei-flow/blob/main/docs/design/adr/0012-no-second-execution-engine.md)
exists to prevent, one layer up.

## Constraints that already bind

**Do not merge this into Satay.** Satay's
[ADR-0025](https://github.com/leejianrong/satay-runtime/blob/main/docs/adr/0025-positioning-agents-first.md)
decision 4 holds the no-agent-abstraction non-goal: Satay ships five durable
primitives and cookbook examples, and no loop framework, provider adapters or
graph DSL. This repo depends on Satay from outside. Merging it in would violate a
merged ADR.

**Satay is async-only.** The loop is async, and so is every provider and tool call.

**Satay's nondeterminism policy defaults to `strict`,** and an LLM loop has a
data-dependent call schedule: the model decides whether turn 3 calls `edit_file`
or `read_file`. That replays correctly *only if* every input to a branch is itself
a durable call. Concretely:

- `provider.complete` must be a `@satay.task`, not a bare call in the workflow body
- every tool dispatch must be a durable call
- the workflow body must not read a clock, an env var, or an RNG directly

This is workable but it constrains how the loop is written, so write it that way
from the first commit rather than retrofitting.

**Anything that starts a process or a container is `side_effect=True`,** with
Satay's runtime-derived idempotency key covering the interrupted-attempt case.

## The lesson worth not relearning

sibei-flow hand-rolled a transcript alongside its agent loop: a `list[str]` built
by appending as the loop ran, with tool output clipped at 1200 characters. It was
lossy by construction (the diagnostic output justifying a fix could be truncated
exactly where a reviewer looks) and free to drift from reality (add a tool call,
forget the append, and the record describes a run that did not happen). Fixing the
contract took a dedicated PR and had to land before a launch to avoid becoming a
migration. See
[ADR-0013](https://github.com/leejianrong/sibei-flow/blob/main/docs/design/adr/0013-transcript-tagged-union.md).

**The journal is the session record.** Do not build a parallel one. Anything a
user or a reviewer needs to see should be derived from the journal, which cannot
drift because replay depends on it.

## Open question

sibei-flow's repair loop is structurally the same thing as this: a bounded agent
loop with tool dispatch, a provider seam and a verification step. Its port onto
Satay is planned
([KAN-648](https://github.com/leejianrong/sibei-flow), ADR-0012 decision 4). Three
bespoke agent loops across the suite would be the mistake this repo exists to
avoid, so when that port happens the question is whether it consumes kopi-engine
rather than growing its own. Not decided, and not a reason to generalise early:
build for kopicode first, and let the second and third consumers show what is
actually shared.

## Licence

Apache-2.0, matching Satay, sibei-flow and kopicode.
