# 08 — Glossary

X-specific jargon and acronyms that appear in the codebase or documentation. Each entry: what it stands for, what it does, where it appears.

## Models & ML

**Phoenix**
The two-stage recommendation model: two-tower retrieval + Grok-based transformer ranking. Replaces the 2023 Heavy Ranker, SimClusters, and TwHIN combo. Lives in `phoenix/`.

**Grok / Grok-1**
xAI's open-source LLM family. The Phoenix transformer architecture is **ported from Grok-1** (`xai-org/grok-1`) with recsys-specific modifications (candidate-isolation attention mask, hash-based input embeddings, multi-action heads).

**Grok VLM**
Grok Vision Language Model. Used by Grox classifiers (spam, safety_ptos) for content-understanding inference.

**MoE**
Mixture of Experts. Phoenix has a MoE variant (`PhoenixMOESource` in home-mixer) used as an alternative candidate source. The published mini-checkpoint is dense (no MoE).

**Two-tower**
Retrieval architecture with a user tower (encodes user history → embedding) and a candidate tower (encodes posts → embedding). Top-K via dot product. The user tower is heavy (transformer); the candidate tower is light (MLP), enabling offline pre-computation of corpus embeddings.

**Candidate isolation**
The attention-masking trick (`phoenix/grok.py:39-71`) that prevents candidate tokens from attending to each other. Each candidate's score depends only on user context, not on which other candidates are in the batch.

**VQV**
Video Quality View — predicted probability that a user watches a video meaningfully (not just scrolls past). Gated by min video duration in ranking. One of the 19 action heads.

**Heavy Ranker**
The 2023 ranking model (MaskNet-based). **Replaced by Phoenix** in 2026. Mentioning Heavy Ranker in a 2026 context indicates you're thinking of the old code.

**SimClusters / TwHIN / real-graph / tweepcred**
2023 algorithm components for community detection, knowledge-graph embeddings, follow-likelihood prediction, and PageRank-based user reputation. **All gone** in 2026.

## Services & infrastructure

**Home Mixer**
The main orchestration service (Rust). Builds the For You feed by running candidate sourcing → hydration → filtering → scoring → selection → blending. Exposes `ForYouFeedService` and `ScoredPostsService` gRPC endpoints.

**Thunder**
In-memory store of recent posts indexed by author (Rust). Kafka-fed. Serves "in-network" candidates to home-mixer with sub-ms latency.

**Grox**
Real-time content-understanding pipeline (Python, multi-process). Runs spam/safety classifiers and multimodal embedders against incoming posts. Writes to Strato + Kafka + UPA.

**Gizmoduck**
X's user-profile service. Called by home-mixer to fetch viewer roles, subscription level, muted keywords, follower count, etc.

**TES**
Tweet Entity Service. Provides core post data — text, media, author info. Called by home-mixer's `CoreDataCandidateHydrator`.

**UAA**
Unified User Actions. A real-time stream of user engagement events. Home-mixer's `ScoringSequenceQueryHydrator` and `RetrievalSequenceQueryHydrator` consume from here.

**USS**
User Signal Service. Centralized platform for explicit (likes, replies) and implicit (profile visits, post clicks) user signals. Referenced in 2023 docs; the 2026 stack reads sequences directly via UAA.

**VF**
Visibility Filter service. Called by `VFCandidateHydrator` (post-selection) to determine if a post should be dropped due to safety/policy violations. Returns an `Action::{Drop, Soften, Show, ...}` verdict.

**Strato / Manhattan**
X's key-value storage. Grox writes embeddings and safety annotations here for home-mixer to read at request time. (Strato is the read/write façade; Manhattan is the underlying store.)

**UPA**
Unified Post Annotation. The system that applies platform-level action labels (demote, block, etc.) to posts. Grox's `grokPtosActionWithLabels` writes labels; `grokPtosDeleteLabels` revokes them.

**AdIndex**
X's ads serving index. `AdsSource` in home-mixer fetches candidates from here.

**AdsBlender**
The component that interleaves ads with organic posts. Two implementations: `SafeGapAdsBlender` and `PartitionOrganicAdsBlender`.

**W2F**
"Who to Follow" — the recommendation module suggesting accounts to follow. Inserted into the feed at a configurable position by `BlenderSelector`.

**TWML**
A legacy ML framework built on TensorFlow v1, used in the 2023 algorithm. **Not present** in the 2026 release.

**Navi**
The 2023 Rust-based ML model server. **Not present** in the 2026 release; Phoenix inference happens in JAX.

**Tweetypie**
The 2023 core post-storage service. **Replaced** by TES in the 2026 stack.

## Datacenters & clusters

**LAP7, FOU**
Datacenter or inference-cluster identifiers used in Phoenix scoring/retrieval routing. `PhoenixScorer` and `PhoenixSource` can choose between them based on:
- New-user thresholds (action-count-based).
- Decider flags (`override_qf_use_lap7`, `override_qf_use_fou`, `enable_phoenix_retrieval_lap7_to_fou`).

**ATLA, PDXA**
Atlanta and Portland datacenter identifiers. Used by `PhoenixRequestCacheSideEffect` to cache requests across multiple DCs.

**Decider**
X's runtime-flag system (separate from feature switches). Booleans evaluated per-request that can override routing without redeployment. Used heavily in Phoenix source/scorer.

**Feature switch**
X's parameter-config system. Provides per-request, per-user, per-DC values for things like action weights, K-cutoffs, and timeouts. Distinct from Deciders (deciders are booleans; feature switches are typed parameters).

## Data & file formats

**URT**
Unified Response Timeline. The protobuf format used to serialize the final For You feed (posts + ads + W2F + prompts + dividers). Returned by `ForYouFeedService`.

**LightPost**
A compact post-metadata struct used by Thunder for in-memory storage (`thunder/posts/post_store.rs`).

**TinyPost**
An even more compact reference (just `{post_id, created_at}`) used in Thunder's per-author timelines. Saves memory when a post lives in multiple per-author lists.

**ScoredPost**
The protobuf returned by `ScoredPostsService` — a `PostCandidate` serialized for over-the-wire transport.

**`sports_corpus.npz`**
The bundled retrieval-corpus demo file. ~537K sports-related post IDs from a 6-hour window, with pre-computed L2-normalized embeddings. NOT a production corpus.

**RoPE**
Rotary Position Embedding. The positional encoding scheme used in Phoenix's transformer (base_exponent=10000). Supports optional right-anchoring.

**RMSNorm**
Root Mean Square layer normalization. Used pre-attention and pre-FFN in Phoenix.

**SwiGLU**
Swish-Gated Linear Unit — the FFN activation pattern used in Phoenix (GELU-gated parallel projections).

**GQA**
Grouped Query Attention. The mini Phoenix uses 4 query heads + 4 KV heads (ratio 1:1, effectively MHA). Production may use a different GQA ratio.

**LCG**
Linear Congruential Generator. The hash function family used for entity-id hashing in Phoenix: `(id * scale + bias) % modulus`.

**SafeGap**
The algorithm in `SafeGapAdsBlender` that identifies feed positions where ads can be placed without proximity to sensitive content.

## Plans & tasks (Grox)

**Plan**
A DAG of tasks executed by Grox's PlanMaster. Each plan declares `TASKS`, `TASK_DEPENDENCIES`, and `REQUIRED_ELIGIBILITY`. 9 plans run in parallel: PlanSpamComment, PlanPostSafety, PlanSafetyPtos, PlanPostEmbeddingV5 (×4 variants), PlanReplyRanking, PlanInitialBanger.

**Task**
A unit of work in Grox. Subclasses: `TaskWithPost`, `TaskWithUser`, `TaskWithUserContext`, `TaskWithContentAnalysis`. Supports skip rules, retries, disable rules.

**Deluxe**
A variant of safety-PTOS plans that uses extended-thinking / a different model. Triggered by `TaskGeneratorType.SAFETY_PTOS_DELUXE`. Lets X A/B-test new safety models on live traffic.

**Eligibility**
A flag-set in `TaskPayload` indicating which Grox plans/tasks should run on this message. Injected by the dispatcher based on the source Kafka stream.

**Banger**
Grox's term for a post predicted to be high-engagement. `PlanInitialBanger` runs a classifier on candidate posts to mark them.

**PTOS**
"Platform Trust and Safety" (X's term). The category of safety classifiers in Grox: ViolentMedia, AdultContent, Spam, IllegalAndRegulatedBehaviors, HateOrAbuse, ViolentSpeech, SuicideOrSelfHarm.

**EAPI**
"Enterprise API" — used in summarizer/sampler names like `EapiSummarizer`, `EapiSampler`. The samplers route to Claude (and other external models) for reasoning-heavy tasks.

**ASR**
Automatic Speech Recognition. `task_asr.py` transcribes audio from videos so the transcript can be embedded alongside post text.

## Acronyms in `phoenix_scores` field names

| Field | Means |
|---|---|
| `vqv_score` | Video Quality View probability |
| `quoted_vqv_score` | VQV for the quoted post's video |
| `dwell_score` | Discrete dwell event (yes/no) probability |
| `dwell_time` | Continuous dwell time prediction (seconds) |
| `not_dwelled_score` | Probability user does NOT dwell — used as negative signal |
| `cont_dwell_time` | Same as `dwell_time` (alternate name in weights) |
| `cont_click_dwell_time` | Continuous post-click dwell time prediction |

## Cross-references

- For more on **Phoenix** and **candidate isolation**: `references/02-phoenix-model.md`.
- For **LAP7/FOU/cluster routing**: `references/03-request-lifecycle.md` (Stage 2 and Stage 5a).
- For **weights** and **author diversity**: `references/05-ranking-and-blending.md`.
- For **VF, Strato, UPA, Gizmoduck** integration points: `references/01-architecture.md`.
- For **PTOS, Banger, Grox plans**: `references/06-content-understanding.md`.
