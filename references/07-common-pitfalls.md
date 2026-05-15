# 07 — Common Pitfalls

Things agents commonly get wrong when reasoning about this codebase. Each pitfall is paired with the correct mental model.

## 1. "The 2023 Twitter algorithm and the 2026 X algorithm are similar"

**Wrong.** The 2026 release is a near-total rewrite. Gone: SimClusters, TwHIN, real-graph, tweepcred, Heavy Ranker, twml, navi (TF-based model server), product-mixer, recos-injector, Scala-everywhere. Kept conceptually: home-mixer (but rewritten in Rust), trust-and-safety models (now Grox).

The architectural through-line is the **pipeline-of-stages** pattern, but every component has been replaced. If a question references SimClusters or TwHIN, the user is talking about the 2023 release at `twitter/the-algorithm`, not the current one.

## 2. "Candidates compete with each other during ranking"

**Wrong.** The whole point of the candidate-isolation attention mask (`phoenix/grok.py:39-71`) is that **each candidate's score is computed independently** given the user context. They do not attend to each other.

Consequences:
- Scoring a post in batch with 100 others vs alone yields the **same score**.
- You can chunk candidates into mini-batches (`run_pipeline.py` uses `candidate_seq_len=64`) without affecting ranking.
- Per-`(user, candidate)` score caching is correct without any "but what about the batch composition?" caveat.

If you're tempted to design something like "let candidates softmax against each other for relative ranking", that's not what Phoenix does.

## 3. "The action weights are hardcoded constants"

**Wrong.** All 22 weights are loaded **per request** from feature switches by `ScoringWeights::from_params()` (`home-mixer/scorers/ranking_scorer.rs:42-114`). They depend on user attributes, datacenter, country, language, account age, decider flags.

**Do not** quote specific weight values. They're not in the repo. The demo weights in `phoenix/run_pipeline.py:340` (`1.0*fav + 0.5*reply + 0.3*rt + 0.2*dwell`) are illustrative only — they ignore 18 of the 22 action heads and the negative-feedback offsets.

## 4. "There are 15 action heads"

**Misleading.** The top-level README says 15, but the actual model emits **19 discrete logits + 4 continuous heads** (`phoenix/runners.py:233-264`). Canonical list:

```
 0 favorite           7 share              14 not_interested    NEG
 1 reply              8 share_via_dm       15 block_author      NEG
 2 repost             9 share_via_copy_link 16 mute_author      NEG
 3 photo_expand      10 dwell              17 report            NEG
 4 click             11 quote              18 dwell_time
 5 profile_click     12 quoted_click
 6 vqv               13 follow_author
```

Plus continuous: `dwell_time, video_watch_time, scroll_depth, +reserved`.

When counting actions, **always specify which list** — model outputs (19+4), weighted-scorer inputs (22 because home-mixer adds `not_dwelled_score`, `quoted_vqv_score`, `click_dwell_time`), or the README's marketing number (15).

## 5. "Retrieval uses FAISS / ScaNN / some ANN library"

**Wrong, at least in the published code.** `phoenix/run_pipeline.py` does a brute-force dot product against a flat L2-normalized embedding table (`sports_corpus.npz`):

```python
scores = corpus_repr @ user_repr[0]   # [N]
top_k  = np.argpartition(scores, K)[:K]
```

Production presumably uses ANN at scale, but **the repo doesn't include that code**. Don't make claims about which ANN library X uses from this codebase.

## 6. "The included Phoenix checkpoint represents production model quality"

**Wrong.** What ships is a **frozen mini checkpoint**:

- 128-dim embeddings (production is wider).
- 4 transformer layers (production has more).
- Sports-only corpus (~537K posts in a 6-hour window).
- A snapshot from continuous training, frozen in time.

Don't extrapolate model quality or NDCG numbers from this artifact. The README explicitly says so.

## 7. "Filters run in parallel"

**Wrong.** Filters in the `candidate-pipeline` framework run **sequentially**. Each filter receives only the candidates kept by the previous filter (`candidate_pipeline.rs:365-371`). This is intentional — order matters for latency optimization (cheap filters first cut the candidate set so expensive ones see fewer).

Hydrators and sources DO run in parallel. Don't confuse the two.

## 8. "Three scores on a candidate is just three names for the same thing"

**Wrong.** They're three distinct values, each set by a different scorer:

- `phoenix_scores.*` — 22 individual per-action probabilities from the model.
- `weighted_score` — linear combination of action probabilities (`RankingScorer`).
- `score` — final value after author-diversity attenuation (`AuthorDiversityScorer`).

`TopKScoreSelector` sorts by `score`, not `weighted_score`. When someone says "the Phoenix score" they could mean any of the three — clarify before answering.

## 9. "Post-selection just means 'after sorting'"

**Underselling.** Six **hydrators** and three **filters** run after `TopKScoreSelector`. These are real stages with real side effects:

- `VFCandidateHydrator` calls the Visibility Filter service for each top-K candidate.
- `AdsBrandSafetyHydrator` adds brand-safety verdicts for ads blending.
- `MutualFollowJaccardHydrator` computes pairwise similarity.
- `VFFilter` can drop candidates with `Action::Drop` visibility.
- `DedupConversationFilter` can drop entire conversation branches.

The "ranked feed" can still shrink after selection. Don't assume `selected.len() == final_result.len()`.

## 10. "Negative actions are filtered out"

**Wrong.** Negative actions (`not_interested`, `block_author`, `mute_author`, `report`, `not_dwelled`) are **scored**, not filtered. The model predicts probabilities for them, and the `RankingScorer` applies **negative weights** to those probabilities, **pushing scores down**.

A post with high P(report) won't be dropped; it'll be ranked below a post with low P(report), all else equal. Hard filtering of disallowed content happens later, via `VFFilter` (after a separate Visibility Filter service decision), not via the model's negative-action predictions.

## 11. "VQV (video quality view) weight applies to all videos"

**Wrong.** VQV is gated by `min_video_duration_ms > MIN_VIDEO_DURATION_MS` (`ranking_scorer.rs:131-144`). Posts with no video, or videos shorter than the threshold, contribute 0 to `weighted_score` from the VQV term, regardless of the model's `vqv_score` output.

Same for `quoted_vqv_score` — gated by `enable_quoted_vqv_duration_check` flag and `quoted_video_duration_ms`.

## 12. "Side effects update the response"

**Wrong.** Side effects are spawned on a background tokio task **after** the response is built and returned to the client (`candidate_pipeline.rs:419-428`). They can't influence the response. They write to Kafka, Redis, and the served-history service for downstream consumption. Their failures are silent.

## 13. "Phoenix is one model"

**Underspecified.** "Phoenix" refers to a family of models:

- **Retrieval user tower** — a full Grok transformer (heavy).
- **Retrieval candidate tower** — a lightweight MLP projection (different architecture).
- **Ranking model** — a Grok transformer with candidate-isolation masking.
- **Phoenix MOE** — mixture-of-experts variant (separate source: `PhoenixMOESource`).
- **Phoenix Topics** — topic-specialized variant (separate source: `PhoenixTopicsSource`).

The `PhoenixScorer` in home-mixer calls "the Phoenix prediction service", but that service may route to different model variants based on cluster (LAP7 vs FOU) and decider flags.

## 14. "Thunder is just an in-memory database"

**Underselling.** Thunder has non-trivial filtering logic in its read path:

- Filters out the viewer's own retweets to prevent self-retweet loops.
- Applies conversation rules: a reply is only returned if it belongs to a conversation where the parent or root is from a followed account.
- Maintains three separate timelines per author (original / secondary / video).
- Tracks soft-deletes across Kafka topics.

If you're modeling Thunder as a key-value store of "recent posts by author", you're missing where most of the complexity lives.

## 15. "Grox is a content-moderation service"

**Underselling.** Grox does:

- Spam detection (Grok VLM).
- PTOS safety classification (multi-category, with deluxe variants).
- Multimodal embedding generation (1024-dim, text + image, ASR for video).
- Summarization (for embedding context + reply ranking).
- Reply ranking signals.
- "Banger" initial-screen detection (high-engagement-likelihood posts).
- Brand-safety verdicts.

Most of Grox's output is **not** safety enforcement — it's **feature generation** for home-mixer's hydrators.

## 16. "The feed is just the Phoenix output"

**Wrong.** The For You feed is a **blended timeline** with at least four item types:

- Organic posts (ranked by Phoenix).
- Ads (positioned by `SafeGapAdsBlender`).
- Who-to-follow modules (inserted at a fixed position).
- Prompts / push-to-home items (pinned).

The `BlenderSelector` is what produces the final timeline, not the `TopKScoreSelector`.

## 17. "I can read the production weights from feature-switch defaults in the repo"

**Wrong.** The feature-switch evaluator is not in this repo. Default values in `params/*.rs` may exist, but they are **not the production values** — actual values come from runtime config that's evaluated per-user, per-DC, per-experiment-bucket.

## 18. "Quote posts and reply posts are first-class candidates"

**Partially.** Quote posts have `quoted_tweet_id, quoted_user_id` fields and dedicated hydrators (`QuoteHydrator`). Replies have separate logic in Thunder's read path (conversation-rule filtering) and in `DedupConversationFilter` (post-selection).

But they go through the same scoring path — the model treats them uniformly via `history_actions` (multi-hot includes whether the post was a quote / reply, but no distinct head).

## 19. "Continuous-action heads are just for analytics"

**Wrong.** The model emits continuous predictions (`dwell_time`, `video_watch_time`, etc.) that **feed back into the weighted scorer** via `ContDwellTimeWeight` and `ContClickDwellTimeWeight`. These let the ranking incorporate "how long would this user dwell" not just "would they tap dwell".

## 20. "The model uses learned post embeddings"

**Wrong.** The model uses **hash-based** embeddings (LCG hashing into a fixed-size bucket table). Multiple hash functions per entity (default 2) provide collision resilience. There are no per-post learned embeddings stored externally — the model only knows posts via their hash buckets.

This means **adding a post to the corpus doesn't require retraining**: as long as the post_id maps to hash buckets the model already has weights for, the model can score it. (See `phoenix/run_pipeline.py:76-114` for the hash function.)

## 21. "The 2026 algorithm follows the original Heavy Ranker architecture"

**Wrong.** Heavy Ranker (2023) was a multi-task MaskNet that ranked candidates with deep cross-feature interactions over hand-engineered signals. The 2026 Phoenix is a **transformer reading user engagement sequences** — completely different architecture, completely different feature philosophy. No hand-engineered features survive.

## 22. "If something goes wrong, the pipeline aborts"

**Wrong.** The `candidate-pipeline` framework's entire design ethos is **graceful degradation**:

- Failed sources contribute zero candidates; pipeline continues.
- Failed per-candidate hydrations retain old field values; pipeline continues.
- Failed query hydrators fall back to defaults; pipeline continues.
- Failed side effects are silent.

There's no "abort and return error" path. The For You feed is designed to **always return something**, even if it's a degraded fallback.

## Quick check before answering

If you're about to make a claim about this codebase, ask yourself:

- Am I citing the 2026 release or the 2023 one? (Repo path tells you.)
- If I'm naming a specific weight or threshold — is that value actually in the repo, or am I inferring it?
- Am I conflating `phoenix_scores`, `weighted_score`, and `score`?
- Am I assuming filters parallelize? (They don't.)
- Am I assuming candidates attend to each other? (They don't.)
- Am I assuming what ships represents production? (It doesn't.)

When in doubt, cite the file:line. The reference files in this skill all do, deliberately.
