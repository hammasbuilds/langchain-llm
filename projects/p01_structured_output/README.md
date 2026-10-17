# 01 · Structured output under pressure (LangChain, Ollama, Pydantic v2)

**A valid document and a correct one are different things, and the metric everyone reports
cannot tell them apart.**

Four ways to get schema-shaped JSON out of a small local model, measured on a corpus of
clinical-trial abstracts where some fields are **genuinely absent from the source** and the
correct answer is `null`.

The project was built to prove that grammar-constrained decoding buys validity at the cost of
inventing content. It does not. What it proved instead is that the *standard way of measuring
this* invents a result.

---

## Results

Run the grid once with the obvious scoring — accuracy over the extractions that parsed — and
the hardest schema level says this:

| L5 (discriminated union) | validity | accuracy |
|---|---|---|
| `prompt_only` | 25% | **96%** |
| `constrained` | 100% | 77% |

A clean story: constraints cost you 19 points of accuracy. It is also false. `prompt_only`
parsed 3 of 12 abstracts, and that 96% is its score *on the three it managed* — necessarily
the easy ones. The nine failures contributed nothing to the denominator, so they cost it
nothing.

Charge every failed extraction for the fields it owed, change nothing else, and re-run:

| L5 (discriminated union) | validity | accuracy (as usually measured) | accuracy (corrected) |
|---|---|---|---|
| `prompt_only` | 25% | 96% | **23%** |
| `repaired` | 25% | 96% | **23%** |
| `error_feedback` | 50% | 92% | **45%** |
| `constrained` | 100% | 77% | **77%** |

> The conclusion does not move. It **inverts**. Constrained decoding is not trading accuracy
> for validity — it is **more than three times more accurate** at this level, and the ordinary
> metric hid that behind an average computed over survivors.

This is the same error as reporting a trial's outcome only for the patients who completed it.

## What the original hypothesis was, and why it failed

The corpus was built so that four fields are frequently absent — 4 abstracts omit the
participant count, 6 omit a confidence interval, 5 omit a p-value, 5 have no registration ID.
A grammar that *permits* `null` does not make `null` likely, so a constrained model was
expected to fill those gaps with well-typed inventions.

It does, on 27% of them. So does every other strategy, on 27% of them:

| strategy | valid | accuracy | fabrication | omission | calls |
|---|---|---|---|---|---|
| `prompt_only` | 85% | 74% | 27% | 1% | 60 |
| `repaired` | 85% | 74% | 27% | 1% | 60 |
| `error_feedback` | 90% | 79% | 24% | 1% | 75 |
| `constrained` | **100%** | **84%** | 27% | 4% | 60 |

**Fabrication is a property of the model, not of the decoding strategy.** It does not move when
the decoding method changes, because the model's willingness to invent a participant count has
nothing to do with how its tokens are sampled. The corpus was designed to catch constrained
decoding making things up, and it caught the evaluation method making something up instead.

Two real costs survive, and they are footnotes rather than headlines:

- At **L4 only**, forcing validity does cost accuracy: 82% against 88%.
- Omission rises from 1% to 4% — constrained decoding returns `null` for a stated value
  slightly more often.

## The strategies

| | what it does | result |
|---|---|---|
| `prompt_only` | Ask for JSON, parse what comes back. | 85% valid |
| `repaired` | Strip fences, fix trailing commas and Python literals, then parse. | **identical to `prompt_only`, on every cell** |
| `error_feedback` | On a validation failure, hand the model its own errors; up to 3 attempts. | +5 points validity for 25% more calls |
| `constrained` | Pass the JSON Schema to ollama as `format`. Invalid output is unrepresentable. | 100% valid, best accuracy |

`repaired` producing numbers byte-identical to `prompt_only` across all twenty cells is a
negative result worth keeping: when this model emitted JSON at all, the JSON was already
clean. The repair layer solves a problem `qwen2.5:3b-instruct` does not have. It is kept
because it is nearly free, and because "the obvious defensive layer did nothing" is more
useful to a reader than silence.

## The schema ladder

Difficulty rises while the task is held constant, so a drop in accuracy is attributable to the
schema rather than to the question.

| | shape | validity (worst strategy) |
|---|---|---|
| L1 | three required strings | 100% |
| L2 | an enum and a nullable int | 100% |
| L3 | a list of nested objects | 100% |
| L4 | constrained numerics, a regex-patterned ID, a nested interval | 100% |
| L5 | a **discriminated union** | **25%** |

Everything up to L4 is handled by a 3B model without difficulty. The cliff is at L5: a
discriminated union requires committing to one branch and then satisfying only that branch's
fields, and unconstrained generation gets it right a quarter of the time.

## The corpus

Twelve abstracts, hand-written for this repo, with ground truth written before any model saw
them. **They were written for this benchmark and the registration IDs are made up** — real abstracts were not
used because their ground truth is itself a judgement call, and because attaching extracted
medical claims to a toy model is a good way to put a wrong number on the internet.

What makes them useful is the pattern of omissions, pinned by
`test_omission_balance_is_pinned` so that a corpus edit cannot silently change what a
fabrication rate means.

## Run it

```bash
make install
ollama pull qwen2.5:3b-instruct

make bench01                # regenerates RESULTS.md; nothing in this README is typed by hand
```

Results are `qwen2.5:3b-instruct` at temperature 0 on one machine. The abstracts were written for this benchmark with hand-written ground truth, so nothing here is a claim about clinical evidence.
