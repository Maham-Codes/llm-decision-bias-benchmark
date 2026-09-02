# Example run — metrics

Pilot run, `N_REPEATS = 3`, model = `gemini-3.5-flash-lite`, temperature = 0.7
(single-agent) / 0.7 & 0.9 (advisors) / 0.3 (aggregator). The multi-agent pass hit a
free-tier rate limit partway through, so only 2 of 5 framings (`neutral`, `loss_framed`)
completed for that half of the experiment — treat the multi-agent numbers as illustrative
of the metric, not a full comparison.

## Mean allocation per framing (single-agent)

| framing | project_a | project_b | project_c |
|---|---|---|---|
| authority_framed | 3333.33 | 4333.33 | 2333.33 |
| gain_framed | 3333.00 | 3333.00 | 3334.00 |
| loss_framed | 3500.00 | 3833.33 | 2666.67 |
| neutral | 4000.00 | 4000.00 | 2000.00 |
| order_reversed | 3000.00 | 4000.00 | 3000.00 |

## Within-framing std (noise)

| framing | project_a | project_b | project_c |
|---|---|---|---|
| authority_framed | 577.35 | 577.35 | 577.35 |
| gain_framed | 0.00 | 0.00 | 0.00 |
| loss_framed | 0.00 | 288.68 | 288.68 |
| neutral | 0.00 | 0.00 | 0.00 |
| order_reversed | 0.00 | 0.00 | 0.00 |

## Invariance ratio (between-framing std ÷ mean within-framing std)

| project | single-agent | multi-agent (2 framings only) |
|---|---|---|
| project_a | 3.16 | 5.72 |
| project_b | 2.11 | inf* |
| project_c | 3.04 | 8.16 |

\* `inf` because within-framing std was 0 for one of the two completed framings —
an artifact of the truncated run, not a real signal. Re-run with all 5 framings
completed before trusting this column.

## ANOVA (project_a allocation across framings)

F = 4.722, p = 0.0249 — significant at the 0.05 level.

## Pairwise cosine similarity of mean allocation vectors

All pairs were fairly similar (0.96–0.99), which makes sense since the three
allocations always sum to $10,000 — cosine similarity is a weak signal here compared
to the raw invariance ratio and ANOVA.
