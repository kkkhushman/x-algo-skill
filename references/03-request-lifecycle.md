# 03 — Request Lifecycle

End-to-end trace of a single For You feed request through home-mixer, with file:line citations.

## Entry point

Client calls **gRPC `ForYouFeedService::get_for_you_feed`** with `ForYouFeedQuery { viewer_id, client_app_id, country_code, language_code, seen_ids[], served_ids[], topic_ids[], excluded_topic_ids[], exclude_videos, is_polling, ... }`. (`home-mixer/server.rs:273-298`, `home-mixer/for_you_server.rs:28-74`)

## Stage 0 — Query building

`server.rs:59-127` constructs a `ScoredPostsQuery` from the incoming `ForYouFeedQuery`:

1. Validate `viewer_id != 0` (server.rs:66-68).
2. Fetch viewer data from **Gizmoduck** (roles, subscription level, muted keywords, follower count, age) with a 200ms timeout (`VIEWER_ROLES_TIMEOUT_MS`). (server.rs:73)
3. Evaluate **feature switches** based on user roles, phone status, datacenter, account age. (server.rs:138-175) — these decide things like which Phoenix cluster, which ads blender, which action weights to use.
4. Build request context with a B3 trace ID for distributed tracing.

## Stage 1 — Query hydration (parallel)

The `PhoenixCandidatePipeline` runs **15 query hydrators in parallel** to fetch user context (`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs:185-232`):

| Hydrator | Fetches |
|---|---|
| `ScoringSequenceQueryHydrator` | User action history for scoring (from UAA stream) |
| `RetrievalSequenceQueryHydrator` | User action history for retrieval |
| `BlockedUserIdsQueryHydrator` | Blocked author IDs |
| `MutedUserIdsQueryHydrator` | Muted author IDs |
| `FollowedUserIdsQueryHydrator` | Following list |
| `SubscribedUserIdsQueryHydrator` | Subscription targets |
| `MutualFollowQueryHydrator` | Mutual-follow graph slice |
| `UserDemographicsQueryHydrator` | Age, gender (inferred), etc. |
| `FollowedGrokTopicsQueryHydrator` | Topics the user follows |
| `FollowedStarterPacksQueryHydrator` | Starter packs |
| `InferredGrokTopicsQueryHydrator` | Topics inferred from behavior |
| `ImpressionBloomFilterQueryHydrator` | Bloom filter of recent impressions |
| `IpQueryHydrator` | Request IP (for geo features) |
| `UserInferredGenderQueryHydrator` | Inferred gender signal |
| `CachedPostsQueryHydrator` | Previously cached candidate IDs (Redis) |

Errors at this stage are **logged but non-fatal** — the pipeline falls back to defaults for whichever hydrator failed.

## Stage 2 — Candidate sourcing (parallel fan-out, then flatten)

Six sources run in parallel (`phoenix_candidate_pipeline.rs:250-257`):

| Source | What it returns |
|---|---|
| `ThunderSource` | In-network posts via Thunder gRPC; also extracts reply relationships into `query.in_network_replies` for later conversation dedup. (`home-mixer/sources/thunder_source.rs:51-62`) |
| `TweetMixerSource` | Social-graph-shuffled candidates |
| `PhoenixSource` | Out-of-network retrieval via Phoenix; selects cluster (LAP7 vs FOU) based on user's action-count vs `PhoenixRetrievalNewUserHistoryThreshold` and decider flags. (`home-mixer/sources/phoenix_source.rs:21-58`) |
| `PhoenixTopicsSource` | Topic-targeted Phoenix retrieval |
| `PhoenixMOESource` | Mixture-of-experts Phoenix routing |
| `CachedPostsSource` | Candidates from Redis cache |

All results flatten into one `Vec<PostCandidate>`. Failed sources contribute zero candidates; the pipeline continues. (`candidate-pipeline/candidate_pipeline.rs:266`)

## Stage 3 — Candidate hydration (parallel)

Ten hydrators run in parallel, each receiving the full candidate batch (`phoenix_candidate_pipeline.rs:259-272`):

| Hydrator | Adds to `PostCandidate` |
|---|---|
| `InNetworkCandidateHydrator` | `in_network: bool` |
| `CoreDataCandidateHydrator` | Core post data from TES — text, media, video duration, language |
| `QuoteHydrator` | Quote post expansion |
| `VideoDurationCandidateHydrator` | `min_video_duration_ms`, `quoted_video_duration_ms` |
| `HasMediaHydrator` | Media presence flag |
| `SubscriptionHydrator` | Subscription gating info |
| `GizmoduckCandidateHydrator` | Author followers count, screen name |
| `BlockedByHydrator` | "is the viewer blocked by the author?" |
| `FilteredTopicsHydrator` | `filtered_topic_ids`, `unfiltered_topic_ids` |
| `LanguageCodeHydrator` | Detected language |

**Critical invariant** (`candidate-pipeline/hydrator.rs:24`): each hydrator must return a vector with the **same candidates in the same order**. Errors on individual candidates are kept-but-skipped (the original is retained); length mismatches mark the whole result as error.

## Stage 4 — Pre-scoring filters (sequential)

Fourteen filters run in order, each receiving only the survivors from the previous filter (`phoenix_candidate_pipeline.rs:274-289`):

1. `DropDuplicatesFilter` — dedup by tweet_id.
2. `CoreDataHydrationFilter` — drop candidates that failed core hydration.
3. `AgeFilter` — drop posts older than `MAX_POST_AGE`.
4. `SelfTweetFilter` — drop the viewer's own posts.
5. `RetweetDeduplicationFilter` — collapse multiple retweets of the same source post.
6. `IneligibleSubscriptionFilter` — drop paywalled content the viewer can't access.
7. `PreviouslySeenPostsFilter` — first checks `query.seen_ids` directly, then bloom filters from impression system. (`previously_seen_posts_filter.rs:17-30`)
8. `PreviouslySeenPostsBackupFilter` — safety net for bloom-filter false negatives.
9. `PreviouslyServedPostsFilter` — drops `query.served_ids` (current session).
10. `MutedKeywordFilter` — drops posts containing any of the viewer's muted keywords.
11. `AuthorSocialgraphFilter` — drops posts from blocked/muted authors.
12. `VideoFilter` — drops videos if `query.exclude_videos` is set.
13. `TopicIdsFilter` — enforces `query.topic_ids` (include) and `query.excluded_topic_ids` (exclude).
14. `NewUserTopicIdsFilter` — additional new-user topic constraints.

The framework logs `(kept, removed)` per filter for observability.

## Stage 5 — Scoring (sequential, three scorers)

Three scorers run in order (`phoenix_candidate_pipeline.rs:299-300`):

### 5a. `PhoenixScorer` (home-mixer/scorers/phoenix_scorer.rs)

Calls the Phoenix prediction service over gRPC:

- Selects cluster (LAP7 vs FOU) based on user's action count vs `PhoenixRankerNewUserHistoryThreshold`. (`phoenix_scorer.rs:29-42`)
- Decider overrides can flip cluster routing: `override_qf_use_lap7`, `override_qf_use_fou`. (`phoenix_scorer.rs:45-54`)
- If `UseEgressSidecar` is on and the sidecar fails, **silently falls back** to primary Phoenix. (`phoenix_scorer.rs:86-98`)
- Fills each candidate's `phoenix_scores` with all 19 action probabilities + continuous heads.

### 5b. `RankingScorer` (home-mixer/scorers/ranking_scorer.rs)

Computes `weighted_score` as a linear combination:

```
weighted_score =
    favorite_score      × w_favorite
  + reply_score         × w_reply
  + retweet_score       × w_retweet
  + photo_expand_score  × w_photo_expand
  + click_score         × w_click
  + profile_click_score × w_profile_click
  + vqv_score           × w_vqv               (gated: only if min_video_duration_ms > MIN_VIDEO_DURATION_MS)
  + share_score         × w_share
  + share_via_dm_score  × w_share_via_dm
  + share_via_copy_link_score × w_share_via_copy_link
  + dwell_score         × w_dwell
  + quote_score         × w_quote
  + quoted_click_score  × w_quoted_click
  + quoted_vqv_score    × w_quoted_vqv        (gated similarly if enable_quoted_vqv_duration_check)
  + dwell_time          × w_cont_dwell_time
  + click_dwell_time    × w_cont_click_dwell_time
  + follow_author_score × w_follow_author
  + not_interested_score × w_not_interested   (NEGATIVE weight)
  + block_author_score  × w_block_author      (NEGATIVE weight)
  + mute_author_score   × w_mute_author       (NEGATIVE weight)
  + report_score        × w_report            (NEGATIVE weight)
  + not_dwelled_score   × w_not_dwelled       (NEGATIVE weight)
```

Then a normalization step (`ranking_scorer.rs:175-183`):

```
if combined_score < 0:
    score = (combined_score + negative_sum) / total_sum × NEGATIVE_SCORES_OFFSET
else:
    score = combined_score + NEGATIVE_SCORES_OFFSET
```

All weights are loaded from feature switches via `ScoringWeights::from_params()` (`ranking_scorer.rs:42-114`). They vary per-request.

### 5c. `VMRanker` / author diversity

Final adjustment of `score`:

```
multiplier(pos) = (1 - floor) * decay_factor^pos + floor
adjusted_score  = weighted_score * multiplier(author_count_so_far)
```

Where `pos` is "how many of this author's posts have already been placed ahead." (`home-mixer/scorers/author_diversity_scorer.rs:29-31`)

## Stage 6 — Selection

`TopKScoreSelector` (`home-mixer/selectors/top_k_score_selector.rs:9`):

1. Sort by `score` descending.
2. Truncate to `TOP_K_CANDIDATES_TO_SELECT` (feature-switch).
3. Return `SelectResult { selected, non_selected }` — non_selected is tracked for side effects.

## Stage 7 — Post-selection hydration (parallel)

Six hydrators enrich the top-K (`phoenix_candidate_pipeline.rs:304-315`):

| Hydrator | Adds |
|---|---|
| `VFCandidateHydrator` | Calls Visibility Filter service; sets `visibility_reason` (Action::Drop, Soften, etc.) |
| `AdsBrandSafetyHydrator` | Brand-safety verdict for the post (consulted by ads blender) |
| `AdsBrandSafetyVfHydrator` | VF-based brand-safety verdict variant |
| `TweetTypeMetricsHydrator` | Engagement-metrics bytes |
| `FollowingRepliedUsersHydrator` | Reply-chain user-overlap info |
| `MutualFollowJaccardHydrator` | Mutual-follow Jaccard similarity |

## Stage 8 — Post-selection filters (sequential)

Three filters run after hydration (`phoenix_candidate_pipeline.rs:317-321`):

1. `VFFilter` — drops candidates with `visibility_reason == Action::Drop`. (`vf_filter.rs:9-20`)
2. `AncillaryVFFilter` — filters ancillary content (quotes/retweets of dropped content).
3. `DedupConversationFilter` — deduplicates multiple branches of the same conversation thread, using `query.in_network_replies` from Stage 2.

The final result can still shrink below the top-K size at this stage.

## Stage 9 — For You blending (only on `ForYouCandidatePipeline`)

After the Phoenix pipeline produces ranked posts, the For You pipeline (`for_you_candidate_pipeline.rs:155-203`) layers in non-organic items via additional sources:

| Source | What |
|---|---|
| `ScoredPostsSource` | Phoenix-ranked posts (just produced) |
| `AdsSource` | Ads from AdIndex with insert positions |
| `WhoToFollowSource` | Recommended accounts to follow |
| `PromptsSource` | Carousel prompts (e.g., follow more topics) |
| `PushToHomeSource` | Pinned posts |

Then `BlenderSelector` (`home-mixer/selectors/blender_selector.rs:25-70`) interleaves:

1. Choose ads blender by feature switch (`AdsBlenderType`):
   - `"safe_gap"` → `SafeGapAdsBlender`
   - else → `PartitionOrganicAdsBlender`
2. Blend organic posts + ads.
3. Insert prompts at position 0.
4. Insert W2F at `WHO_TO_FOLLOW_POSITION`.
5. Pin push-to-home at position 0 (overrides prompts if present).

## Stage 10 — Side effects (background, parallel)

After the response is built, side effects are spawned on a background `tokio::spawn` and **don't block the response** (`candidate-pipeline/candidate_pipeline.rs:419-428`). Failures are silent.

### Phoenix pipeline side effects (6)

- `PhoenixExperimentsSideEffect` — logs experiment arms to Kafka (`PHOENIX_SCORES_TOPIC`).
- `RerankingKafkaSideEffect` — publishes reranking events.
- `RedisPostCandidateCacheSideEffect` — caches candidate list in Redis.
- `ScoredStatsSideEffect` — metrics.
- `MutualFollowStatsSideEffect` — mutual-follow stats.
- `PhoenixRequestCacheSideEffect` — cross-datacenter (ATLA + PDXA) request cache.

### For You pipeline side effects (8)

- `AdsInjectionLoggingSideEffect` — logs ad insertions to Kafka.
- `PublishSeenIdsToKafkaSideEffect` — publishes `query.served_ids` to Kafka.
- `ServedCandidatesKafkaSideEffect` — logs final served posts with served_type, engagement.
- `ClientEventsKafkaSideEffect` — publishes URT timeline operations.
- `ForYouResponseStatsSideEffect` — metrics histograms.
- `UpdatePastRequestTimestampsSideEffect` — rate-limit state.
- `UpdateServedHistorySideEffect` — writes served-history records (gated by `EnableUrtMigrationComponents`). (`update_served_history_side_effect.rs:34-74`)
- `TruncateServedHistorySideEffect` — TTL cleanup.

## Stage 11 — Response serialization

`for_you_server.rs:43-74` serializes the blended feed into the URT (Unified Response Timeline) protobuf format. Each entry has a `served_type`, position, and engagement metadata.

## Total path

```
gRPC in
 ↓
[QueryBuilder: Gizmoduck + feature switches]
 ↓
[15 query hydrators in parallel]
 ↓
[6 sources in parallel → flatten]
 ↓
[10 hydrators in parallel]
 ↓
[14 filters sequential]
 ↓
[PhoenixScorer (gRPC) → RankingScorer → AuthorDiversity]
 ↓
[TopKScoreSelector]
 ↓
[6 post-selection hydrators in parallel]
 ↓
[3 post-selection filters sequential]
 ↓
[5 For-You sources in parallel for non-organic items]
 ↓
[BlenderSelector: ads + W2F + prompts + push-to-home]
 ↓
[URT serialization]
 ↓
gRPC out ────► [8 side effects spawned in background]
```

## Latency-sensitive observations

- **Sources fan-out**: total source latency ≈ slowest source, not sum.
- **Filters sequential**: each filter adds latency proportional to candidate count (which monotonically decreases).
- **Phoenix scorer is the heaviest stage** — a single gRPC call to a JAX inference service, batched over all post-filter candidates.
- **Side effects are background** — they don't add to response latency.
- **VFCandidateHydrator** is a network call to the Visibility Filter service and is intentionally placed **after** selection so it only runs on top-K candidates, not on the full hydrated set.
