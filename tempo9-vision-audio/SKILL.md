---
name: tempo9-vision-audio
description: Send images and audio to a Tempo9 vision/audio model — how multimodal message parts are ordered, how a vision tower's embedding is placed, and the prompt patterns that make a small VLM answer reliably. Use when building a camera/screen assistant, when a VLM invents or misses events, or when a vision badge in the UI must tell the truth about what ran.
---

# Vision and audio on Tempo9

## Overview

Tempo9 runs the language model; a **vision or audio tower** turns pixels or
sound into an embedding that is woven into the token stream. On Apple
hardware the tower can run on the Neural Engine while the language model runs
on the GPU. This skill covers the message shape and the prompt patterns that
decide whether a small VLM is usable.

Server-side, point at a tower with `--tower`. In-process, hand the session an
image or audio placement (see `tempo9-embed-swift`).

## Message parts: order is load-bearing

The chat template emits **one placeholder token per part, in the order you
list them**, and the engine expands each placeholder into the run of
embeddings that part covers. List them in this order:

```jsonc
{"role": "user", "content": [
  {"type": "image"},                       // picture
  {"type": "audio"},                       // sound
  {"type": "text", "text": "your question"} // words
]}
```

Picture, then sound, then words — which is also the order these models were
trained to read them. Getting the order wrong does not error; it misaligns
which embedding fills which run.

For a text-only turn, keep `content` a plain string. Only use the parts array
when a non-text part is actually present.

## Send the transcript *with* the audio

When a platform speech recogniser is available, send both its transcript and
the raw audio:

> The recogniser is better at words than the model is; the model is the only
> one that hears tone, music and the room. Sending both keeps the words as
> accurate as they were and adds what the recogniser threw away.

## Ask a binary question, do not mine a caption

The most valuable pattern here, because it inverts the obvious design.

**Do not** generate a free-form caption and then look for the event in it.
A caption is a terrible detector: it invents events to satisfy the
instruction, and it forgets to mention real ones.

**Do** ask one binary question at temperature 0 with a small `max_tokens`,
and parse the verdict (see `tempo9-sampling` for the parsing rule).

## Judge the caption as text, with the image absent

A counter-intuitive result that was measured and then relied on:

> Every phrasing judged the caption correctly in text-only mode, while the
> same question with the image attached flipped clear positives to *no*. The
> picture poisons the verdict that its own caption gets right.

So the reliable two-stage shape is:

1. **image → text**: caption or describe, with the image attached
2. **text → verdict**: ask the yes/no question about the *caption*, with the
   image deliberately **absent**

## Cache the tower output when the picture has not changed

Tower work is separable from language-model work. Give each encoded frame a
content key and reuse the embedding while the content is unchanged — a
camera assistant answering follow-up questions about one frame should encode
once, not once per question.

Preheating the encode of the current frame is worth a small fixed cost, but
only while it is fresh: cap the age (≈1.5 s for a live camera) so the answer
is still about what the user was looking at.

## Do not let the UI claim a compute unit the model cannot reach

A recurring product bug worth naming: a badge that says the vision tower ran
on a particular accelerator, or that speculation was active, when the loaded
model has no path to it. Detect the capability and report what actually ran.

> The strip read "MTP on · k=3" while nothing was speculating — the same
> untruth as a vision badge naming a compute unit the model has no path to.

## Checklist

- [ ] parts ordered image → audio → text, one part per placeholder
- [ ] plain string content for text-only turns
- [ ] transcript sent alongside audio when a recogniser is available
- [ ] detection is a binary question, not a caption search
- [ ] the verdict stage is text-only; the image is not attached
- [ ] tower output cached by content key, with a freshness cap
- [ ] UI badges reflect what actually ran
