---
name: tempo9-http-client
description: Talk to a Tempo9 server over HTTP from any language — start the server, stream tokens over SSE, count tokens correctly, run requests concurrently, and fail loudly when the engine is gone. Use when building an app against a local Tempo9 endpoint, when a streaming client returns wrong token counts or empty text, or when deciding between the OpenAI, Anthropic and Responses dialects.
---

# Tempo9 over HTTP

## Overview

Tempo9 serves several API dialects on one port. Point any OpenAI-compatible
client at it and it works — the value of this skill is the handful of
behaviours that differ from a cloud endpoint and that silently corrupt
measurements or hide failures if you assume otherwise.

## Start a server

```bash
tempo9 --gguf /path/to/model.gguf --port 11435
```

| flag | meaning |
|---|---|
| `--gguf` | the model file, straight from Hugging Face, no conversion |
| `--graph` | precomputed graph file, when the model needs one |
| `--port` | listen port |
| `--max-length` | declared context in tokens |
| `--speculation-k` | speculative decoding depth (see `tempo9-sampling`) |
| `--tower` | vision/audio front end (see `tempo9-vision-audio`) |
| `--ollama` / `--list-ollama` | serve a model from a local Ollama store |

## Endpoints

| route | dialect |
|---|---|
| `POST /v1/chat/completions` | OpenAI Chat Completions |
| `POST /v1/messages` | Anthropic Messages |
| `POST /v1/responses` | OpenAI Responses |
| `POST /v1/messages/count_tokens` | Anthropic token counting |
| `GET /v1/models`, `GET /v1/models/{id}` | model listing |

Pick the dialect your client library already speaks. They are served by the
same engine over the same weights; there is no quality or speed difference.

## A stream chunk is a flush, not a token

**This is the single most expensive mistake to make against this server, and
it has been made more than once.** Counting SSE chunks and calling the result
`completion_tokens` produces numbers that are wrong by a factor that varies
with the *engine*, so it corrupts any comparison:

- with speculative decoding on, one chunk can carry 2–4 tokens
- other servers batch differently again — one measured at ~7 tokens per chunk

A rate computed from chunk counts once produced an apparent 6.5× win that
was, on real token counts, a dead heat.

Always ask the server for the count:

```python
body = {
    "model": "local",
    "messages": [...],
    "stream": True,
    "stream_options": {"include_usage": True},   # <- not optional
}
```

and read `usage.completion_tokens` off the final chunk. If you need a
character-rate for a quick sanity check, `len(text)/seconds` is at least
honest about what it measures.

## A minimal streaming client, stdlib only

```python
import json, urllib.request

def stream(base, body):
    """Yields (content_delta, usage_or_None). Raises on a server error event."""
    body = dict(body, stream=True, stream_options={"include_usage": True})
    rq = urllib.request.Request(
        base + "/chat/completions",
        data=json.dumps(body).encode(),
        headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(rq, timeout=300) as r:
        for line in r:
            line = line.decode("utf-8", "replace").strip()
            if not line.startswith("data:"):
                continue
            payload = line[5:].strip()
            if payload == "[DONE]":
                break
            try:
                d = json.loads(payload)
            except json.JSONDecodeError:
                continue
            # A refused request arrives as an error EVENT inside the stream,
            # before [DONE]. Reading only choices/usage turns that into a
            # silent zero-token run that reads like a slow model.
            if d.get("error"):
                raise RuntimeError("server error: %s" % json.dumps(d["error"])[:300])
            if d.get("usage"):
                yield None, d["usage"]
            for c in d.get("choices", []):
                delta = c.get("delta") or {}
                # Thinking models put their first tokens in reasoning_content.
                # A client that reads only `content` sees nothing for seconds
                # and reports a false TTFT.
                piece = delta.get("content") or delta.get("reasoning_content")
                if piece:
                    yield piece, None
```

Three things that block are worth naming, because each has produced a wrong
conclusion:

1. **`reasoning_content`** — thinking models emit reasoning before content.
   Time-to-first-token measured on `content` alone counts the whole think as
   latency.
2. **The error event** — it arrives inside the stream, not as an HTTP status.
3. **The role-only first chunk** — the server sends `{"role":"assistant"}`
   before generation starts, for client compatibility. Timing your first
   token off *any* chunk reports ~1 ms. Time the first **non-empty** delta.

## Fail loudly when the engine is gone

A dead server must not degrade into an app full of defaults. In a multi-agent
game, the model process was killed mid-run and the remaining rounds were
"played" against fallback choices — the run looked complete and meant nothing.

```python
class ModelUnavailable(RuntimeError):
    """The server is gone; the caller cannot continue honestly."""

try:
    ...
except Exception as e:
    raise ModelUnavailable(f"{who}: {str(e)[:80]}") from e
```

Reserve silent defaults for a **malformed but present** answer. An absent
server is a different event and deserves a different code path.

## Concurrency is real, so use it

The server does continuous batching: N in-flight requests are genuinely
batched, not queued. Fanning out with threads is the correct shape and it is
what the throughput advantage is for.

```python
import concurrent.futures
with concurrent.futures.ThreadPoolExecutor(max_workers=len(jobs)) as ex:
    results = list(ex.map(one_request, jobs))
```

Two cautions:

- **Do not run two engines on one machine.** Two processes each holding a
  multi-gigabyte model evict each other's pages continuously and both crawl.
  One server, many requests.
- **Greedy output is not bit-stable across different batch compositions.**
  At `temperature: 0`, two runs of the same prompt can differ when the set of
  requests batched alongside it differs. This is inherent to continuous
  batching. If you need byte-identical output for a test, send the request
  alone.

## Checklist before trusting a number

- [ ] `stream_options.include_usage` is set and the count comes from `usage`
- [ ] TTFT is measured on the first **non-empty** delta, including `reasoning_content`
- [ ] error events inside the stream raise instead of yielding empty text
- [ ] the run is not sharing the machine with a second engine
- [ ] for A/B, arms are **interleaved** — throughput drifts within a session
