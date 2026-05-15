# 01 — Architecture

The full system view: what runs where, what talks to what, what the data shapes look like at each boundary.

## The five components and their relationship

```
                                ┌──────────────────────────────────────┐
                                │       FOR YOU FEED REQUEST           │
                                │  (gRPC: ForYouFeedService)           │
                                └──────────────────────────────────────┘
                                                  │
                                                  ▼
                ┌────────────────────────────────────────────────────────────────┐
                │                       HOME-MIXER (Rust)                       │
                │                                                                │
                │   ForYouCandidatePipeline                                      │
                │     ├─ ScoredPostsSource ──────────────┐                       │
                │     ├─ AdsSource                       │                       │
                │     ├─ WhoToFollowSource               │                       │
                │     ├─ PromptsSource                   │                       │
                │     └─ PushToHomeSource                │                       │
                │              │                         │                       │
                │              ▼                         ▼                       │
                │     BlenderSelector              (calls ScoredPostsService)    │
                │              │                         │                       │
                │              │                         ▼                       │
                │              │              PhoenixCandidatePipeline           │
                │              │                ├─ ThunderSource  ───────┐       │
                │              │                ├─ PhoenixSource ────────┤       │
                │              │                ├─ PhoenixMoESource      │       │
                │              │                ├─ PhoenixTopicsSource   │       │
                │              │                ├─ TweetMixerSource      │       │
                │              │                └─ CachedPostsSource     │       │
                │              │                         │               │       │
                │              │                         ▼               │       │
                │              │              [hydrate → filter → score] │       │
                │              │                  │   │              ▲   │       │
                │              │                  │   │              │   │       │
                │              │                  │   ▼              │   │       │
                │              │                  │ PhoenixScorer ───┘   │       │
                │              │                  │ (gRPC to Phoenix)    │       │
                │              │                  ▼                      │       │
                │              │              TopKScoreSelector          │       │
                │              │                  │                      │       │
                │              │                  ▼                      │       │
                │              │           [post-selection hydrate]      │       │
                │              │              ├─ VFCandidateHydrator     │       │
                │              │              │  (visibility/safety)     │       │
                │              │              └─ AdsBrandSafetyHydrator  │       │
                │              ▼                      │                  │       │
                │       (blend ads + organic)         │                  │       │
                │              │                      │                  │       │
                │              ▼                      ▼                  ▼       │
                │   ResponseSerializer ────► Side effects (Kafka, Redis, etc.)   │
                └────────────────────────────────────────────────────────────────┘
                              │                                          │
                              ▼                                          ▼
                       URT TimelineResponse                  ┌──────────────────────┐
                              (to client)                    │   THUNDER (Rust)     │
                                                             │  in-memory store     │
                                                             │  fed by Kafka        │
                                                             └──────────────────────┘

  ┌──────────────────────┐                  ┌─────────────────────────────────────┐
  │   PHOENIX (Python)   │                  │            GROX (Python)            │
  │   JAX inference      │                  │   spam, PTOS, embed, summarize      │
  │   retrieval + rank   │                  │   Kafka in → Strato/Kafka out       │
  └──────────────────────┘                  └─────────────────────────────────────┘
```

## Service surfaces

### Home-Mixer

- **gRPC `ForYouFeedService::get_for_you_feed`** — entry point for client-facing For You feed requests. Returns a URT (Unified Response Timeline) protobuf with posts, ads, W2F modules, and prompts interleaved. (`home-mixer/server.rs:273-298`, `home-mixer/for_you_server.rs:28-74`)
- **gRPC `ScoredPostsService::get_scored_posts`** — entry point for raw scored posts. Used internally by ForYou and externally by other surfaces. Returns ranked `ScoredPost` protobufs. (`home-mixer/scored_posts_server.rs:41-113`)

### Thunder

- **gRPC `GetInNetworkPosts`** — single RPC that returns `Vec<LightPost>` for a given `(user_id, following_user_ids[], exclude_tweet_ids[], max_results, is_video_request)`. (`thunder/thunder_service.rs:154-330`)
- Server uses Zstd compression. (`thunder/thunder_service.rs:58-59`)
- Backpressure via Semaphore — rejects with `RESOURCE_EXHAUSTED` when overloaded. (`thunder_service.rs:160-170`)

### Phoenix

- Not a Rust service in this repo — Phoenix is invoked over gRPC by `PhoenixScorer` (`home-mixer/scorers/phoenix_scorer.rs`). The repo contains the **JAX model code + inference scripts**, not the serving binary.
- `phoenix/run_pipeline.py` is the runnable demo that loads checkpoints and runs retrieval → ranking end-to-end against the bundled `sports_corpus.npz`.

### Grox

- Not a request/response service. It's a **multi-process streaming pipeline**: Dispatcher process consumes Kafka, Engine processes execute plans, results are written to Strato (KV store) and other Kafka topics. (`grox/main.py:21-50`, `grox/dispatcher.py:39-370`, `grox/engine.py:25-137`)

## Data flow at the boundaries

### Phoenix → home-mixer

The `PhoenixScorer` calls the Phoenix prediction service via gRPC and fills these fields on each `PostCandidate.phoenix_scores`:

```
favorite_score, reply_score, retweet_score, photo_expand_score, click_score,
profile_click_score, vqv_score, share_score, share_via_dm_score,
share_via_copy_link_score, dwell_score, quote_score, quoted_click_score,
follow_author_score, not_interested_score, block_author_score,
mute_author_score, report_score, not_dwelled_score, dwell_time,
click_dwell_time, quoted_vqv_score
```

These are floats in `[0, 1]` (sigmoid of the model's logits) plus continuous predictions like `dwell_time`. See `phoenix/runners.py:233-264` for the canonical mapping from output indices to action names.

### Grox → home-mixer

Grox writes outputs to two sinks (`grox/tasks/task_write_*.py`):

1. **Strato (Manhattan-style KV store)** — `StratoPostMultimodalEmbeddingMhSearchAi`, `StratoSafetyPostAnnotationsResultMh`. Read by home-mixer hydrators at request time for embeddings + safety annotations.
2. **Kafka topics** — `MultiModalEmbeddingTopic`, `SafetyPostAnnotationsResultKafka`. Consumed by downstream services and other home-mixer hydrators.

Additionally, **UPA (Unified Post Annotation)** is called via `grokPtosActionWithLabels` and `grokPtosDeleteLabels` to apply/revoke platform-level action labels (demote, block, etc.) on policy violations. (`grox/tasks/task_write_safety_post_annotations_result_sink.py:210-328`)

### Thunder → home-mixer

`ThunderSource` in home-mixer (`home-mixer/sources/thunder_source.rs`) calls Thunder's `GetInNetworkPosts` RPC, passing the user's `following_user_ids`, `seen_ids`, `max_results`. Thunder returns unsorted `LightPost`s; home-mixer's `score_recent()` sorts them by `created_at` descending. Thunder also extracts reply relationships into `query.in_network_replies` for later conversation dedup.

## The `PostCandidate` model

The canonical struct flowing through the home-mixer pipeline is `PostCandidate` (`home-mixer/models/candidate.rs`). It accumulates fields as it passes through hydrators:

```
PostCandidate {
  // Identifiers (set at source)
  tweet_id, author_id, tweet_text, in_reply_to_tweet_id,
  retweeted_tweet_id, retweeted_user_id, quoted_tweet_id, quoted_user_id

  // Scores (set during scoring)
  phoenix_scores: PhoenixScores,       // 19+ floats from Phoenix
  weighted_score: Option<f64>,         // set by RankingScorer
  score: Option<f64>,                  // set by author diversity / VMRanker
                                       // ← this is what TopKScoreSelector sorts by

  // Incremental hydration
  in_network: Option<bool>,                       // InNetworkCandidateHydrator
  min_video_duration_ms, quoted_video_duration_ms, // CoreDataCandidateHydrator
  author_followers_count, author_screen_name,      // GizmoduckCandidateHydrator
  visibility_reason: Option<...>,                  // VFCandidateHydrator (post-sel)
  brand_safety_verdict: Option<...>,               // AdsBrandSafetyHydrator (post-sel)
  tweet_type_metrics: Option<Vec<u8>>,             // TweetTypeMetricsHydrator (post-sel)
  following_replied_user_ids: ...,                 // FollowingRepliedUsersHydrator (post-sel)
  mutual_follow_jaccard: Option<f64>,              // MutualFollowJaccardHydrator (post-sel)

  // Safety / filtering
  safety_labels: Vec<SafetyLabelInfo>,
  filtered_topic_ids, unfiltered_topic_ids,
  language_code,
  ...
}
```

## Dual-pipeline architecture

Home-mixer assembles **two pipelines** built on the same `candidate-pipeline` framework:

1. **`PhoenixCandidatePipeline`** (`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`) — fetches and ranks posts. Produces `Vec<PostCandidate>` sorted by `score`.
2. **`ForYouCandidatePipeline`** (`home-mixer/candidate_pipeline/for_you_candidate_pipeline.rs`) — wraps Phoenix as a source, adds `AdsSource` / `WhoToFollowSource` / `PromptsSource` / `PushToHomeSource`, then blends via `BlenderSelector`. Produces the final URT timeline.

The reason this split exists: other surfaces (Explore, Notifications, embedded recommendations) can call `ScoredPostsService` to reuse just the Phoenix scoring pipeline without the For You-specific blending.

## What's NOT in the repo

- The Phoenix serving binary (only the JAX model code).
- Gizmoduck (user-profile service), Strato (KV store), TES (tweet entity service), Manhattan, UAA (Unified User Actions stream), VF (visibility filter) service — all called via gRPC clients but their server code is elsewhere.
- The full production Phoenix checkpoint — only a frozen 128-dim, 4-layer mini model.
- The trained ads / W2F / prompts ranking models — only the integration points.
- The `tweetypie`, `recos-injector`, `simclusters-ann`, `representation-manager` etc. services that existed in the 2023 release.
