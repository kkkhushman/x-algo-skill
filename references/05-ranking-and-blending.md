# 05 — Ranking and Blending

The math that turns 19 engagement probabilities into a single rank, and the algorithms that mix ads + W2F + prompts into the final feed.

## From Phoenix probabilities to `weighted_score`

`PhoenixScorer` fills each candidate's `phoenix_scores` with action probabilities + continuous predictions. `RankingScorer` then computes a linear combination (`home-mixer/scorers/ranking_scorer.rs:118-183`):

```
weighted_score =
    favorite_score      × w_favorite
  + reply_score         × w_reply
  + retweet_score       × w_retweet
  + photo_expand_score  × w_photo_expand
  + click_score         × w_click
  + profile_click_score × w_profile_click
  + vqv_score           × w_vqv                          // gated, see below
  + share_score         × w_share
  + share_via_dm_score  × w_share_via_dm
  + share_via_copy_link_score × w_share_via_copy_link
  + dwell_score         × w_dwell
  + quote_score         × w_quote
  + quoted_click_score  × w_quoted_click
  + quoted_vqv_score    × w_quoted_vqv                   // gated, see below
  + dwell_time          × w_cont_dwell_time
  + click_dwell_time    × w_cont_click_dwell_time
  + follow_author_score × w_follow_author
  + not_interested_score × w_not_interested              // negative weight
  + block_author_score  × w_block_author                 // negative weight
  + mute_author_score   × w_mute_author                  // negative weight
  + report_score        × w_report                       // negative weight
  + not_dwelled_score   × w_not_dwelled                  // negative weight
```

Then normalization (`ranking_scorer.rs:175-183`):

```
NEGATIVE_SCORES_OFFSET = (feature switch, ensures final scores stay ≥ 0)

if combined_score < 0:
    score = (combined_score + |negative_weights_sum|) / |total_weights_sum| * NEGATIVE_SCORES_OFFSET
else:
    score = combined_score + NEGATIVE_SCORES_OFFSET
```

The offset trick lets negative-weighted heads pull scores toward zero (and below) without making the final number negative, which would break downstream code expecting `≥ 0`.

## The 22 weights — all runtime-configurable

`ScoringWeights::from_params()` (`ranking_scorer.rs:42-114`) reads these feature-switch parameter names:

| Param | Action it weights |
|---|---|
| `FavoriteWeight` | like |
| `ReplyWeight` | reply |
| `RetweetWeight` | repost |
| `PhotoExpandWeight` | photo expand |
| `ClickWeight` | link click |
| `ProfileClickWeight` | profile click |
| `VqvWeight` | video quality view |
| `ShareWeight` | share |
| `ShareViaDmWeight` | share via DM |
| `ShareViaCopyLinkWeight` | share via copy link |
| `DwellWeight` | discrete dwell prediction |
| `QuoteWeight` | quote |
| `QuotedClickWeight` | clicks on quoted content |
| `QuotedVqvWeight` | VQV on quoted videos |
| `ContDwellTimeWeight` | continuous dwell time |
| `ContClickDwellTimeWeight` | continuous click-dwell time |
| `FollowAuthorWeight` | follow author |
| `NotInterestedWeight` | "not interested" (negative) |
| `BlockAuthorWeight` | block author (negative) |
| `MuteAuthorWeight` | mute author (negative) |
| `ReportWeight` | report (negative) |
| `NotDwelledWeight` | not-dwelled (negative) |

**These values are not in the repo**. They live in feature-switch config evaluated per-request based on `(user_roles, datacenter, country, language, account_age, ...)` (`server.rs:144-161`). Do not assume canonical values.

## Video gating

VQV (video quality view) weights are gated by post media:

- `w_vqv` only applies if `min_video_duration_ms > MIN_VIDEO_DURATION_MS` (feature switch). Otherwise the term contributes 0.
- `w_quoted_vqv` has its own flag `enable_quoted_vqv_duration_check` and similar gating against `quoted_video_duration_ms`.

This prevents text-only or short-video posts from getting "phantom" video-engagement boost.

## Author diversity attenuation

After `RankingScorer` sets `weighted_score`, the `AuthorDiversityScorer` produces final `score` (`home-mixer/scorers/author_diversity_scorer.rs:29-73`):

### Algorithm

```python
multiplier(author_position) = (1 - floor) * decay_factor ** author_position + floor

# Iterate candidates sorted by weighted_score descending.
# author_count[author_id] starts at 0.
for candidate in sorted_candidates:
    pos = author_count[candidate.author_id]
    candidate.score = candidate.weighted_score * multiplier(pos)
    author_count[candidate.author_id] += 1
```

### Example with `decay_factor=0.8, floor=0.1`

```
Author's 1st post:  multiplier = 0.9 * 0.8^0 + 0.1 = 1.000
Author's 2nd post:  multiplier = 0.9 * 0.8^1 + 0.1 = 0.820
Author's 3rd post:  multiplier = 0.9 * 0.8^2 + 0.1 = 0.676
Author's 4th post:  multiplier = 0.9 * 0.8^3 + 0.1 = 0.5608
Author's 5th post:  multiplier = 0.9 * 0.8^4 + 0.1 = 0.4686
                                                       ...
asymptote → 0.1   (the floor)
```

### Properties

- **Stable**: the floor prevents an author's posts from being multiplied to zero.
- **Position-based, not similarity-based**: there's no notion of topical diversity, only author-count diversity.
- **Score-preserving order locally**: a single author's posts maintain their relative order; only the magnitude shrinks.

## What sets `score` vs `weighted_score` vs `phoenix_scores`

This is **the single most confusing thing** about the home-mixer pipeline. Be precise:

| Field | Set by | Type | Meaning |
|---|---|---|---|
| `phoenix_scores.*` | `PhoenixScorer` | `f64` per action (~22 fields) | Raw per-action probabilities from the model. |
| `weighted_score` | `RankingScorer` | `f64` | Linear combination of action probs with negative-offset normalization. |
| `score` | `AuthorDiversityScorer` (or `VMRanker` overlay) | `f64` | Final sortable rank value used by `TopKScoreSelector`. |

When someone says "the Phoenix score", they could mean any of these three. **Always ask which.**

## Ads blending

After organic scoring, the For You pipeline (`ForYouCandidatePipeline`) layers in ads. Two algorithms are available; the choice comes from feature switch `AdsBlenderType`:

- `"safe_gap"` → `SafeGapAdsBlender` (`home-mixer/ads/safe_gap_blender.rs`)
- else → `PartitionOrganicAdsBlender` (`home-mixer/ads/partition_organic_blender.rs`)

### `SafeGapAdsBlender` algorithm

```
blend(scored_posts, ads):
    if ads.is_empty() or len(scored_posts) < MIN_POSTS_FOR_ADS:
        return scored_posts        # no ads at all

    # 1. Identify positions that are "safe" — no sensitive content adjacent.
    safe_gaps = find_safe_gaps(scored_posts)

    # 2. Compute desired spacing between ads (min + ideal).
    min_spacing, ideal_spacing = compute_spacing(len(scored_posts), len(ads))

    # 3. Best-fit assignment: starting from ads[0].insert_position,
    #    for each ad search forward to the safe gap closest to ideal_spacing
    #    from the previous ad. Respect min_spacing.
    placements = []
    prev_pos = 0
    for ad in ads:
        target = max(prev_pos + min_spacing, ad.insert_position)
        chosen = closest_safe_gap_to(target, ideal_spacing, safe_gaps, prev_pos)
        placements.append((chosen, ad))
        prev_pos = chosen

    # 4. Interleave ads into the organic stream at the chosen positions.
    return interleave_and_finalize(scored_posts, placements)
```

(`ads/safe_gap_blender.rs:8-95`)

### What makes a gap "safe"?

The blender consults each candidate's `brand_safety_verdict` (set by `AdsBrandSafetyHydrator` in post-selection hydration). Positions adjacent to LOW-safety content are excluded. The full logic depends on the brand-safety enum from `models/candidate.rs`.

`record_ad_risk_stats()` (`home-mixer/ads/util.rs:8-20`) emits metrics about how many gaps were rejected, useful for debugging "why aren't my ads being placed?"

### `PartitionOrganicAdsBlender`

Alternative algorithm — partitions organic posts and inserts ads between partitions. Used as fallback or A/B variant.

## Other inserts via `BlenderSelector`

`home-mixer/selectors/blender_selector.rs:25-114` runs after ads are blended:

```
1. Partition FeedItems into:
     organic_posts, ads, who_to_follow, prompts, push_to_home

2. Run AdsBlender → blended (organic + ads)

3. Insert prompts at position 0  (push_to_home will override this if present)

4. Insert who_to_follow at WHO_TO_FOLLOW_POSITION  (feature switch)

5. Pin push_to_home at position 0  (final override)
```

The result is the URT timeline that the client receives.

## Why this structure?

- **Sequential scorers** because each step builds on the previous: action probs → weighted sum → diversity attenuation. Parallelizing them would require a different (and less interpretable) formulation.
- **Author diversity as a separate scorer** so its parameters (`decay_factor`, `floor`) can be feature-switched independently of action weights.
- **Ads blending as a Selector**, not a Scorer, because it operates on the post-selection set and produces a non-sortable interleaved feed.
- **Brand safety as a post-selection Hydrator** so it only runs on the top-K — calling the brand-safety service on every candidate would be prohibitively expensive.
- **Push-to-home as a final-override insert** so editorial pins (e.g., breaking news pinned by ops) override everything else.

## Things you should NOT assume about the weights

- They're NOT in the repo. Anyone telling you "the favorite weight is 1.0" is making it up.
- They're NOT the same for every user. New accounts, premium accounts, accounts in specific countries, accounts using specific clients — all get different weights.
- They CAN be zero. Several action weights are likely 0 for new users or low-engagement users where the model's signal is weak.
- They CAN be negative for actions that look "positive" — e.g., if A/B testing shows that boosting profile clicks degrades long-term retention, `ProfileClickWeight` can flip negative.
- They CHANGE. Feature switches are updated continuously without code changes.

## Things you should NOT assume about the model

- The weights you see in `phoenix/run_pipeline.py` step 5 (`1.0*fav + 0.5*reply + 0.3*rt + 0.2*dwell`) are **a demo only**. They are not the production weights and they do not include 18 of the 22 action heads.
- The model outputs **logits**; the home-mixer's `phoenix_scores` are **sigmoid-transformed probabilities**. Don't mix the two.
- VQV gating happens **at the home-mixer level**, not inside the model — the model emits `vqv_score` for every post regardless of duration; home-mixer's `RankingScorer` decides whether to apply the weight.
