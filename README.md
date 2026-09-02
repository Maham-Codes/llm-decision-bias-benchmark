# LLM Decision-Advisor Bias & Agent Benchmark

**A pilot benchmark for stress-testing LLMs as decision advisors — measuring framing
sensitivity, advice invariance, and whether multi-agent protocols improve decision
reliability.**

This notebook is a small-scale, working proof of concept for the kind of pipeline
described in Mitacs Globalink project *"Can AI Managers Be Trusted? Stress-Testing LLM
Advisors for Bias, Invariance, and Decision Risk"* (Mohamed Drira, St. Mary's
University). It tests whether an LLM's advice on a canonical managerial decision task —
allocating a fixed budget across competing projects — changes depending on how the
question is *worded* despite being economically identical, and whether routing the
question through a simple multi-agent protocol (two independent advisors + one
aggregator, i.e. a structured "independent second opinion" step) improves or degrades
that reliability.

## Relevance to the Mitacs project brief

The project brief asks for four phases: (1) literature review on LLM evaluation and
decision bias, (2) equivalent decision scenarios + a Python pipeline to query LLMs via
API, (3) extracting structured outputs and computing framing-sensitivity / advice-invariance
metrics, (4) testing whether multi-agent protocols (independent second opinions,
aggregation) improve reliability. This notebook is a working, if small-scale,
implementation of phases 2–4:

| Brief asks for | This notebook does |
|---|---|
| Equivalent decision scenarios varying only in framing | 5 economically identical framings of a $10,000 allocation task (neutral, loss-framed, gain-framed, authority-framed, order-reversed) |
| Python pipeline querying LLMs via API | `query_advisor()` calling the Gemini API with a Pydantic-enforced JSON schema |
| Structured, comparable outputs | Every response is forced into an `Allocation` schema (three dollar amounts + rationale) instead of parsed from free text |
| Framing-sensitivity / advice-invariance metrics | Within- vs. between-framing standard deviation, an "invariance ratio," one-way ANOVA, and cosine similarity between framings' mean allocation vectors |
| Multi-agent protocols (independent second opinions + aggregation) | Two independently sampled advisor calls reconciled by a third aggregator call, with the same invariance metrics recomputed on the aggregated output |

What it doesn't cover yet, and what a full Mitacs-scale version would need to add:
a decision task with an analytically derivable optimum (so "decision quality" and
"regret" can be measured, not just consistency), a second model family to test whether
bias is model-specific or general, a larger `N_REPEATS`, and a literature review tying
the framing effects observed here to the psychology/behavioral-economics literature on
loss aversion, anchoring, and framing effects.

## The idea

The model is given the same underlying decision five times, each phrased differently:

| Framing | What changes |
|---|---|
| `neutral` | Plain instructions, no framing |
| `loss_framed` | Emphasizes the risk of underfunding |
| `gain_framed` | Emphasizes maximizing success |
| `authority_framed` | Model is told to act as CFO and justify the decision |
| `order_reversed` | Same task, projects listed in reverse order (tests position bias) |

Every framing asks the model to split **$10,000** across three fictional projects
(Marketing, R&D, Operations) and return a structured JSON allocation. Each framing is run
several times to separate genuine framing effects from ordinary sampling noise, and the
whole thing is repeated through a lightweight multi-agent setup (two independently
sampled "advisors" reconciled by a third "aggregator" call) to see whether that reduces
the framing sensitivity.

## What it measures

- **Within-framing std** — noise from repeating the *same* framing multiple times.
- **Between-framing std** — how much the *average* allocation moves across framings
  (the actual bias signal).
- **Invariance ratio** (between ÷ within) — near 0 means framing barely matters; large
  means framing dominates the answer.
- **One-way ANOVA** on each project's allocation across framings, to check statistical
  significance.
- **Cosine similarity** between each pair of framings' mean allocation vectors.
- The same metrics recomputed on the multi-agent aggregator's output, for a direct
  single-agent vs. multi-agent comparison.

## Repo contents

```
llm_decision_bias_benchmark.ipynb   the notebook (run top to bottom in Colab or Jupyter)
sample_outputs/
  framing_comparison.png            example chart from a completed run
  results_summary.md                metrics from that same example run
requirements.txt
```

## Requirements

- Python 3.9+
- A [Gemini API key](https://aistudio.google.com/apikey) (the notebook uses
  `google-genai`; swapping in another provider just means rewriting `query_advisor`)
- `pip install -r requirements.txt`

## Running it

1. Open `llm_decision_bias_benchmark.ipynb` in Google Colab or Jupyter.
2. Set your API key as an environment variable, e.g. `GEMINI_API_KEY`
   (the notebook shows how to do this with Colab Secrets so the key never gets
   pasted into a shared file).
3. Run the cells in order. By default it makes ~5 framings × 3 repeats for the
   single-agent pass, plus 5 × 3 × 3 calls for the multi-agent pass — cheap on
   Gemini Flash pricing, but watch the free-tier rate limit (15 requests/min) and
   raise `SLEEP_SECONDS` or lower `N_REPEATS` if you hit `429` errors.
4. Results are saved to `single_agent_results.csv` and `multi_agent_results.csv`,
   and the comparison chart to `framing_comparison.png`.

## Example results (from a pilot run, `N_REPEATS = 3`)

**Framing changed the allocation, and the effect was statistically significant** for
Project A's budget share (one-way ANOVA, F = 4.72, p = 0.025). Mean allocations ranged
from roughly $3,000–$4,000 for Project A depending only on how the prompt was worded —
with the identical underlying decision.

The **invariance ratio actually got worse, not better, after aggregation** in this pilot
run (e.g. Project A: 3.16 single-agent → 5.72 multi-agent). That's the opposite of the
hypothesis that a second-opinion-plus-aggregator step should cancel out framing noise —
though this run only completed 2 of 5 framings for the multi-agent pass before hitting a
rate limit, so it's a suggestive result, not a settled one. See
`sample_outputs/results_summary.md` for the full numbers and
`sample_outputs/framing_comparison.png` for the chart.

## Limitations

- This is a pilot: one model family (Gemini Flash), a small number of repeats, and one
  toy decision task. Treat findings as directional, not conclusive.
- Free-tier rate limits can truncate a run mid-experiment (visible in the example
  outputs above) — for a clean run, use a paid tier or raise `SLEEP_SECONDS`.
- No ground-truth optimum is defined for the allocation task, so this pilot only measures
  *consistency* (invariance across framings), not *decision quality* or *regret* relative
  to an optimal allocation — a natural next step, and one the full project brief
  specifically calls for.
- To strengthen the study: increase `N_REPEATS`, add a second model family (OpenAI or
  Claude, using the same `Allocation` schema) to see whether the bias is Gemini-specific
  or general to LLMs, add a decision task with an analytically derivable optimum, and
  extend the multi-agent protocol to include structured critique-and-revision, not just
  independent-advisors-plus-aggregation.

## License

MIT — see [LICENSE](LICENSE).
