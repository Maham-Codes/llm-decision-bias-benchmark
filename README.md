# LLM Decision-Advisor Bias & Agent Benchmark

**A pilot benchmark for stress-testing LLMs as decision advisors: measuring framing
sensitivity, advice invariance, and whether multi-agent protocols reduce decision risk.**

LLMs are increasingly used as advisors that recommend and justify decisions in business,
operations, and financial contexts, but it's not obvious whether that advice reflects
the actual economics of a decision or just the way the question happened to be phrased.
This project builds a small Python benchmark to test that directly: it takes a canonical
managerial decision task (allocating a fixed budget across competing projects), asks an
LLM to solve it under several economically identical but differently worded framings, and
measures how much the advice moves. It then tests whether a lightweight multi-agent
protocol — independent second opinions reconciled by an aggregator — makes the advice
more reliable, or just adds noise.

## What this covers

- **Equivalent decision scenarios varying only in framing** — 5 economically identical
  framings of a $10,000 allocation task across three fictional projects (Marketing, R&D,
  Operations): `neutral`, `loss_framed` (emphasizes risk of underfunding), `gain_framed`
  (emphasizes maximizing success), `authority_framed` (model told to act as CFO and
  justify the decision), and `order_reversed` (same task, projects listed in reverse
  order, to test position bias). Each framing is run several times to separate genuine
  framing effects from ordinary sampling noise.
- **A Python pipeline for querying LLMs via API** — `query_advisor()` calls the Gemini
  API with a Pydantic-enforced JSON schema so every response is directly comparable.
- **Structured, comparable outputs** — every response is forced into an `Allocation`
  schema (three dollar amounts + rationale) instead of parsed from free text.
- **Framing-sensitivity / advice-invariance metrics** — within-framing standard deviation
  (noise from repeating the same framing), between-framing standard deviation (how much
  the average allocation moves across framings — the actual bias signal), an "invariance
  ratio" (between ÷ within), one-way ANOVA per project to check statistical significance,
  and cosine similarity between framings' mean allocation vectors.
- **A multi-agent protocol test** — two independently sampled advisor calls reconciled
  by a third aggregator call ("independent second opinions" + aggregation), with the
  same invariance metrics recomputed on the aggregated output to see whether it reduces
  decision risk relative to a single call.

What it doesn't cover yet, and what a more thorough version would need to add: a decision
task with an analytically derivable optimum (so decision quality and regret can be
measured, not just consistency), a second model family to test whether bias is
model-specific or general, a larger `N_REPEATS`, and a proper literature tie-in to the
behavioral-economics work on loss aversion, anchoring, and framing effects.

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
  to an optimal allocation — a natural next step.
- To strengthen the study: increase `N_REPEATS`, add a second model family (OpenAI or
  Claude, using the same `Allocation` schema) to see whether the bias is Gemini-specific
  or general to LLMs, add a decision task with an analytically derivable optimum, and
  extend the multi-agent protocol to include structured critique-and-revision, not just
  independent-advisors-plus-aggregation.

## License

MIT — see [LICENSE](LICENSE).
