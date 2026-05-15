# 06 — Content Understanding: Thunder, Grox, and Filtering

The supporting infrastructure that gives the algorithm its **inputs** (in-network posts, embeddings, safety labels) and its **constraints** (visibility filters, blocked content).

## Thunder — in-network post store

**Location:** `thunder/`
**Language:** Rust
**Role:** Sub-millisecond lookup of "what did the people I follow post recently?"

### Why a dedicated service?

Hitting the post-storage tier (Manhattan / TES) on every For You request to find recent posts from each followee would be prohibitively expensive — most users follow hundreds to thousands of accounts. Thunder keeps all "recent" posts (default 2-day window) in memory, partitioned by author, so the in-network candidate set can be assembled with a per-author O(K) scan.

### Storage model

Thunder maintains **three separate per-author timelines** plus one central post store (`thunder/posts/post_store.rs:40-47`):

```rust
posts:                       DashMap<i64, LightPost>           // post_id → full data
original_posts_by_user:      DashMap<i64, VecDeque<TinyPost>>  // author_id → originals
secondary_posts_by_user:     DashMap<i64, VecDeque<TinyPost>>  // author_id → replies + retweets
video_posts_by_user:         DashMap<i64, VecDeque<TinyPost>>  // author_id → video posts
deleted_posts:               DashMap<i64, ...>                  // soft-delete tracking
```

`TinyPost { post_id, created_at }` is a 16-byte reference; full `LightPost` data lives only once in `posts`. This avoids ~Nx memory duplication when a post appears in multiple per-author timelines.

### Concurrency

- `DashMap` provides lock-free reads with per-shard locking on writes.
- Read path uses `.value().clone()` immediately upon lookup to release the entry lock before any further work (`post_store.rs:276`).
- Inserts use the `entry()` API to minimize contention (`post_store.rs:138-145`).

### Kafka ingestion

Two topics consumed in parallel (`thunder/kafka/`):

| Topic | Format | Listener |
|---|---|---|
| V2 (primary) | proto `InNetworkEvent` → `TweetCreateEvent` / `TweetDeleteEvent` | `tweet_events_listener_v2.rs:23-66` |
| Legacy | Thrift `TweetEvent` | `tweet_events_listener.rs:92-122` |

Per-event fields used: `post_id, author_id, created_at, in_reply_to_post_id, is_retweet, source_post_id, source_user_id, has_video, conversation_id`.

**Idempotency:** Create events check if `post_id` already exists in `posts`; if so, the timeline insertion is skipped (`post_store.rs:127-131`). Delete events soft-delete in `deleted_posts`, then `finalize_init()` hard-removes from `posts` and from per-author timelines (`post_store.rs:108-110`). This ordering ensures deletes win against any racing create.

**Throughput control:** A semaphore limits batch processing to 3 concurrent during serving (`tweet_events_listener_v2.rs:56`) so Kafka consumption doesn't starve the request handlers.

### Retention / trimming

`start_auto_trim(2)` spawns a task that runs every 2 minutes (`thunder/main.rs:85`). `trim_old_posts()` (`post_store.rs:409-476`):

- Iterates all three per-author timelines.
- Drops entries with `current_time - created_at > retention_seconds` (default 2 days).
- VecDeques are front-popped (oldest first, since insertion maintains temporal order).
- `shrink_to_fit()` when capacity >> size.
- Empty user entries are removed to prevent memory leaks.

### Read path

`GetInNetworkPosts(user_id, following_user_ids, exclude_tweet_ids, max_results, is_video_request)`:

1. For each `followee_id` in `following_user_ids`:
   - Fetch up to `MAX_ORIGINAL_POSTS_PER_AUTHOR` from `original_posts_by_user`.
   - Fetch up to `MAX_REPLY_POSTS_PER_AUTHOR` from `secondary_posts_by_user`.
2. For each `TinyPost`, look up the full `LightPost` from the central `posts` map.
3. Filter out:
   - Posts in `deleted_posts`.
   - Posts in `exclude_tweet_ids` (caller-provided seen list).
   - **Self-retweets** (`source_user_id == request_user_id` for retweets — `post_store.rs:288-289`).
   - **Replies that don't fit the conversation rules**: for secondary posts, only include replies-to-followed-users or replies-to-replies in the same conversation (`post_store.rs:291-315`).
4. Return unsorted; home-mixer's `score_recent()` sorts by `created_at` descending.

### Performance properties

- All reads in memory; no DB calls in the hot path.
- `spawn_blocking` for iteration + filtering so the async runtime isn't stalled (`thunder_service.rs:279-312`).
- Semaphore-based admission control rejects with `RESOURCE_EXHAUSTED` rather than queueing (`thunder_service.rs:160-170`).
- `In-flight Drop guard` decrements an active-request counter so metrics survive panics (`thunder_service.rs:174-180`).

### Non-obvious behaviors

- **Video inference from retweets** — a retweet of a video gets `has_video=true` on the retweet's record (`post_store.rs:147-156`). Lets video-only requests grab retweeted videos.
- **Per-stage metrics** — post-statistics (freshness, reply ratio, unique authors) emitted at both "retrieved" (raw from store) and "scored" (after recency sort) for A/B comparison (`thunder_service.rs:304, 309`).
- **Init signaling** — main thread waits for all Kafka partitions to emit a signal that their lag dropped below one batch (`thunder/main.rs:71-76`). Service only starts serving after caught up.

## Grox — content-understanding pipeline

**Location:** `grox/`
**Language:** Python
**Role:** Real-time, multi-process Kafka pipeline for spam detection, PTOS safety enforcement, multimodal embeddings, summarization, and reply ranking.

### Core abstractions

```
ScheduleContext
   ├─ task_queue       (multiprocessing Queue)
   ├─ resp_queue       (multiprocessing Queue)
   ├─ shutdown_event
   └─ ...

Dispatcher (own process)
   ├─ consumes Kafka via StreamGenerator(s)
   ├─ multiplexes 13+ stream types via PriorityTaskGenerator
   ├─ enqueues TaskPayload onto task_queue
   ├─ enforces max-in-flight
   └─ polls resp_queue, retries with backoff

Engine (own process)
   ├─ pulls TaskPayload from task_queue
   ├─ calls PlanMaster.exec(payload)
   ├─ writes TaskResult onto resp_queue
   └─ measures task latency, retries with timeouts

PlanMaster
   ├─ runs all 9 plans in parallel
   ├─ merges TaskResults
   └─ returns single combined result

Plan
   ├─ TASKS: dict[str, type[Task]]
   ├─ TASK_DEPENDENCIES: dict[str, set[str]]
   ├─ REQUIRED_ELIGIBILITY: set[TaskEligibility]
   └─ exec(ctx) — async DAG resolution via futures

Task
   ├─ _exec(ctx: TaskContext)
   ├─ TaskWithPost / TaskWithUser / TaskWithUserContext / TaskWithContentAnalysis
   └─ supports skip rules, retries, disable rules
```

(`grox/engine.py:25-137`, `grox/dispatcher.py:39-370`, `grox/plans/plan.py:16-106`, `grox/tasks/task.py:27-150`)

### Execution model

Grox is a **real-time, multi-process streaming pipeline**, not a service:

```
[Kafka topic A] ─┐
[Kafka topic B] ─┼─► StreamGenerator(s) ─► PriorityTaskGenerator
[Kafka topic C] ─┘                              │
                                                ▼
                                        TaskPayload (eligibility flags set per stream)
                                                │
                                                ▼
                                          Dispatcher
                                                │ (max-in-flight throttled)
                                                ▼
                                          task_queue
                                                │
                                                ▼
                                            Engine
                                                │
                                                ▼
                                       PlanMaster.exec(payload)
                                                │
                                                ▼
                  ┌──────────────────────────────────────────────────────────┐
                  │  9 plans run in parallel (via asyncio.gather):           │
                  │  PlanSpamComment, PlanPostSafety, PlanSafetyPtos,        │
                  │  PlanPostEmbeddingV5(*4 variants), PlanReplyRanking,     │
                  │  PlanInitialBanger, ...                                  │
                  │                                                          │
                  │  Each plan = task DAG with futures for dependencies      │
                  └──────────────────────────────────────────────────────────┘
                                                │
                                                ▼
                                          TaskResult
                                                │
                                                ▼
                                          resp_queue
                                                │
                                                ▼
                                        Dispatcher loop → ack / retry
                                                │
                                                ▼
                       ┌────────────────────────┴───────────────────────────┐
                       ▼                                                    ▼
                  Strato (KV)                                           Kafka topics
                  - StratoPostMultimodalEmbeddingMhSearchAi               (consumed by
                  - StratoSafetyPostAnnotationsResultMh                    home-mixer
                                                                          hydrators and
                                                                          downstream svcs)
                       │
                       ▼
                  UPA (Unified Post Annotation)
                  - grokPtosActionWithLabels (apply demote/block labels)
                  - grokPtosDeleteLabels (revoke labels when fixed)
```

### Plan vs Task contract

A `Plan` is a DAG of `Task`s with declared dependencies. Example: `PlanPostSafety` (`grox/plans/plan_post_safety.py:11-32`):

```
filter → rate_limit → media_hydration → safety_screen → upa_action → manhattan_write
```

Dependency resolution (`plan.py:36-92`):

```python
dependencies = {task: loop.create_future() for task in self.TASKS}
await asyncio.gather(*[self._execute_task(t, ctx, dependencies) for t in self.TASKS])
```

Each task awaits the futures for its declared dependencies. If any dependency returns `TaskResultCategory.SKIPPED`, downstream tasks are skipped too.

A `Task` can:
- Read from `TaskContext` (eligibility flags, post, user, accumulated results).
- Mutate shared `TaskContext` (content_categories, embedding, summary, etc.).
- Throw (triggers plan failure) or return `TaskResultCategory.SUCCESS / SKIPPED`.

### The classifiers

#### Spam (`grox/classifiers/content/spam.py:25-104`)

- **Input:** Post (text, author, media, thread context).
- **Model:** Grok VLM via `VisionSampler` API.
- **Prompt:** `SpamSystemLowFollower` template + `ThreadRenderer` (renders the post and its conversation as a chat-style message sequence).
- **Output:** JSON `{decision: "spam"|"not_spam", reasoning: "..."}` parsed into a `ContentCategoryResult` with binary score (1.0 spam / 0.0 not).
- `temperature=0` for deterministic output.

#### Safety PTOS Category (`grox/classifiers/content/safety_ptos.py:57-138`)

- **Two-phase design:** "standard" vs "deluxe" (deluxe uses extended thinking / different model).
- **Input:** Post + user context, optionally with transcript (from ASR task).
- **Model:** `VLM_SAFETY` or `VLM_PRIMARY_CRITICAL`.
- **Output:** JSON `SafetyPostAnnotations { violatedPolicies: [{category, score, safetyPolicy: {policyType, details}}, ...] }`.

Categories: `ViolentMedia, AdultContent, Spam, IllegalAndRegulatedBehaviors, HateOrAbuse, ViolentSpeech, SuicideOrSelfHarm`.

#### Safety PTOS Policy (`grox/classifiers/content/safety_ptos.py:140-289`)

- **Conditional reasoning:** Deluxe mode invokes Claude Sonnet (via `EapiSampler`) for critical categories (AdultContent, ViolentMedia).
- **Policy-specific prompts:** different prompts loaded per violation type.
- **Fallback:** If reasoning fails, reverts to the standard VLM path.

### The multimodal embedder (V5)

`grox/embedder/multimodal_post_embedder_v5.py:18-120`:

- **Modalities:** text + images. Detects video presence but passes transcript (from ASR) instead of embedding video directly.
- **Embedding dim:** 1024 (after `TRUNCATE_DIM` truncation + L2 normalization).
- **Model endpoint:** `ModelName.RECSYS_EMBED_V5` via `XaiEmbeddingClientHttp.embed_http`.
- **Rendering:** `V5EmbedPostRenderer` converts post → `(text, image_bytes[])`.
- **Async:** full encode is awaited.
- **Output:** `(document: list[(modality_type, data)], embedding: list[float])`.

Used in `PlanPostEmbeddingV5`: rate limit → media hydration → ASR → embedding → sink write.

### Sinks

#### Manhattan / Strato (KV)

- `StratoPostMultimodalEmbeddingMhSearchAi.put(post_id, model_version, TweetEmbedding)` (`task_write_mm_embedding_sink.py:47-91`)
- `StratoSafetyPostAnnotationsResultMh.put(post_id, SafetyPostAnnotationsResult)` (`task_write_safety_post_annotations_result_sink.py:330`)

Home-mixer hydrators read these for embedding + safety lookups at request time.

#### Kafka

- `MultiModalEmbeddingTopic` — for downstream streaming consumers.
- `SafetyPostAnnotationsResultKafka` — for downstream services.

#### UPA (Unified Post Annotation)

- `grokPtosActionWithLabels` applies platform-level action labels (demote, block, etc.) based on detected violations.
- `grokPtosDeleteLabels` revokes labels if the post is no longer in violation (e.g., classifier changed mind, post was edited).

### Non-obvious Grox patterns

1. **Eligibility-based routing** — each task is invoked only if the corresponding `TaskEligibility` flag is set in the payload. Dispatcher injects eligibilities based on the source stream type. Same post can run spam, embedding, safety, summarization plans **in parallel** if all flags are set.

2. **Disable rules** — separate from eligibility. `DisableTaskForNonPtosProd` prevents safety writes in non-prod envs. Operator control without redeployment.

3. **Deluxe mode** — A/B variant of safety classifiers. `TaskGeneratorType.SAFETY_PTOS_DELUXE` triggers the deluxe path with a heavier model. Lets you A/B-test new safety models on real traffic.

4. **Lazy classifier init** — classifier instances are class variables; created once at first task execution, reused across invocations. Avoids paying VLM startup cost per task.

5. **Safemodel fallback** — `task_write_safety_post_annotations_result_sink.py:239-304` falls back to a legacy SafeModel sex-and-nudity classifier if PTOS didn't flag the post as NSFW. Belt-and-suspenders for critical violations.

6. **Kafka offset management** — `KafkaLoader.ack()` is a no-op (`kafka_loader.py:92-93`). Offsets are managed by the Kafka consumer group on the broker. Rebalancing prevents duplicate processing.

## The 14 pre-scoring filters in detail

Pulled from `home-mixer/filters/`:

| # | Filter | What it drops |
|---|---|---|
| 1 | `DropDuplicatesFilter` | Candidates with duplicate `tweet_id`. |
| 2 | `CoreDataHydrationFilter` | Candidates whose TES core-data hydration failed (no text, no author info). |
| 3 | `AgeFilter` | Posts older than `MAX_POST_AGE` (feature switch). |
| 4 | `SelfTweetFilter` | The viewer's own posts. |
| 5 | `RetweetDeduplicationFilter` | Multiple retweets of the same source post; keeps highest-ranked. |
| 6 | `IneligibleSubscriptionFilter` | Subscriber-only posts the viewer can't access. |
| 7 | `PreviouslySeenPostsFilter` | Posts in `query.seen_ids` directly + posts matching impression bloom filters. |
| 8 | `PreviouslySeenPostsBackupFilter` | Safety net for bloom-filter false negatives. |
| 9 | `PreviouslyServedPostsFilter` | Posts already served in the current session (`query.served_ids`). |
| 10 | `MutedKeywordFilter` | Posts containing any of the viewer's muted keywords. |
| 11 | `AuthorSocialgraphFilter` | Posts from blocked or muted authors. |
| 12 | `VideoFilter` | Videos, if `query.exclude_videos` is set. |
| 13 | `TopicIdsFilter` | Enforces `query.topic_ids` (include) and `query.excluded_topic_ids` (exclude). |
| 14 | `NewUserTopicIdsFilter` | Additional topic constraints for new accounts. |

## The 3 post-selection filters

| # | Filter | What it drops |
|---|---|---|
| 1 | `VFFilter` | Candidates with `visibility_reason == Action::Drop` (set by VFCandidateHydrator). Removes deleted/spam/gore/policy-violating content. |
| 2 | `AncillaryVFFilter` | Quotes/retweets of dropped content. |
| 3 | `DedupConversationFilter` | Multiple posts in the same conversation tree; keeps the canonical reply via `query.in_network_replies` (populated by ThunderSource). |

## How Thunder + Grox + filters interact

```
ThunderSource ───► hydrate ──┐
                              ├──► filter ───► score ───► select ───┐
PhoenixSource ───► hydrate ──┘                                       │
                                                                     ▼
                                                          [post-selection]
                                                                     │
                                                                     ▼
                                                  VFCandidateHydrator (consults
                                                  Grox-written safety annotations
                                                  in Strato + VF service)
                                                                     │
                                                                     ▼
                                                          VFFilter drops disallowed
                                                                     │
                                                                     ▼
                                                          DedupConversationFilter
                                                          uses Thunder's
                                                          in_network_replies
                                                                     │
                                                                     ▼
                                                          AdsBrandSafetyHydrator
                                                          (consults brand-safety
                                                          verdicts from Grox)
                                                                     │
                                                                     ▼
                                                          SafeGapAdsBlender
                                                          consults brand-safety
                                                          to find "safe gaps"
```

Grox is **upstream of every safety / brand-safety / embedding hydration** in home-mixer. Its outputs are queried via Strato + Kafka and the VF service.
