---
name: x-for-you-algorithm
description: |
  Reason about X's open-sourced For You feed algorithm (xai-org/x-algorithm, May 2026):
  Phoenix Grok-based transformer with candidate isolation, two-tower retrieval, Thunder
  in-network store, home-mixer orchestration, candidate-pipeline framework, Grox content
  understanding, ads safe-gap blending.

  Use when: explaining how X ranks posts, designing or auditing a feed recsys, building
  on the For You codebase, comparing 2023 Twitter algorithm vs 2026 X algorithm, or
  troubleshooting questions about "candidate isolation", "Phoenix scorer", "weighted_score
  vs score", "two-tower retrieval", or why X's stack has no hand-engineered features.
metadata:
  keywords:
    - x-algorithm
    - twitter-algorithm
    - for-you-feed
    - recommendation-system
    - recsys
    - phoenix
    - grok-transformer
    - two-tower-retrieval
    - candidate-isolation
    - home-mixer
    - thunder
    - candidate-pipeline
    - grox
    - xai
    - elon-musk
    - feed-ranking
license: Apache-2.0
---

# X For You Algorithm

Deep, code-grounded understanding of X's open-source recommendation algorithm (`xai-org/x-algorithm`, released 2026-05-15). This skill is **reference knowledge**, not a workflow. Use it to reason correctly about the algorithm — what each component does, how a request flows end-to-end, where the load-bearing design decisions live, and the misconceptions that look plausible but are wrong.

## What the algorithm is

The For You feed is a **two-stage recommendation system** wrapped in a **service composition framework**. Stage one retrieves ~hundreds of candidate posts from millions using a two-tower model. Stage two ranks them with a Grok-based transformer that predicts 19 engagement probabilities per post. A linear combination of those probabilities — with per-request weights loaded from feature switches — produces the final score. Author diversity attenuation and ads blending sit on top. Trust-and-safety, brand safety, and visibility filters bracket the pipeline at both ends.

The 2023 Twitter algorithm (`twitter/the-algorithm`) is **architecturally obsolete**: SimClusters, TwHIN, real-graph, tweepcred, Heavy Ranker — all gone. The 2026 release puts a single transformer at the center and feeds it raw engagement sequences. No hand-engineered relevance features remain.

## The five components

| Component | Lang | Role |
|---|---|---|
| **`phoenix/`** | Python (JAX) | The model. Two-tower retrieval + Grok-1-derived transformer ranking. |
| **`thunder/`** | Rust | In-memory store of recent posts per author, Kafka-fed. Serves in-network candidates with sub-ms latency. |
| **`home-mixer/`** | Rust | Orchestration. Builds For You feed from Phoenix + Thunder + ads + W2F + prompts via a gRPC service (`ScoredPostsService`, `ForYouFeedService`). |
| **`candidate-pipeline/`** | Rust | The reusable composition framework. Six traits: `Source`, `QueryHydrator`, `Hydrator`, `Filter`, `Scorer`, `Selector`, `SideEffect`. Used by home-mixer to assemble pipelines. |
| **`grox/`** | Python | Content understanding. Real-time Kafka pipeline running spam classifiers, PTOS safety, multimodal embeddings (V5, 1024-dim), summarization. Writes to Strato (KV) + Kafka for hydrators to consume. |

## Request lifecycle in 7 stages

A For You feed request flows through home-mixer in this order:

1. **Query hydration** — 15 hydrators fetch user context in parallel: engagement history, social graph (follows/blocks/mutes), followed topics, starter packs, impression bloom filters, IP, mutual-follow graph, demographics, served history.
2. **Candidate sourcing** — 6 sources fan out in parallel: `ThunderSource` (in-network), `PhoenixSource` (OON retrieval), `PhoenixTopicsSource`, `PhoenixMOESource`, `TweetMixerSource`, `CachedPostsSource`. Results are flattened into one candidate list.
3. **Candidate hydration** — 10 hydrators enrich each candidate: core post data (TES), quote expansion, video duration, media flags, subscription status, author info (Gizmoduck), blocked-by, filtered topics, language code.
4. **Pre-scoring filters** — 14 filters run **sequentially**: dedup, age, self-post, retweet dedup, ineligible subscription, **previously-seen (bloom-filter-backed)**, previously-served, muted keywords, social-graph block/mute, video, topic constraints, new-user topic constraints.
5. **Scoring** — three scorers run **sequentially**: (a) `PhoenixScorer` calls the Phoenix prediction service and fills `phoenix_scores` (19 actions); (b) `RankingScorer` computes `weighted_score` as a weighted sum (weights loaded per-request from feature switches); (c) `VMRanker` / author diversity adjusts to final `score`.
6. **Selection** — `TopKScoreSelector` sorts by `score`, truncates to top K.
7. **Post-selection** — 6 more hydrators (visibility filter API, ads brand safety, engagement metrics, mutual-follow Jaccard) and 3 more filters (`VFFilter` drops VF=Drop, `AncillaryVFFilter`, `DedupConversationFilter`).

Then 8 side effects fire in parallel on a background task (Kafka publishes, Redis caches, served-history writes). The For You pipeline wraps Phoenix output with `AdsSource`, `WhoToFollowSource`, `PromptsSource`, `PushToHomeSource`, then runs `BlenderSelector` to interleave ads via the `SafeGapAdsBlender`.

Full trace with file:line citations: **`references/03-request-lifecycle.md`**.

## Core design insights you must internalize

These are the things agents commonly get wrong. **Read them before answering any question about this code.**

### 1. Candidate isolation in attention

The Phoenix ranking transformer uses a **custom attention mask** so candidates can only attend to (a) the user-context tokens preceding them and (b) themselves — never to each other. The mask is built in `phoenix/grok.py:39-71` (`make_recsys_attn_mask`):

```python
causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len), dtype=dtype))
attn_mask = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)
candidate_indices = jnp.arange(candidate_start_offset, seq_len)
attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)
```

**Why it matters:** a candidate's score doesn't change when batch composition changes → scores are cacheable, parallelizable, and stable across paginations. Most "I'm porting Twitter's algorithm" attempts miss this and end up with a bog-standard cross-encoder.

### 2. Ranking weights are runtime-loaded, not hardcoded

`RankingScorer` reads ~22 action weights from feature switches **per request** (`home-mixer/scorers/ranking_scorer.rs:42-114`). Weights vary by user attributes, datacenter, country, language, account age, decider overrides. Don't claim "the favorite weight is 1.0" — there is no canonical static weight set.

### 3. No hand-engineered features

The transformer reads raw engagement sequences (`(post_hash, author_hash, action_multihot, product_surface, timestamps)`) and figures out relevance itself. There is no SimClusters, no TwHIN, no real-graph, no Heavy Ranker. If you're "missing the embedding service", you're thinking of the 2023 release.

### 4. 19 actions, not 15

The top-level README mentions 15 actions; the actual model emits **19 discrete logits + 8 continuous heads** (`phoenix/runners.py:233-264`). Canonical ordering: `favorite, reply, repost, photo_expand, click, profile_click, vqv, share, share_via_dm, share_via_copy_link, dwell, quote, quoted_click, follow_author, not_interested, block_author, mute_author, report, dwell_time`. Last 5 (indices 14–17) are **negative-feedback heads** with negative weights — they push scores down.

### 5. Retrieval is brute-force dot product, not ANN

`run_pipeline.py` loads `sports_corpus.npz` as a **flat embedding table** (537K × D, pre-L2-normalized) and computes `corpus @ user_repr` directly. No FAISS, no ScaNN. Production likely uses ANN at scale; the repo doesn't.

### 6. The released model is a frozen mini checkpoint

128-dim embeddings, 4 transformer layers, 4 heads, sports-only corpus, single frozen snapshot from continuous training. Production is bigger and trained continuously. Don't extrapolate model quality numbers from this artifact.

### 7. The pipeline framework is asymmetric

Within `candidate-pipeline`: **sources fan out in parallel**, **hydrators fan out in parallel**, but **filters and scorers run sequentially**. Each filter sees the survivors of the previous filter; each scorer can read fields set by the previous scorer. Common mistake: assuming filters parallelize.

### 8. Three different "scores" on a candidate

- `phoenix_scores`: the 19 per-action probabilities from the model.
- `weighted_score`: the linear combination from `RankingScorer`.
- `score`: the final value after diversity attenuation / VMRanker — this is what `TopKScoreSelector` sorts by.

When someone says "Phoenix score" they could mean any of these. Always ask which.

### 9. Post-selection is real

Six hydrators and three filters run **after** the top-K is picked. Visibility filtering (deletion, spam, gore), brand safety, and conversation dedup are post-selection. The "ranked" feed can still shrink at this stage.

### 10. Two new-user thresholds, gated independently

Retrieval and ranking each have their own new-user action-count threshold and "new-user inference cluster" (LAP7 vs FOU). They can be A/B tested independently. Decider flags can override cluster routing at runtime.

### 11. Author diversity formula

`adjusted_score = weighted_score × ((1 - floor) × decay_factor^author_count + floor)`

Where `author_count` is "how many of this author's posts have already been placed ahead of this one in the sorted order." Defaults give the second post from an author ~0.82× and the third ~0.68× when `decay=0.8, floor=0.1` (`home-mixer/scorers/author_diversity_scorer.rs:29-31`).

### 12. Ads blending preserves "safe gaps"

`SafeGapAdsBlender` finds positions between sensitive content where ads can legally be placed, computes ideal spacing, and best-fits ads to those gaps (`home-mixer/ads/safe_gap_blender.rs`). Brand-safety verdicts come from a separate post-selection hydrator that the blender consults.

## Reference index

When the question goes deeper than the SKILL.md, load the relevant file:

| If the question is about… | Read |
|---|---|
| The system as a whole, gRPC surfaces, data flow | `references/01-architecture.md` |
| The Phoenix transformer, two-tower, hashing, masking, action heads | `references/02-phoenix-model.md` |
| What happens on a single For You request, stage by stage | `references/03-request-lifecycle.md` |
| The `Source` / `Hydrator` / `Filter` / `Scorer` / `Selector` / `SideEffect` traits | `references/04-pipeline-framework.md` |
| Weighted scorer formula, all 22 weights, author diversity, ads blender internals | `references/05-ranking-and-blending.md` |
| Thunder's in-memory store, Grox's plans/tasks, all the filters | `references/06-content-understanding.md` |
| Subtle bugs / misconceptions to avoid | `references/07-common-pitfalls.md` |
| What does LAP7 / FOU / Gizmoduck / Strato / UPA / VQV mean? | `references/08-glossary.md` |

## How to use this skill

This is a **knowledge skill**, not a workflow. When you (the agent) are answering a question or making a design decision related to X's recommendation algorithm:

1. Read this SKILL.md once to get the model.
2. Load the relevant reference file only when the question demands it — don't pre-load everything.
3. Cite file paths from the upstream repo (`xai-org/x-algorithm` on GitHub) when making claims. The reference files include the exact `file:line` citations.
4. If you're making a recommendation that depends on something this skill calls out as runtime-configurable or version-specific, flag the assumption explicitly.

The source of truth is the code at `https://github.com/xai-org/x-algorithm`. This skill is a navigated, sanity-checked map of it — not a replacement.
