---
name: tempo9-agent-loop
description: Build multi-turn and multi-agent loops on Tempo9 that stay fast and honest — keep the prefix cacheable, budget context, know the real TTFT numbers, and detect the failure modes that return HTTP 200 with nothing in it. Use when writing an agent, a chat session with history, or several characters/workers sharing one local server.
---

# Agent loops on Tempo9

## Overview

Agent traffic is a long stable prefix with a small moving tail. Tempo9 is
built for that shape, and the caching works — but only if the client does not
accidentally destroy the prefix, and only if the client checks what came back.

## The measured baseline

Ten consecutive turns in one session, agent-shaped: a 34 KB system prompt,
34 tool schemas, history growing each turn. Time to the **first non-empty
token of text**:

| model | cold, turn 1 | steady median, turns 2–10 |
|---|---|---|
| 4B-class | 6.75 s | **0.20 s** |
| 35B-class MoE | 9.57 s | **0.43 s** |

The widely repeated claim that local servers take 30–90 s per agent turn once
the prefix moves does **not** hold here. Budget for a slow first turn and a
sub-second steady state.

## Four blocks, and the first three are append-only

The shape that makes prefix reuse work, from a multi-agent host where every
character is an isolated agent on one engine:

```
[rules + roster]         identical for every agent -> shared across agents
[my private block]       fixed for this agent, for the whole session
[history, append-only]   public events and this agent's own notes
[tail: what to do now]   the only block whose shape changes
```

Nothing else ever reaches the model. Two properties fall out of it: agents
share the first block, so one prefill serves all of them; and only the tail
is rewritten per turn, so everything above it stays a cache hit.

**Assert the isolation, do not assume it.** Have the assembler return exactly
what was sent, and let a test check that no other agent's private block
appears in it. This runs with no model loaded — an offline table can deal a
game, build every prompt and pass the isolation checks, and only the first
actual request needs an engine.

## Keep the prefix cacheable

Prefix reuse is a prefix match. It survives only what does not change.

**Do**

- put the system prompt and tool schemas first, and byte-identical every turn
- append history; never rewrite earlier turns
- keep per-turn variation in the tail

**Do not**

- inject a timestamp, request id, or random nonce near the top of the prompt
- re-order tool schemas between turns
- reformat history (a changed separator invalidates everything after it)

A single changed character early in the prompt costs the whole cache.

## Cap the history, deliberately

Replaying every turn forever grows the prompt without bound and eventually
buys nothing. In a shipping app, two previous turns is the tuned value:

> Two is enough for "and the one next to it?" and small enough that the
> prefix stays cacheable.

Pick a number, write down why, and enforce it.

## Context budget

Declared context is set at server start (`--max-length`). Paged KV means a
longer declared context does **not** pre-allocate memory — but peak use still
grows with actual length, so the ceiling is a memory decision, not a free one.
Make it configurable rather than hard-coded.

A workable split, from an app that feeds long transcripts to a small model:

```
maxLength                 # engine context, e.g. 16384
direct-ask input budget   = maxLength * 3/4    # leaves room for history + answer
summarisation input       = maxLength * 7/8
```

For Chinese text, roughly one token per character is a serviceable estimate
when sizing these budgets.

## The engine goes cold between questions

Prefill slows down while nothing asks the engine anything. Measured on a
laptop-class machine:

| idle before the question | prefill |
|---|---|
| under 2 s | 480 ms |
| past 20 s | 1170 ms |

A keep-warm ping — one request, `max_tokens: 1` — holds this back. It is a
real trade, not a free win: a periodic ping costs GPU time on a machine that
may also be doing other work, so make it **opt-in**, and gate it:

```python
# Only when idle. A ping issued while the user is waiting queues ahead of
# the request they are waiting for.
if keep_warm_enabled and not thinking and not recording and engine_ready:
    ping()
```

Warming the model at load time is the higher-value half: it removes the
multi-second first question entirely.

## Check what came back — HTTP 200 is not success

Two failure shapes return a normal-looking response.

**1. An error event inside the stream.** Covered in `tempo9-http-client`:
read `d["error"]` on every chunk and raise.

**2. A request the engine interrupted.** The engine gives up on a request
under memory pressure, and whatever was generated is not trustworthy. Current
versions raise this as an error rather than returning the partial text — if
you are pinned to an older one, check the finish reason yourself and treat
`interrupted` as a failure, never as a short answer.

**3. An empty assistant turn.** Rarely, and so far only under concurrency, a
turn returns `finish_reason: "stop"` with `completion_tokens: 1` and no text.
Nothing reports an error. The user sees an agent that goes quiet for a few
turns and then recovers.

This one has a client-side amplifier worth removing regardless of cause: a
naive loop appends the empty answer back into the conversation

```python
msgs.append({"role": "assistant", "content": text})   # text == ""
```

and the next turn now contains a precedent that the assistant said nothing —
which makes another empty turn more likely. Observed runs of three to ten
consecutive empty turns from a single first occurrence.

Defend on both sides:

```python
ct = (usage or {}).get("completion_tokens")
if ct is not None and ct <= 1 and not text.strip():
    # Do NOT append this to the conversation.
    # Retry once, or surface it — never feed it back as a turn.
    ...
```

Treat an empty turn as an anomaly to log and retry, never as a normal turn.

## Several agents, one server

Fan out with threads; the server batches them for real (see
`tempo9-http-client`). Two shapes that work well:

- **Parallel independent decisions** — every character/worker decides at
  once, then deterministic code resolves the results in order.
- **A file-backed mailbox seat** — write the pending question to a file,
  block until an answer file appears. This lets a human on another device, or
  another agent, occupy one seat in an otherwise automated loop without any
  networking.

Keep resolution single-threaded and deterministic. Concurrency belongs in the
*asking*, never in the state updates.

## Checklist

- [ ] system prompt + tool schemas byte-identical every turn, and first
- [ ] no timestamp/nonce/uuid near the top of the prompt
- [ ] history capped at a written-down number
- [ ] context budget derived from `--max-length`, not hard-coded
- [ ] keep-warm opt-in and gated on idle
- [ ] empty assistant turns detected and never appended to history
- [ ] state updates deterministic and single-threaded
