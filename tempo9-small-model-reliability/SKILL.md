---
name: tempo9-small-model-reliability
description: Make a small local model dependable inside an application — split the work so code counts and the model judges, validate every generated string before it is used, frame remembered information correctly, and grade the model's stated reasoning against ground truth with a chance baseline. Use when a local model fabricates facts, echoes its input, drifts over long sessions, or reasons correctly but acts wrongly.
---

# Making a small local model reliable

## Overview

A 4B–35B model running locally is strong at judgement and weak at
bookkeeping. Most application-level failures come from asking it to do the
second. This skill is the division of labour that works, the validators that
catch what still gets through, and how to measure whether any of it helped.

Every rule here comes with the failure that produced it.

## Code counts. The model judges.

**Do the counting and the obvious inference in your own code**, and hand the
model a stat sheet plus the genuinely ambiguous question.

The failure that forces this: in a hidden-role game, a seat read "X declined
to spend a healing card on Y" out of a raw log, concluded X was hostile, and
attacked its own ally twice. The record was right there; the model simply
counted and inferred badly under load.

```python
# Build facts the code can vouch for, one line per subject.
"X: looks hostile - attacked the leader 2x, declined to help an ally 1x"
"Y: unclear - has not acted yet"
```

Then the prompt asks only what remains: *given these facts, who do you
trust?* The model is now doing the part it is good at.

## Two-layer memory: facts by code, interpretation by model

For anything remembered across sessions, keep two layers with different trust
levels:

| layer | produced by | trusted |
|---|---|---|
| facts | your code, from the log | always |
| interpretation | the model, one subject at a time, facts revealed | never as input |

> A small model handed a raw transcript invents habits that cannot exist.
> Handed a stat sheet, it mostly reads it back — which is what we want.

**The model's sentence must never re-enter a prompt.** It fabricates — one
note claimed "he attacked me 5 times" against a sheet reading 0. Keep it as
displayed flavour, clearly marked as the character muttering, and feed only
the code-generated facts into the next prompt.

## Validate every generated string before you use it

Small models fail in recognisable shapes: loops, echoes of the input, and
blanks. A cheap validator catches all three:

```python
def sane(t):
    """Reject the small-model failure shapes: loops, stat echo, blanks."""
    t = t.strip()
    if not (6 <= len(t) <= 40):                  # blank, or a runaway
        return False
    if any(t.count(ch) > 3 for ch in "x,"):      # echoing the stat sheet
        return False
    grams = [t[i:i+4] for i in range(0, len(t) - 3)]
    return len(set(grams)) > len(grams) * 0.6    # repetition = repeated n-grams
```

The n-gram uniqueness test is the useful one: a looping answer has few
distinct 4-grams relative to its length, and this catches it without knowing
anything about the content.

Its companion, for "did it just say that again" **across** two strings, is
character-bigram overlap — one agent repeating its own last thought, or
restating the description the previous speaker just gave:

```python
def near_duplicate(a, b, threshold=0.55):
    def grams(s):
        t = [c for c in s if not c.isspace() and c not in "，。,.、:：;；!！?？'\"()（）"]
        return {"".join(t[i:i+2]) for i in range(max(1, len(t) - 1))}
    x, y = grams(a), grams(b)
    return len(x & y) / max(1, len(x | y)) > threshold
```

Both are cheap enough to run on every reply.

**Fail soft.** If the string does not pass, drop it. An optional embellishment
that is sometimes absent is a better product than one that is sometimes
wrong.

**Accept the answer from either channel.** Read a tool call if there is one,
otherwise take the first sentence of `content` — models drop out of tool mode
under pressure, and refusing a good answer for arriving in the wrong field is
a self-inflicted failure.

## Frame remembered information as past — this is measured

A note written as `X: spy, attacks the leader` reads to a model as *who X is
now*. If roles are re-dealt each session, that is a factual error injected by
your own memory system.

Measured, same model, same scenario:

| framing | outcome |
|---|---|
| `X: spy, ...` | the leader died in **4 of 5** runs |
| `last session X was the spy, ...` | the leader survived **5 of 5** |

Prefix every remembered item with when it happened, and say in the prompt how
much it is worth:

> Impressions from past sessions are for reference only and weigh far less
> than this session's record; identities are re-dealt, so the current state
> is authoritative.

## Remove illegal options where the list is built

The most important structural rule, and the one that is easy to get wrong by
trying to fix it in the prompt.

An ally seat was offered its own leader as a target, and took it — while the
same response said, correctly, *"I must protect the leader and must not
attack them."*

> The reasoning was right and the chosen option was wrong. That is a small
> model's failure shape, not a prompt problem.

No amount of instruction fixes this reliably. Remove the option from the list
you pass in — privately to that caller, so nothing about the hidden state
leaks — and the illegal move becomes unrepresentable.

This is the same principle as constraining choices with an `enum`
(`tempo9-tool-calling`), applied one level up: the enum's *contents* are a
correctness surface.

## Order the fields so a truncated reply still carries the decision

When the reply is schema-constrained, the model writes the properties in the
order the schema declares them — so **decision fields first, free text last**.
A reply cut off by `max_tokens` then loses the commentary rather than the
move. Full treatment, including the serialiser that silently sorts your keys
and breaks this, in `tempo9-structured-output`.

## Grade the stated reasoning against ground truth

Benchmarks measure the model. This measures **your application**. If your app
already records what the model said and later learns the truth, you can score
it for free.

Score along axes that have a defensible baseline:

| axis | how |
|---|---|
| self-consistency | did it state its own known attributes correctly? |
| inference accuracy | claims about hidden state vs the revealed truth |
| factual claims | claims about the record vs the record |
| say/do consistency | the stated intent vs the action taken next |

Two things make the numbers mean something:

**1. Publish the chance baseline next to the score.** In a five-seat game
with a known distribution, guessing scores 40% on one role and 20% on the
others. A 45% accuracy is barely above chance; without the baseline printed
beside it, it reads as competence.

**2. Bucket by phase.** Early-, mid- and late-session accuracy differ, and an
aggregate hides both the cold-start problem and late-session drift.

The say/do check is the cheapest and catches the most: pair each recorded
intent with the action that immediately followed and flag the mismatches.

```
intended to act against A -> acted against B
```

Keep the offending quotes, not just the counts. A rate tells you something
regressed; the quote tells you what.

## Fuzz the deterministic half

If code owns the rules, the rules can be fuzzed without a model at all.
Random legal play, many runs, asserting only that nothing raises:

> 40 games of random legal play, zero exceptions, ~28 turns each.

This separates "the model chose badly" from "the app crashed", which are
different bugs with different owners, and it runs in seconds.

## Checklist

- [ ] counting and obvious inference done in code, not in the prompt
- [ ] remembered facts generated by code; model interpretation never re-enters a prompt
- [ ] every generated string passes a validator; failures drop silently
- [ ] answers accepted from a tool call **or** plain content
- [ ] remembered items carry when they happened, and their stated weight
- [ ] illegal options removed at list construction, not forbidden in prose
- [ ] stated reasoning graded against truth, with the chance baseline printed
- [ ] scores bucketed by session phase, with offending quotes kept
- [ ] the deterministic half fuzzed with random legal input
