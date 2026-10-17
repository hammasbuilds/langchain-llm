# langchain-llm

[![ci](https://github.com/hammasbuilds/langchain-llm/actions/workflows/ci.yml/badge.svg)](https://github.com/hammasbuilds/langchain-llm/actions/workflows/ci.yml)
![python](https://img.shields.io/badge/python-3.11%2B-blue)
![models](https://img.shields.io/badge/models-local%20via%20ollama-success)
![api keys](https://img.shields.io/badge/API%20keys-none%20required-success)
[![license](https://img.shields.io/badge/license-MIT-green)](LICENSE)

We built five LangChain projects, each around a technique that is usually shown working, and
measured the case where it does not: structured output, retrieval, conversation memory, prompt
injection defences and LLM-as-judge. Every project holds everything still, varies one thing and
counts what survives, on a local model through Ollama, so every number can be reproduced with no
API key and no bill. It is for engineers building LLM pipelines with LangChain who want to know
where the standard recipe and the standard metric go wrong.

## Projects

| | Project | Result | Tests |
|---|---|---|---|
| [**01**](projects/p01_structured_output/) | [**Structured output under pressure**](projects/p01_structured_output/) | On the hardest schema (discriminated union), plain prompting parses 25% of abstracts and constrained decoding 100%. Scored over everything, constrained decoding is **77%** accurate against **23%**; scored only over what parsed, the ranking inverts. Fabrication is 27% for every strategy. | 43 |
| [**02**](projects/p02_retrieval_absences/) | [**Retrieval manufactures absences**](projects/p02_retrieval_absences/) | Retrieving from a paper the model could have read whole cut accuracy from **82% to 29%**; 61 of the 89 fields retrieval withheld came back as invented values. Per-field queries recover **79%** in a third of the context. | 28 |
| [**03**](projects/p03_memory_recall/) | [**Memory that forgets the wrong thing**](projects/p03_memory_recall/) | Summary-plus-window memory scores the same as a free sliding window (**42% survival, 33% recall**) at 5 model calls. The naive summariser keeps every preference and drops every hard constraint; one paragraph in its prompt takes survival to **100%**. | 28 |
| [**04**](projects/p04_injection_defence/) | [**The injection that isn't an instruction**](projects/p04_injection_defence/) | The model refused every instruction a document gave it and believed every fact it stated. The one attack that worked, a quiet "erratum" in the document, succeeded **100%** of the time under every defence. | 30 |
| [**05**](projects/p05_judge_bias/) | [**The judge prefers the longer answer**](projects/p05_judge_bias/) | Between two equally correct answers the judge chose the longer one **30 times out of 30**, including under a rubric saying length is not a criterion. A tie option made it call a tie on **14 of 16** pairs where one answer was wrong. | 29 |

## What it does

- **01** extracts schema-shaped JSON from clinical-trial abstracts with four strategies (prompt
  only, JSON repair, error feedback, grammar-constrained decoding) at five schema levels, and
  scores validity, exactness, **fabrication** (truth is `null`, model gave a value) and
  **omission** (truth is a value, model gave `null`) separately.
- **02** adds retrieval before extraction (`nomic-embed-text`, cosine similarity in plain
  Python) and classifies every field by what the model could have known: grounded, phantom,
  honest null or fabrication.
- **03** plants 12 facts in a 24-turn conversation and checks each, under six memory
  strategies, twice: is the value still in the context, and can the model produce it.
- **04** writes six attack objectives in a loud and a quiet register inside a retrieved policy
  document and runs them against five defences, separating "the filter removed it" from "the
  model declined it".
- **05** judges answer pairs in both orders under three judge prompts, using tie pairs (equal
  correctness, different length) where any preference is bias.
- `shared/models.py` registers the model fleet by capability; each benchmark uses what is
  pulled and names the models it used. `shared/llm.py` counts every call.

## Sample input / output

`make bench01` runs project 01 and writes
[`RESULTS.md`](projects/p01_structured_output/RESULTS.md). Input: 12 clinical-trial abstracts,
a schema at five levels, four strategies, `qwen2.5:3b-instruct` at temperature 0. Output, as
committed (20 cells, 255 model calls, 4.0 minutes):

```
## By strategy, pooled over models and levels

| strategy | valid | accuracy | fabrication | omission | calls |
|---|---|---|---|---|---|
| `prompt_only` | 85% | 74% | 27% | 1% | 60 |
| `repaired` | 85% | 74% | 27% | 1% | 60 |
| `error_feedback` | 90% | 79% | 24% | 1% | 75 |
| `constrained` | 100% | 84% | 27% | 4% | 60 |

## By schema level (L5)

| level | strategy | valid | accuracy | fabrication |
|---|---|---|---|---|
| L5_union | `prompt_only` | 25% | 23% | n/a |
| L5_union | `repaired` | 25% | 23% | n/a |
| L5_union | `error_feedback` | 50% | 45% | 0% |
| L5_union | `constrained` | 100% | 77% | 25% |
```

Each project's `RESULTS.md` is written by its `benchmark.py`; no number in them is typed by hand.

## Results

| Project | Key numbers | Source |
|---|---|---|
| 01 | L5 accuracy, failures charged: constrained 77%, prompt only 23%, error feedback 45% | [RESULTS.md](projects/p01_structured_output/RESULTS.md) |
| 02 | full context 82%, single query k=4 29% (phantom rate 69%), per-field n=1 79% | [RESULTS.md](projects/p02_retrieval_absences/RESULTS.md) |
| 03 | survival / recall: full 100/100, window 42/33, summary+window naive 42/33, guarded 100/92 | [RESULTS.md](projects/p03_memory_recall/RESULTS.md) |
| 04 | success rate per defence, loud / quiet: none 17/17, hierarchy 17/17, sanitise 0/17 (one objective at 100%, five at 0%) | [RESULTS.md](projects/p04_injection_defence/RESULTS.md) |
| 05 | longer answer won: plain 14/14, rubric 14/14, tie allowed 2/2; tie option said tie on 14 of 16 wrong-answer pairs | [RESULTS.md](projects/p05_judge_bias/RESULTS.md) |

All numbers are `qwen2.5:3b-instruct` (and `nomic-embed-text` for 02) on one machine at
temperature 0; project 03 repeats its summarising strategies five times and 04 three times.

## Tech stack

- Python 3.11+
- LangChain: `langchain-core`, `langchain-ollama`
- Pydantic v2 for schemas and validation; `httpx`
- Ollama with `qwen2.5:3b-instruct` and `nomic-embed-text` (the registry also lists
  `granite3.3:2b`, `qwen2.5-coder:3b`, `llama3.2:3b` and `qwen2.5:7b-instruct`)
- pytest (+ pytest-asyncio), ruff, hatchling, uv, GitHub Actions

## Run it

```bash
git clone https://github.com/hammasbuilds/langchain-llm
cd langchain-llm

make install                 # uv sync --extra dev
make models                  # ollama pull qwen2.5:3b-instruct and nomic-embed-text

make test                    # 153 tests, no GPU and no ollama needed
make test-live               # adds the 5 tests that need a running ollama
make bench                   # regenerates every project's RESULTS.md from real calls
make bench01                 # or one project at a time (bench01 ... bench05)
```

The corpora are written for the benchmarks (abstracts, papers, conversations, policy
documents, answer pairs) so that ground truth is exact; the abstracts' registration IDs are
made up and nothing here is a claim about real clinical evidence.

## Licence

MIT, see [LICENSE](LICENSE).
