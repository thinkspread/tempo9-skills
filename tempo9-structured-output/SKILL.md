---
name: tempo9-structured-output
description: Constrain a Tempo9 model to emit JSON that matches your schema, using guided decoding — over HTTP with response_format, or in-process with SamplingConfig. Covers why property order changes what survives a truncated reply, how to bound fields so a small model stops rambling, when anyOf beats one flat object, and the first-use compile cost. Use when a reply must be machine-readable.
---

# Structured output (guided decoding)

## Overview

Guided decoding constrains the sampler to a grammar, so the reply *cannot* be
invalid JSON — a stronger guarantee than asking nicely, and stronger than
tool calling, which still lets the model answer in prose.

Two forms of the same feature:

```jsonc
// any valid JSON
{"response_format": {"type": "json_object"}}

// constrained to your schema
{"response_format": {"type": "json_schema",
                     "json_schema": {"schema": { ... }}}}
```

In-process it is the same two fields on the sampling config:

```swift
var cfg = SamplingConfig()
cfg.responseFormat = "json_schema"
cfg.responseSchema = schemaText     // the schema, as JSON text
```

**`json_schema` without a `schema` is refused, not downgraded.** The server
returns `invalid_request_error` rather than quietly serving free-form JSON —
a caller who asked for its schema to be enforced must not be told "success"
by something that ignored it.

## Property order is load-bearing

The sampler makes the model write properties **in the order the schema
declares them**. So:

> Put the decision fields first and free-text reasoning last. A reply cut off
> by `max_tokens` then loses the commentary, not the move.

And the trap that enforces this the hard way:

> `JSONSerialization` sorts keys. A sorted schema once put `reasoning` before
> `targets`, and every reply came back unparseable.

Build the schema text with the property order you intend — string
concatenation if necessary — and do not round-trip it through a serialiser
that sorts. If you must serialise, sort only the *inside* of each property,
never the property list.

```swift
// properties in DECLARED order; each value serialised independently
let properties = props.map { "\(js($0.0)):\(js($0.1))" }.joined(separator: ",")
return "{\"type\":\"object\",\"properties\":{\(properties)},"
     + "\"required\":\(js(fieldNames)),\"additionalProperties\":false}"
```

## Bound every field

A grammar stops invalid JSON; it does not stop a 4B model writing three
paragraphs into a string. Bounds do:

| field | bound |
|---|---|
| string | `maxLength` |
| array | `maxItems`, plus `maxLength` on the items |
| number / integer | `minimum`, `maximum` |
| a choice | `enum` — never a free string |

`"additionalProperties": false` and a full `required` list keep the model
from inventing fields instead of filling yours.

Any field naming a person, place or option should be an `enum` of the legal
values, computed by your code — the same rule as `tempo9-tool-calling`, and
for the same reason: an illegal value should be unrepresentable rather than
forbidden in prose.

## Several move types: `anyOf`, not one flat object

When a turn can be one of several kinds of action, a single object with every
possible field is the wrong shape — the model fills fields that do not apply.

> Five tools is not one flat object. Use `anyOf` branches, with the people
> fields as name enums.

Give each branch a `const` discriminator so your parser can tell them apart
without guessing.

## Warm each schema before you need it

**A schema is compiled on first use, and that costs seconds.** Left alone, the
cost lands on whichever request happens to use it first — in a game, on
whoever moves first; in an app, on the user.

Send one tiny throwaway request per schema at startup:

```swift
/// One tiny request per move so every schema is compiled before a game;
/// first-use cost is seconds and would otherwise land on whoever moves first.
public func warm(_ moves: [Move]) async {
    for m in moves {
        _ = try? await tinyAgent.call("warm-up: fill anything.", move: m)
    }
    resetNumbers()          // and do not let the warm-up into your statistics
}
```

Reset your counters afterwards, or the warm-up shows up in the latency you
report.

## Streaming a JSON reply to a human

A constrained reply streams as JSON, and braces are not what a reader wants.
Show the string value currently being written:

```swift
// free text as is; a JSON object reduced to the string value being written
// -- the reader wants the sentence, not the braces.
```

Match the known prose keys (`reasoning`, `statement`, `thought`, …) and fall
back to "the last string value still open". This is the guided-decoding
analogue of streaming a thought before a tool call.

## Guided decoding or tool calling?

| use | when |
|---|---|
| **guided decoding** | you want data — one object, known shape, every time |
| **tool calling** | the model should choose *whether* and *which* to call, or the framework on the other side speaks tools |

For a pure extraction or a per-turn decision with a fixed shape, guided
decoding is the stronger guarantee: tool calling can still come back as
prose, and `tempo9-tool-calling` has a whole section on recovering from that.
With a grammar, that failure mode does not exist.

## Checklist

- [ ] schema built with properties in intended order, not through a sorting serialiser
- [ ] decision fields first, free text last
- [ ] every string bounded, every choice an `enum`, `additionalProperties: false`
- [ ] several action types expressed as `anyOf` with a `const` discriminator
- [ ] every schema warmed at startup, and the warm-up excluded from the numbers
- [ ] `json_schema` requests always carry a `schema` — the server refuses them otherwise
