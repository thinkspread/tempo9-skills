---
name: tempo9-tool-calling
description: Get reliable structured decisions out of a local model on Tempo9 using tool/function calling — constrain the choice with an enum the caller computes, stream a visible thought before the call, parse partial argument JSON, and recover when no tool call arrives. Use when a small local model must pick an action, fill a form, or emit machine-readable output.
---

# Tool calling on Tempo9

## Overview

A small local model asked to "reply in JSON" will eventually reply in prose.
Tool calling is the reliable shape, and this skill is about the parts that
are not obvious: who owns the option list, how to keep the UI alive while the
call is being written, and what to do when the model answers in words anyway.

> **If you want data rather than a decision, read `tempo9-structured-output`
> first.** Guided decoding constrains the sampler to your schema, so an
> unparseable reply is impossible. Tool calling is the right shape when the
> model should choose *whether* and *which* to call; for one object of a known
> shape, a grammar is the stronger guarantee and removes the whole "no tool
> call arrived" recovery path below.

## The rule that makes small models reliable

**The caller computes the legal options; the model only picks one.**

Do not ask the model what is possible. Ask it to choose from an `enum` that
your own code derived from real state. This turns an open-ended generation
problem into a classification problem, which is what a 4B–35B local model is
actually good at.

```python
def choose_tool(options):
    return [{"type": "function", "function": {
        "name": "choose",
        "description": "Pick exactly one of the given options.",
        "parameters": {"type": "object", "properties": {
            "option": {"type": "string", "enum": options},
            "thought": {"type": "string",
                        "description": "Under 20 words: why."}},
        "required": ["option"]}}}]
```

The companion rule, from a multi-agent game where six characters act
overnight without supervision:

> The model emits an action; deterministic code applies it. The model never
> narrates state.

A model asked to both role-play *and* remember who owes whom a favour drifts
within an hour. Keep the world in your own data structure; let the model
produce one decision per call.

## `tool_choice`: only `auto`

Tempo9 runs tools in `auto` mode: the model decides whether to call. Over
HTTP, any other `tool_choice` — `"required"`, `"none"`, `"any"` or a named
tool — is refused with a 400 `invalid_request_error`, because nothing in the
server would enforce it, and answering 200 while ignoring it would be worse.

When a turn *must* produce an action — a game tick, a form field — do not
make it a tool call. Constrain the reply to the action's JSON schema with
guided decoding (see `tempo9-structured-output`), so a reply that is not the
action cannot be produced. Keep tools for turns where an answer in prose is
legitimate, with the no-call fallback below.

## Streaming a thought that arrives before the call

Tool-call arguments only arrive complete at the end of the stream, so a UI
that waits for them shows a spinner for the whole request. Two techniques,
both used in production:

**1. Ask for prose first, then the call.** Append to the prompt:

> First write your thinking in one or two sentences (not a tool call), then
> call `choose` once.

The prose streams immediately and the call lands after it.

**2. Pull the field out of partial argument JSON.** While `arguments` is
still an incomplete JSON string, a regex can already read a string field:

```python
import re
m = re.search(r'"thought"\s*:\s*"((?:[^"\\]|\\.)*)', args_so_far)
if m:
    live_update(name, m.group(1))
```

This is deliberately a regex and not a JSON parse — the text is not valid
JSON yet, and will not be until the call completes.

## Accumulating a streamed tool call

Deltas carry the function name once and the arguments in pieces:

```python
content, args, calls = "", "", []
for delta in deltas:
    if delta.get("content"):
        content += delta["content"]
    for tc in delta.get("tool_calls") or []:
        f = tc.get("function") or {}
        if f.get("name"):
            calls.append({"function": {"name": f["name"], "arguments": ""}})
        if f.get("arguments"):
            args += f["arguments"]
            if calls:
                calls[-1]["function"]["arguments"] = args
```

`arguments` is a **string** containing JSON, not an object. Parse it, and
accept both shapes defensively:

```python
a = calls[0]["function"]["arguments"]
if isinstance(a, str):
    a = json.loads(a)
```

## When no tool call arrives

Small models sometimes answer the question in words instead of calling. That
is recoverable if the options are known strings — scan the prose for one,
**longest name first** so that `"attack north"` is not matched by `"attack"`:

```python
for o in sorted(options, key=len, reverse=True):
    if o in content:
        return o
```

Only after both the call and the scan fail should you use a fixed default —
and choose defaults that are *inert* (pass, decline, end turn), never ones
that alter state in the caller's favour.

## Note: a large tool schema is a large prompt

Agent frameworks send tool schemas on every turn, and dozens of them render
to a very large prompt. This is normal and the server handles it, but two
consequences are worth designing for:

- The rendered prompt, not your message text, is what fills the context
  window. Budget accordingly (see `tempo9-agent-loop`).
- Tool schemas are a **stable prefix**. Put them, and the system prompt,
  ahead of anything that changes per turn, so prefix caching can reuse them.

## Checklist

- [ ] options come from an `enum` your code computed, not from the model
- [ ] deterministic code owns state; the model owns one decision
- [ ] `arguments` parsed as a JSON **string**, both shapes handled
- [ ] no-call path scans prose longest-name-first before defaulting
- [ ] defaults are inert
- [ ] tool schemas sit in the stable part of the prompt
