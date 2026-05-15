# 04 — The `candidate-pipeline` Framework

The Rust trait system that composes every recommendation pipeline in home-mixer. Six core traits, an opinionated execution model, and a few sharp edges.

## The six traits

All traits are generic over `Q: PipelineQuery, C: PipelineCandidate`. `PipelineQuery` and `PipelineCandidate` are blanket impls over any `Clone + Send + Sync + 'static` type (`candidate-pipeline/candidate_pipeline.rs:59-65`).

### `Source<Q, C>` (`candidate-pipeline/source.rs:9-39`)

```rust
#[async_trait]
pub trait Source<Q, C>: Any + Send + Sync {
    fn enable(&self, _query: &Q) -> bool { true }
    async fn run(&self, query: &Q) -> Result<Vec<C>, String> { self.source(query).await }
    async fn source(&self, query: &Q) -> Result<Vec<C>, String>;
    fn name(&self) -> &'static str;
}
```

Sources are candidate generators. They return `Result<Vec<C>, String>`. **Failed sources contribute zero candidates; the pipeline continues** (`candidate_pipeline.rs:266`).

### `QueryHydrator<Q>` (`candidate-pipeline/query_hydrator.rs:9-41`)

```rust
#[async_trait]
pub trait QueryHydrator<Q>: Any + Send + Sync {
    fn enable(&self, _query: &Q) -> bool { true }
    async fn run(&self, query: &Q) -> Result<Q, String> { self.hydrate(query).await }
    async fn hydrate(&self, query: &Q) -> Result<Q, String>;
    fn update(&self, query: &mut Q, hydrated: Q);
    fn name(&self) -> &'static str;
}
```

Each hydrator returns a **full Q**, but only **its own fields** are merged back via `update()`. This is a convention, not enforced — hydrator implementers must populate only their fields.

Query hydration has **two passes**: independent hydrators run first (parallel), then dependent hydrators that may need results from the first pass (parallel within the second pass). (`candidate_pipeline.rs:202-218, 229-248`)

### `Hydrator<Q, C>` (`candidate-pipeline/hydrator.rs:11-67`)

```rust
#[async_trait]
pub trait Hydrator<Q, C>: Any + Send + Sync {
    fn enable(&self, _query: &Q) -> bool { true }
    async fn hydrate(&self, query: &Q, candidates: &[C]) -> Vec<Result<C, String>>;
    async fn run(&self, query: &Q, candidates: &[C]) -> Vec<Result<C, String>> { ... }
    fn update(&self, candidate: &mut C, hydrated: C);
    fn update_all(&self, candidates: &mut [C], hydrated: Vec<Result<C, String>>) { ... }
    fn name(&self) -> &'static str;
}
```

**Critical invariant** (`hydrator.rs:24`):

> The returned vector must have the same candidates in the same order as the input.

Length mismatch → all results become `Err`. Per-candidate errors → that candidate retains its old field values; the candidate is **not dropped**.

### `Filter<Q, C>` (`candidate-pipeline/filter.rs:16-70`)

```rust
pub trait Filter<Q, C>: Any + Send + Sync {
    fn enable(&self, _query: &Q) -> bool { true }
    fn run(&self, query: &Q, candidates: Vec<C>) -> FilterResult<C> { ... }
    fn filter(&self, query: &Q, candidates: Vec<C>) -> FilterResult<C>;
    fn name(&self) -> &'static str;
    fn stat(&self, _result: &FilterResult<C>) { /* default no-op */ }
}

pub struct FilterResult<C> {
    pub kept:    Vec<C>,
    pub removed: Vec<C>,
}
```

Filters are **synchronous** (no async) and **run sequentially** — each filter operates on the kept output of the previous one (`candidate_pipeline.rs:365-371`). Removed candidates accumulate in `all_removed` for later side effects.

### `Scorer<Q, C>` (`candidate-pipeline/scorer.rs:9-65`)

```rust
#[async_trait]
pub trait Scorer<Q, C>: Send + Sync {
    fn enable(&self, _query: &Q) -> bool { true }
    async fn run(&self, query: &Q, candidates: &[C]) -> Vec<Result<C, String>> { ... }
    async fn score(&self, query: &Q, candidates: &[C]) -> Vec<Result<C, String>>;
    fn update(&self, candidate: &mut C, scored: C);
    fn update_all(&self, candidates: &mut [C], scored: Vec<Result<C, String>>) { ... }
    fn name(&self) -> &'static str;
}
```

Identical pattern to `Hydrator`: same length-preserving contract, same per-candidate error tolerance, **no candidate dropping allowed** (`scorer.rs:45`). Scorers run **sequentially** so each can read fields set by the previous (`candidate_pipeline.rs:394-404`).

### `Selector<Q, C>` (`candidate-pipeline/selector.rs:21-85`)

```rust
pub trait Selector<Q, C>: Send + Sync {
    fn enable(&self, _query: &Q) -> bool { true }
    fn run(&self, query: &Q, candidates: Vec<C>) -> SelectResult<C> { self.select(query, candidates) }
    fn select(&self, _query: &Q, candidates: Vec<C>) -> SelectResult<C> {
        let sorted = self.sort(candidates);
        match self.size() {
            Some(k) => {
                let (selected, non_selected) = split_at(sorted, k);
                SelectResult { selected, non_selected }
            }
            None => SelectResult { selected: sorted, non_selected: vec![] },
        }
    }
    fn score(&self, candidate: &C) -> f64;
    fn sort(&self, candidates: Vec<C>) -> Vec<C> { /* sort by score desc */ }
    fn size(&self) -> Option<usize> { None }
    fn name(&self) -> &'static str;
}

pub struct SelectResult<C> {
    pub selected:     Vec<C>,
    pub non_selected: Vec<C>,
}
```

Default selection is **score-based + truncating**. Non-selected candidates aren't discarded — they're passed to side effects. To do anything stateful (diversification, MMR), an implementation must override `select()` entirely.

### `SideEffect<Q, C>` (`candidate-pipeline/side_effect.rs:17-37`)

```rust
#[async_trait]
pub trait SideEffect<Q, C>: Send + Sync {
    fn enable(&self, _query: Arc<Q>) -> bool { true }
    async fn run(&self, input: Arc<SideEffectInput<Q, C>>) -> Result<(), String> { ... }
    async fn side_effect(&self, input: Arc<SideEffectInput<Q, C>>) -> Result<(), String>;
    fn name(&self) -> &'static str;
}

pub struct SideEffectInput<Q, C> {
    pub query: Arc<Q>,
    pub selected_candidates: Vec<C>,
    pub non_selected_candidates: Vec<C>,
}
```

Side effects run **in parallel on a background tokio task** spawned after the response is returned (`candidate_pipeline.rs:419-428`). **Failures are silent.**

## Execution model

```
                  ┌──────────────────────────────────────┐
                  │  CandidatePipeline::execute(query)   │
                  └──────────────────────────────────────┘
                                    │
                                    ▼
        ┌─────────────────────────────────────────────────────────┐
        │  query_hydrators() — independent       (join_all PAR)   │
        ├─────────────────────────────────────────────────────────┤
        │  dependent_query_hydrators()           (join_all PAR)   │
        ├─────────────────────────────────────────────────────────┤
        │  sources()                             (join_all PAR)   │
        │      └─ flatten Result<Vec<C>> → Vec<C>                 │
        ├─────────────────────────────────────────────────────────┤
        │  hydrators()                           (join_all PAR)   │
        │      └─ each receives full candidate batch              │
        ├─────────────────────────────────────────────────────────┤
        │  filters()                              (SEQUENTIAL)    │
        │      └─ each sees survivors of prior                    │
        ├─────────────────────────────────────────────────────────┤
        │  scorers()                              (SEQUENTIAL)    │
        │      └─ each reads fields set by prior                  │
        ├─────────────────────────────────────────────────────────┤
        │  selector().select() → SelectResult                     │
        ├─────────────────────────────────────────────────────────┤
        │  post_selection_hydrators()            (join_all PAR)   │
        ├─────────────────────────────────────────────────────────┤
        │  post_selection_filters()               (SEQUENTIAL)    │
        ├─────────────────────────────────────────────────────────┤
        │  truncate to result_size() if set                       │
        │      └─ excess → non_selected                           │
        ├─────────────────────────────────────────────────────────┤
        │  finalize() hook (impl-specific)                        │
        ├─────────────────────────────────────────────────────────┤
        │  RESPONSE RETURNED ◄──────────────────────────┐         │
        ├───────────────────────────────────────────────┘         │
        │  side_effects() spawned on tokio::spawn       │         │
        │      └─ join_all PAR, errors silent           ▼         │
        └─────────────────────────────────────────────────────────┘
```

## Composition

There is **no single `CandidatePipeline` struct**. `CandidatePipeline<Q, C>` is an async trait (`candidate_pipeline.rs:68-493`) that concrete pipelines implement by returning slices of trait objects:

```rust
#[async_trait]
pub trait CandidatePipeline<Q, C> {
    fn sources(&self) -> &[Box<dyn Source<Q, C>>];
    fn query_hydrators(&self) -> &[Box<dyn QueryHydrator<Q>>];
    fn dependent_query_hydrators(&self) -> &[Box<dyn QueryHydrator<Q>>] { &[] }
    fn hydrators(&self) -> &[Box<dyn Hydrator<Q, C>>];
    fn filters(&self) -> &[Box<dyn Filter<Q, C>>];
    fn scorers(&self) -> &[Box<dyn Scorer<Q, C>>];
    fn selector(&self) -> &dyn Selector<Q, C>;
    fn post_selection_hydrators(&self) -> &[Box<dyn Hydrator<Q, C>>] { &[] }
    fn post_selection_filters(&self) -> &[Box<dyn Filter<Q, C>>] { &[] }
    fn side_effects(&self) -> &[Box<dyn SideEffect<Q, C>>] { &[] }
    fn result_size(&self) -> Option<usize> { None }
    async fn finalize(&self, _selected: &mut Vec<C>, _query: &Q) { }
    async fn execute(&self, query: Q) -> ExecuteResult<C> { /* default orchestration */ }
}
```

To define a pipeline (e.g., `PhoenixCandidatePipeline`), implement this trait, return the desired components, and call `execute(query)`.

## Error handling philosophy

The framework is built around **graceful partial failure**:

| Failure | Behavior |
|---|---|
| Source returns `Err` | Logged; that source contributes 0 candidates; pipeline continues. |
| Hydrator returns `Err` for one candidate | That candidate retains old field values; not dropped. |
| Hydrator returns wrong-length vector | All results become `Err`; all candidates skipped for this hydrator. |
| Scorer returns `Err` for one candidate | Same as hydrator. |
| QueryHydrator returns `Err` | Logged; old query value retained (field-by-field merge means failed fields stay default). |
| SideEffect returns `Err` | Silent — never reaches the client, never reaches the response. |

There's no "abort the pipeline" path. The system is designed to **always return a result**, even if degraded.

## Caching support: `CachedHydrator`

`hydrator.rs:84-189` defines a blanket impl:

```rust
pub trait CachedHydrator<Q, C>: Send + Sync {
    type CacheKey: Eq + Hash + Send + Sync + 'static;
    type CacheValue: Clone + Send + Sync + 'static;

    fn cache(&self) -> &Cache<Self::CacheKey, Self::CacheValue>;
    fn key_for(&self, query: &Q, candidate: &C) -> Self::CacheKey;
    async fn hydrate_uncached(&self, query: &Q, keys: &[Self::CacheKey])
        -> Vec<Result<Self::CacheValue, String>>;
    fn apply(&self, candidate: &mut C, value: Self::CacheValue);
}

// Blanket impl of Hydrator<Q, C> for everything implementing CachedHydrator:
//   - check cache for each candidate
//   - call hydrate_uncached only for misses
//   - populate cache, then apply
//   - emit cache_hit / cache_miss metrics
```

This is how things like `GizmoduckCandidateHydrator` avoid hammering the user-profile service for repeat lookups within and across requests.

## Observability

Every stage is instrumented:

- `#[tracing::instrument(skip_all, name = "query_hydrators", fields(total_count = Empty, enabled_count = Empty, disabled = Empty))]`
- `#[xai_stats_macro::receive_stats]`

The framework records:

- `total_count`, `enabled_count`, `disabled` (component names) per stage. (`candidate_pipeline.rs:435-454`)
- `latency_ms` for query hydration, source fetching, hydration, scoring. (`candidate_pipeline.rs:458-460`)
- `size` (candidate count) at producer/consumer boundaries. (`candidate_pipeline.rs:462-463`)
- `filter_rate`, `kept`, `removed` per filter. (`candidate_pipeline.rs:466-476`)
- `result_size` histogram, `result_empty` counter. (`candidate_pipeline.rs:478-491`)
- `cache_hit`, `cache_miss` for `CachedHydrator`. (`hydrator.rs:104-114`)

Disabled components are counted but not run — this lets ops dashboards measure "how often is feature X off?"

## Surprising behaviors

### 1. Post-selection stages exist

A separate hydration + filtering pass runs **after** selection (`candidate_pipeline.rs:82-83, 107-112`). This allows late-binding enrichment that's too expensive to run on the full candidate set (e.g., visibility filtering, brand safety, mutual-follow Jaccard).

### 2. Result truncation is separate from selection

After post-selection filters, the framework **again** truncates to `result_size()` and moves the excess to `non_selected` (`candidate_pipeline.rs:115-117`). So `non_selected` can contain three populations: selector's non-selected, post-filter removals, and final truncation excess.

### 3. Query hydrators return full queries but merge selectively

Each `QueryHydrator.hydrate()` returns a new `Q`, but only the hydrator's `update()` method decides what to merge. Implementers must only populate their fields and leave others default — otherwise they'd clobber peer hydrators.

### 4. No parallel candidate processing

Despite each hydrator running in parallel with other hydrators, **within a single hydrator** the candidate batch is processed however that hydrator likes — usually one batched gRPC call or one parallel scatter, but the framework doesn't enforce this. The framework doesn't fan out per-candidate.

### 5. Disabled-but-counted

Components that report `enable(query) → false` are still listed in trace fields (`disabled`). This makes feature-flag visibility automatic.

### 6. Side-effect failures are completely silent

No metric, no log at the framework level. The side-effect implementation must do its own error handling and emission. This is a deliberate "fire and forget" design.

## Common patterns in concrete pipelines

- **Sources**: 5-7 in parallel.
- **Hydrators**: 8-15 in parallel pre-selection, 4-8 post-selection.
- **Filters**: 10-20 pre-scoring, 2-5 post-selection.
- **Scorers**: 2-4 sequential.
- **Selector**: usually `TopKScoreSelector` or `BlenderSelector`.
- **Side effects**: 5-10 in parallel, all writing to Kafka/Redis/HTTP.

This framework is **opinionated about the shape of recommendation pipelines** — try to bend it into something like a bandit explorer or a multi-armed sampler and you'll be working against the grain.
