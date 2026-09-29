# tempo9-skills

Claude Code skills for building applications on [Tempo9](https://github.com/thinkspread/tempo9)
— the engine and the Swift SDK.

These are not API reference pages. Each one collects the decisions that a
shipping app had to get right, with the measurement or the failure that
forced the decision. Where a number appears, it was measured; where a rule
appears, something broke without it.

## Skills

### [tempo9-http-client](tempo9-http-client/)
Talk to a Tempo9 server over HTTP from any language: starting the server, the
API dialects it serves, SSE streaming, and correct token accounting. Covers
the mistake that has corrupted more measurements than any other here — a
stream chunk is a flush, not a token.

### [tempo9-structured-output](tempo9-structured-output/)
Guided decoding: constrain the sampler to your JSON schema so an unparseable
reply is impossible — over HTTP with `response_format`, or in-process on the
sampling config. Why property order decides what survives a truncated reply,
how to bound fields so a small model stops rambling, when `anyOf` beats one
flat object, and the first-use compile cost you should warm away.

### [tempo9-tool-calling](tempo9-tool-calling/)
Reliable structured output from a small local model: let the caller compute
the option `enum`, keep world state in deterministic code, stream a visible
thought before the call arrives, parse partial argument JSON, and recover
when the model answers in prose instead.

### [tempo9-agent-loop](tempo9-agent-loop/)
Multi-turn and multi-agent loops: what keeps a prefix cacheable, the measured
TTFT baseline (0.20 s steady on a 4B, 0.43 s on a 35B MoE), context budgets,
the engine going cold between questions, and the failure shapes that return
HTTP 200 with nothing in them.

### [tempo9-sampling](tempo9-sampling/)
Sampling presets per mode, why thinking needs a *wider* nucleus, why
`max_tokens` is a correctness parameter, how to parse a verdict, and why
speculative-decoding depth is a property of the model rather than a constant.

### [tempo9-vision-audio](tempo9-vision-audio/)
Multimodal message construction, why part order is load-bearing, and the
two-stage pattern that makes a small VLM reliable: caption with the image,
then judge the caption **without** it.

### [tempo9-small-model-reliability](tempo9-small-model-reliability/)
The division of labour that makes a 4B–35B model dependable in an app: code
counts, the model judges. Validators for the failure shapes small models
actually have, why remembered facts must be framed as past (measured: 4/5
runs ruined vs 5/5 saved), why an illegal option must be removed rather than
forbidden, and how to grade the model's stated reasoning against ground truth
with a chance baseline.

### [tempo9-embed-swift](tempo9-embed-swift/)
Running the engine in-process in a Swift app: session lifetime (including the
actor re-entrancy bug that ships two engines), warm-up phases, pairing Metal
kernels with the engine build, and degrading honestly to a fallback.

## Where these came from

Three applications built on Tempo9:

- a live camera/screen assistant — vision, audio, in-process engine
- an offline audio library with question-answering — long transcripts,
  retrieval, a fallback engine, token accounting
- a multi-agent game — HTTP, tool calling, many concurrent characters

## Using them

Drop the directory somewhere Claude Code looks for skills, or read them
directly — each `SKILL.md` stands alone and ends with a checklist.

## Contributing

Pull requests are welcome. Every commit needs a `Signed-off-by` line — the
[Developer Certificate of Origin](DCO) — which `git commit -s` adds. See
[CONTRIBUTING.md](CONTRIBUTING.md).

## License

Copyright (c) 2026 Jiejing Zhang. Licensed under the Apache License 2.0; see
[LICENSE](LICENSE).
