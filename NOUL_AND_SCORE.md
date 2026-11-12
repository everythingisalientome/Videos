# Noul and Score

Reference for building the other two Jev question types in Decision One. Written to be read offline.

Part A describes what Jev does (from public docs, checked 2026-11-12). Part B is how to get the same result from `decide()`.

## The short version

Jev has three question types. All three are the same mechanism underneath: put probabilities on a fixed list of options.

| Jev type | Question it answers | Is really | Returns |
|---|---|---|---|
| `choice` | Which one of these? | A list of named options | The winner plus a probability per option |
| `noul` | Is this statement true? | A choice with two options: true, false | One number: probability of true |
| `score` | Where on this scale? | A choice over ordered levels 0, 1, 2... | The probability-weighted average level |

So noul and score need no new model call logic. Each is `decide()` plus one small calculation afterwards. Do not build three separate code paths.

---

## Part A: what Jev does

### Request (same envelope for every type)

```json
{
  "model": "jev-latest",
  "state": "Hi, my Stripe integration keeps failing for 3 days. Please refund my subscription.",
  "questions": {
    "asks_refund": {
      "type": "noul",
      "instructions": "The customer explicitly requests a refund"
    },
    "frustration": {
      "type": "score",
      "instructions": "How frustrated the customer appears",
      "criteria": ["Calm, just stating facts", "Frustrated but civil", "Very angry, strong language"]
    }
  }
}
```

- `state`: the text being judged.
- `questions`: a map. Each key is a name you choose and read the answer back under.
- Several questions of different types can share one state in one request.
- Type names are lowercase.

### Noul

| | |
|---|---|
| Meaning | Probability that a statement is true |
| `instructions` | The statement to evaluate, written as a statement ("The customer explicitly requests a refund") |
| `criteria` | Optional. A map with `true` and `false` keys describing what each means |
| Answer | `{"type": "noul", "noul": 0.91}` |
| Range | 0 to 1 |
| Confidence field | None. The noul value is itself the probability |

Optional criteria example:

```json
"is_urgent": {
  "type": "noul",
  "instructions": "Does this convey urgency?",
  "criteria": {"true": "Explicitly time-sensitive", "false": "No urgency expressed"}
}
```

### Score

| | |
|---|---|
| Meaning | Position on an ordered rubric |
| `instructions` | What is being rated |
| `criteria` | Required. A list of level descriptions, ordered lowest to highest |
| Levels | 2 to 10 |
| Level numbers | Start at 0. Three levels are 0, 1, 2 |
| Range of `score` | 0 to (number of levels minus 1) |

Answer:

```json
"frustration": {
  "type": "score",
  "score": 0.67,
  "confidence": 0.5,
  "legend": {"0": "Calm, just stating facts", "1": "Frustrated but civil", "2": "Very angry, strong language"},
  "probabilities": {"0": 0.33, "1": 0.67, "2": 0.0}
}
```

- `probabilities`: one per level, adding up to 1.
- `legend`: maps each level number back to its description.
- `score`: the probability-weighted average of the level numbers (worked below).

### Other limits reported for Jev

| Limit | Value | Source quality |
|---|---|---|
| Choice options | Up to 255 | Stated in several places |
| Score levels | 2 to 10 | Stated in several places |
| Tokens per request | 64k; state plus any one question 32k | Third-party guide only |

### What is not documented

- **How `confidence` is calculated.** It appears on choice and score answers, but no public page I could reach defines it. It is not simply the top probability (one example shows probabilities 0.85 / 0.15 with confidence 0.78). Do not try to match it; see Part B.
- **The score formula.** Not stated on the quick start page. The weighted average fits every example found, and a third-party API reference describes it as a probability-weighted rating.

---

## Part B: building them on `decide()`

Assume `decide(state, question, options)` already works and returns a probability per option that adds up to 1.

### Noul = decide with two options

1. Call `decide()` with two options: one for true, one for false.
2. Return the probability of the true option.

Worked example. Statement: "The customer explicitly requests a refund."

| Label | Meaning | Logprob from API | e^logprob |
|---|---|---|---|
| A | True | -0.163 | 0.850 |
| B | False | -2.526 | 0.080 |
| (other tokens) | | | 0.070 |

Total of our two options = 0.850 + 0.080 = 0.930.

noul = 0.850 ÷ 0.930 = **0.914**

That is the whole type. The renormalising step is the same one from STUDY_NOTES.md section 9.

```python
def noul(state, statement, true_means=None, false_means=None):
    options = {
        "true": true_means or "The statement is true",
        "false": false_means or "The statement is false",
    }
    probs = decide(state, statement, options)
    return probs["true"]
```

### Score = decide over levels, then a weighted average

1. Call `decide()` with one option per level. Use the digits 0, 1, 2... as labels.
2. Multiply each level number by its probability and add them up.

Worked example with three levels:

| Level | Description | Probability | Level × probability |
|---|---|---|---|
| 0 | Calm | 0.33 | 0 × 0.33 = 0.00 |
| 1 | Frustrated but civil | 0.67 | 1 × 0.67 = 0.67 |
| 2 | Very angry | 0.00 | 2 × 0.00 = 0.00 |
| | | **Score** | **0.67** |

Read it as: "between calm and frustrated, closer to frustrated."

A second example, to show why the average is useful. Probabilities 0.10, 0.30, 0.60:

0 × 0.10 + 1 × 0.30 + 2 × 0.60 = 0 + 0.30 + 1.20 = **1.50**

If we only reported the winner we would say "level 2". The score of 1.50 says "leaning very angry, but not certain", which is more honest.

```python
def score(state, question, levels):
    # levels: list of descriptions, lowest first
    options = {str(i): text for i, text in enumerate(levels)}
    probs = decide(state, question, options)
    value = sum(int(level) * p for level, p in probs.items())
    return {
        "score": value,
        "probabilities": probs,
        "legend": options,
    }
```

Digits 0 to 9 are convenient labels: there are exactly ten of them, which matches the 2 to 10 level limit, and it stays well under the top-20 logprobs cap. Still check that each digit is a single token in the Gemma tokenizer.

### Matching Jev's response shape

Return the same field names as Jev so the two can be swapped and compared with the same test code:

```json
{"type": "noul", "noul": 0.914}
```

```json
{
  "type": "score",
  "score": 0.67,
  "legend": {"0": "Calm", "1": "Frustrated but civil", "2": "Very angry"},
  "probabilities": {"0": 0.33, "1": 0.67, "2": 0.0}
}
```

### What to do about `confidence`

Leave it out for now. Its definition is unknown, and a made-up number under the same field name would look comparable to Jev's when it is not. The `probabilities` already carry the information. If a single number is needed later, pick one we can explain (for example the top probability) and give it a different field name.

---

## Things that will go wrong

| Problem | What it looks like | What to do |
|---|---|---|
| Yes-bias in noul | The model leans towards "true" regardless of the statement | Test with statements that are clearly false. Try the reversed statement ("The customer does not request a refund") and check the two answers add up to roughly 1 |
| Label bias | "A" or "0" gets extra probability just for being first | Same shuffle test planned for choice (ROADMAP step 14). For noul, swap which label means true |
| Score hides a split | Probabilities 0.5, 0.0, 0.5 give a score of 1.0, which reads as "middle" though the model never thought so | Always return `probabilities` beside `score`, and look at them when a score sits mid-scale |
| Score levels not really ordered | Middle levels get skipped or confused | Write each level so it clearly sits between its neighbours. Lowest first, always |
| Uncalibrated probabilities | noul says 0.9 and is right 70% of the time | Same calibration work as choice (ROADMAP steps 12 and 13). Noul and score inherit whatever `decide()` has |
| Questions answered one at a time | Jev answers many questions in one request; we make one call per question | Fine for now. Send them concurrently with the state first in the prompt (ROADMAP step 16) |

## How to test each one

- **Noul**: 20 statements with known true or false answers. Count how many land on the right side of 0.5. Then check the reversed-statement test above.
- **Score**: 20 examples that a person has rated on the same rubric. Compare the model's score with the human level. A score within 0.5 of the human level is a reasonable first pass mark.

## Sources

Public pages read on 2026-11-12. The first is TypeSafe's own; the rest are third-party and less authoritative.

- TypeSafe quick start: https://docs.typesafe.ai/introduction/quickstart
- Cloudflare AI model page for Jev: https://developers.cloudflare.com/ai/models/typesafe/jev/
- Vercel guide: https://vercel.com/kb/guide/typesafe-jev-and-ai-sdk
- Third-party API reference: https://typesafe-jev.com/en/guides/api/
- Third-party API reference: https://infron.ai/models/typesafe/jev-1.13/api-reference
