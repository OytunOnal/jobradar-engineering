# What was measured, and what it decided

Every ranking layer in JobRadar was chosen by a measurement against a frozen ground truth. This page summarises the measurements and the decisions they forced; the raw records (one JSON object per posting, per run) live in the private repository. Numbers are as measured; where a sample is small the text says so, because a small sample gives direction, not a rate.

## 1. The judge's settings: reasoning effort, temperature, ceiling

**Question.** GLM-5.3-Flash offers `reasoning_effort` low / high / max. Which one, and at what output ceiling?

**Method.** The same 133 postings, judged under each setting, compared with each other and with two earlier judges' verdicts. Then 20 postings × 3 identical runs to measure self-consistency.

**Findings.**
- `low` was bimodal: 223 reasoning tokens on one run of a posting and 4,328 on another of the same posting, and it mis-read requirement lists in both directions (counted nice-to-haves, missed listed items). Stopped at 74 postings.
- `high` at an 8,000-token ceiling was the full run.
- `max` at 12,000 agreed with `high` on 86% of verdicts and 97% of gates, was more generous on "no gaps → strong", cost about three times the output tokens, and missed one explicit location gate that `high` caught. Stopped at 108.
- `medium` does not exist on this model (HTTP 400).
- Self-consistency at `high`: 13 of 20 postings changed verdict across three runs, and every flip went through a *count* — a borderline requirement counted or not, absent items merged or split.

**Decisions.** Effort `high`, temperature 0, ceiling 8,000. And the prompt had to stop asking for a count.

## 2. The ledger prompt (v8.5 → v8.6.3)

**Question.** Can a per-requirement ledger make the verdict stable and explainable?

**Method.** The prompt was rewritten in stages: gates first (work authorization, language, location), then the role's identity in one sentence, then a line-by-line ledger — each must-have and nice-to-have marked direct / adjacent / none with the evidence, and the ramp to close a gap marked small or substantial — from which the code computes a 0-100 score and a verdict threshold (≥75 strong, 50-74 possible). Each stage was re-measured on the same postings, and a stratified 1,183-posting set was judged with the final version to serve as ground truth for the ranking layers.

**Findings.** Two prompt-independent bugs surfaced on the way: the posting parser dropped the "nice-to-have" section in 12 of 133 postings because the first line under a heading was itself treated as a heading, and the heading vocabulary lacked forms like "What Sets You Apart" and "Bonus Points". Fixed, then measured again: 12 → 1 → 0. A version that added an identity paragraph (v8.6.2) lowered scores across the board; the identity stayed but its weight in the scoring was removed (v8.6.3). Frozen 2026-09-06.

**Cost.** 0.1-0.2 US cents per judged posting at these settings, with two concurrent streams giving ~48% prefix-cache hits (a single stream hit the cache on 6 of 133 calls).

## 3. Embeddings: the query, not the model, decides

**Question.** Which embedding model, and what text to embed on the query side?

**Method.** A bake-off with pre-frozen queries: several small embedding models, each ranking the pool against (a) the raw CV and (b) a set of short "pseudo-adverts" — imagined job postings written for the CV. Judged against the ground truth by precision in the top 100 and by how many of the best postings were found.

**Findings.** The advert query beat the raw CV on every model (precision at 100: 0.46 vs 0.30). Similarity is taken as the *maximum* over the adverts, not the mean: a CV that fits three roles has three adverts, and averaging would bury exactly the specialised postings that fit best. A 1,500-character and a 2,500-character posting window measured the same. Qwen3-Embedding-0.6B was kept.

**Decision.** Adverts are generated from the CV by the model at profile time; the raw CV is a fallback.

## 4. The keyword/embedding blend

**Question.** How much of the ranking should the keyword score keep once the embedding is good?

**Method.** Rank by keyword, rank by similarity, blend the ranks with weight *w* on the keyword side, sweep *w*, score against the 1,183-posting ground truth.

**Findings.** Strong postings in the top fifth: 34 / 34 / 34 / 34 / 33 / 31 % at *w* = 0 / 0.1 / 0.2 / 0.3 / 0.4 / 0.5; the best postings found: 75 / 75 / 74 / 74 / 72 / 68 %. A flat plateau to 0.3, then a cost. An earlier sweep with the raw-CV query had put the weight at 0.4; a stronger query left the keyword score nothing to add.

**Decision.** *w* = 0.2 — not zero, because a posting with no vector yet still needs a rank.

## 5. Storing the ranking key

**Question.** The blend was measured on *ranks*, which are relative to the whole pool and cannot be stored. Does an absolute stored score reproduce them?

**Findings.** On the ground truth, a stored score blend tracked the rank blend at Spearman 0.987-0.990 and lost one to two points of precision; on the live pool of 102,619 postings the two orders agree at 0.97 with 87% of the top 100 shared. Similarity alone matched the rank blend exactly on the ground truth — the keyword weight's only remaining job is the not-yet-embedded posting.

**Decision.** Store similarity and an absolute blend; the blend is NULL until a vector exists, because scoring an unembedded posting by keyword alone would put it on a scale it does not share.

## 6. Hosted embedding

**Question.** Can the embedding leave the laptop?

**Findings (September 2026).** DeepInfra serves the same Qwen3-Embedding-0.6B checkpoint at $0.01 per million tokens: about fifty cents per hundred thousand postings. Vector-space compatibility with the local model is likely from the model card and is the next thing to measure.

## What is deliberately not here

Posting texts, the CV the ground truth was built against, and the per-posting records. They are personal or third-party data and stay private; the measurements are what the portfolio is for.
