---
name: tempo9-sampling
description: Choose sampling parameters, thinking mode and speculative-decoding depth for a local model on Tempo9, with the measured reasons behind each preset. Use when output repeats or degenerates, when a thinking model never closes its think, when a yes/no answer parses wrong, or when deciding whether speculation is worth turning on.
---

# Sampling, thinking and speculation

## Overview

Every value here was chosen because a default produced a visible failure.
Copy the presets; read the reason before changing one.

## Presets by mode

| mode | temperature | top_p | top_k | notes |
|---|---|---|---|---|
| chat, no thinking | 0.7 | 0.8 | 20 | |
| chat, thinking on | 0.6 | **0.95** | 20 | wider nucleus, not narrower |
| extraction / summarisation | 0 (greedy) | 1.0 | 1 | plus repetition penalty, below |
| binary verdict | 0 (greedy) | 1.0 | 1 | `max_tokens` 16, not 8 |

### Thinking wants a *wider* nucleus

The instinct is to tighten sampling for reasoning. It is wrong here:
near-greedy thinking degenerates into endless repetition. Thinking runs at a
**lower temperature and a wider `top_p`** (0.6 / 0.95).

### Greedy on long inputs needs a repetition penalty

Pure greedy decoding over a long transcript collapses into the same sentence
repeating. A light penalty is enough:

```
temperature = 0, top_k = 1, repetition_penalty = 1.1
```

Do not raise it much further — a heavy penalty distorts summaries of text
that legitimately repeats terms.

## `max_tokens` is a correctness parameter, not just a cost cap

Three failures, all from a budget that was too small:

- **A verdict cut off.** A 16-token budget was reduced to 8 for a yes/no
  question. The model sometimes narrates before it concludes
  ("in the image, a man … yes"), so every judgment truncated to nothing and
  parsed as *no*. Keep 16.
- **A think that never closes.** With thinking on and a small budget, the
  reasoning consumes the whole allowance, `</think>` never closes, no answer
  is emitted, and an empty string is parsed as a negative verdict. Give
  thinking a large budget (2048+), and treat an empty answer as a
  **re-ask**, not as an answer:

  ```python
  if thinking and not answer.strip():
      return ask(question, thinking=False)   # greedy, small budget, decisive
  ```

- **A budget added to the wrong number.** When thinking is on, add the think
  allowance to the answer budget (`max_tokens + 3584`), rather than sharing
  one budget between them.

## Parse a verdict by its first decisive character

For a yes/no answer, scan for the first character that decides, wherever it
appears — do not test for containment:

```python
hit = False
for ch in answer:
    if ch in "是":            hit = True;  break
    if ch in "否不没":         hit = False; break
if answer.lower().startswith("yes"):
    hit = True
```

Containment gets `不是` ("is not") wrong, because it contains `是`. First
decisive character resolves `是的`, `否`, `不是`, `没有` and a narrated
verdict correctly.

## Speculative decoding: depth matters more than on/off

`--speculation-k` (server) / `speculationK` (in-process). Interleaved
measurements, three passes each, on one machine:

| k | throughput | acceptance | mean accepted length |
|---|---|---|---|
| 0 | 96.5 tok/s | — | — |
| 1 | 89.7 | 0.647 | 1.65 |
| 2 | 87.4 | 0.643 | 2.29 |
| 3 | **99.4** | 0.719 | 3.09 |

`k=1` and `k=2` **lose**: a draft plus a verify costs about what one plain
step costs, and 1.65 accepted tokens does not repay it. At `k=3` the chain is
long enough that one verify covers three tokens.

**The right depth is a property of the model, not a constant:**

| model shape | best k | effect |
|---|---|---|
| large MoE | 3 | +3% |
| mid dense | 1 | +16% (k=3 was −1.5%) |
| model with no draft head | 0 | speculation does nothing at all |

The heavier per-token model pays more for a broken chain and less for a
shallow verify; the MoE amortises the verify over deep chains.

Two rules that follow:

1. **Measure per model.** Interleave the arms — throughput drifts across a
   session, so consecutive blocks are not comparable.
2. **Do not advertise what is not running.** If the model's graph carries no
   draft head, `k=3` and `k=0` measure the same. A UI badge reading
   "speculation on, k=3" while nothing speculates is a lie in the product.
   Detect the capability, or default to 0 for models you have not checked.

Verified speculation does not change the output: on the same greedy prompt,
`k=3` and `k=0` produce **byte-identical** text. If yours does not, that is a
bug to report, not a tuning knob.

## Checklist

- [ ] thinking uses temperature 0.6 / top_p 0.95, never near-greedy
- [ ] greedy long-input work carries `repetition_penalty ≈ 1.1`
- [ ] thinking budget is added to, not shared with, the answer budget
- [ ] empty answer after thinking triggers a greedy re-ask
- [ ] verdicts parsed by first decisive character
- [ ] speculation depth measured for this model, interleaved
- [ ] `k = 0` for any model whose draft head you have not confirmed
