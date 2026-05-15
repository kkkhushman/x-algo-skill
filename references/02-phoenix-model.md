# 02 — Phoenix Model

The ML brain. JAX/Haiku code, ported from `xai-org/grok-1`, adapted for two-tower retrieval and candidate-isolation ranking.

## Two-stage architecture

```
   ┌─────────────────────┐     ┌─────────────────────┐
   │  STAGE 1: RETRIEVAL │     │  STAGE 2: RANKING   │
   │  Two-Tower Model    │     │  Transformer        │
   │                     │     │                     │
   │  Millions → 1000s   │     │  1000s → Scored 19  │
   └─────────────────────┘     └─────────────────────┘
```

Both stages **share an embedding table** (a single unified hash-bucket table for users, items, authors, with a `pad=65` offset to separate the namespaces inside one tensor).

## Stage 1: Two-tower retrieval

File: `phoenix/recsys_retrieval_model.py`

### User tower
- A full Grok transformer (same architecture as the ranking model).
- Input: `[user_hashes, history_post_hashes, history_author_hashes, history_actions, ...]`.
- Pooling: **masked mean** over the history positions, producing one user representation per request.
- Output: `[B, D]` L2-normalized.

### Candidate tower
- **Lightweight projection**, not a transformer. (`recsys_retrieval_model.py:47-113`)
- Input: concatenated `[post_embeddings, author_embeddings]` of shape `[B, C, 2*num_hashes, D]`.
- Two modes:
  - `enable_linear_proj=True`: 2-layer MLP with SiLU activation, `[2*num_hashes*D, 2*D] → SiLU → [2*D, D]`, then L2 normalize.
  - `enable_linear_proj=False`: mean-pool across the hash dimension, then L2 normalize.
- Output: `[B, C, D]` L2-normalized — designed to be **pre-computable offline** for the entire corpus.

### Inference

```
corpus_repr [N, D]  (pre-computed, L2-normalized)
user_repr   [1, D]  (computed online via the user tower)
scores = corpus_repr @ user_repr[0]   # [N]
top_k  = np.argpartition + sort
```

No FAISS, no ScaNN — the `sports_corpus.npz` is a flat 537K × D table. Production presumably uses ANN at scale; the repo doesn't include that path.

`sports_corpus.npz` keys: `post_ids`, `candidate_representations` (the L2-normalized embeddings), `author_ids`, optional `topics`.

## Stage 2: Transformer ranking with candidate isolation

File: `phoenix/recsys_model.py` + `phoenix/grok.py`

### Architecture (mini-checkpoint shipped in the repo)

| Setting | Value |
|---|---|
| Embedding dim `D` | 128 |
| Layers | 4 |
| Attention heads | 4 query / 4 KV (effectively MHA, ratio 1:1) |
| Key size per head | 32 |
| FFN dim | derived: `widening_factor(2.0) * D * 2 / 3`, padded to multiple of 8 → 176 |
| FFN type | SwiGLU (GELU gate × identity, parallel projections, elementwise multiply) |
| Norm | RMSNorm, pre-attention and pre-FFN |
| Position encoding | RoPE, base_exponent=10000 |
| Attention dtype | computed in fp32, cast back |
| Attention logit clamping | `30 * tanh(logits / 30)` — bounded softmax, no inf masking |
| MoE | none in this port |

(`phoenix/grok.py:32-35, 116, 141, 351, 363-379, 453-464`)

### Inputs

`RecsysBatch` (`recsys_model.py:126-145`):

| Field | Shape | Notes |
|---|---|---|
| `user_hashes` | `[B, num_user_hashes]` | hash bucket indices, 0 = pad |
| `history_post_hashes` | `[B, S, num_item_hashes]` | |
| `history_author_hashes` | `[B, S, num_author_hashes]` | |
| `history_actions` | `[B, S, num_actions]` | multi-hot |
| `history_product_surface` | `[B, S]` | small categorical |
| `candidate_post_hashes` | `[B, C, num_item_hashes]` | |
| `candidate_author_hashes` | `[B, C, num_author_hashes]` | |
| `candidate_product_surface` | `[B, C]` | |
| `history_continuous_actions` | optional | dwell time etc. |
| `candidate_impr_ts` | optional | for time-aware features |
| `candidate_post_creation_ts` | optional | post age bucketing |
| `user_ip_hashes` | optional | if `use_ip_address=True` |

Embeddings are looked up from the unified table by hash indices into `RecsysEmbeddings`, then **collisions are mixed by concatenate + linear projection** down to dim `D`. (`recsys_model.py:175, 185, 233, 260, 300, 326`)

### Outputs

`RecsysModelOutput` (`recsys_model.py:119-124`):

- `logits`: `[B, C, num_actions]` — raw engagement logits per candidate. Sigmoid applied at inference to get probabilities.
- `continuous_preds`: `[B, C, num_continuous_actions]` — sigmoid-activated continuous predictions (e.g., dwell time, watch time).

### Canonical action ordering (output index → name)

From `phoenix/runners.py:233-253` (`RankingOutput`):

```
 0  favorite_score
 1  reply_score
 2  repost_score
 3  photo_expand_score
 4  click_score
 5  profile_click_score
 6  vqv_score                 (video quality view)
 7  share_score
 8  share_via_dm_score
 9  share_via_copy_link_score
10  dwell_score
11  quote_score
12  quoted_click_score
13  follow_author_score
14  not_interested_score      ← NEGATIVE feedback
15  block_author_score        ← NEGATIVE feedback
16  mute_author_score         ← NEGATIVE feedback
17  report_score              ← NEGATIVE feedback
18  dwell_time                (the discrete dwell logit; continuous version below)
```

Continuous predictions (`runners.py:255-264`):

```
 0  reserved
 1  dwell_time
 2  video_watch_time
 3  scroll_depth
 4-7 reserved
```

**Note:** the top-level repo README mentions "15 actions" in the overview. The model emits **19 discrete + ~4 continuous**. Use the canonical list above.

## Candidate-isolation attention mask

This is the load-bearing design decision in the ranking model. Built by `make_recsys_attn_mask` (`phoenix/grok.py:39-71`):

```python
causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len), dtype=dtype))
attn_mask   = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)
candidate_indices = jnp.arange(candidate_start_offset, seq_len)
attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)
```

The resulting mask:

- **Rows in `[0, candidate_start_offset)`** (user-context tokens): standard causal — can attend to themselves and prior context tokens.
- **Rows in `[candidate_start_offset, seq_len)`** (candidate tokens): can attend to **all user-context tokens** AND to **themselves only** (the diagonal). They **cannot** attend to other candidates.

Applied in `Transformer.__call__` at `phoenix/grok.py:571-581` via elementwise multiplication with the padding mask:

```python
mask = mask * attn_mask
```

### Consequences

- Each candidate's logit depends **only** on the user context, not on which other candidates happen to be in the batch.
- Scores are **stable** across paginations and **cacheable** per `(user, candidate)`.
- The model can score candidates in parallel chunks without changing results — `run_pipeline.py:325-352` does exactly this with `candidate_seq_len=64`.
- The model can be **distilled / served at scale** with simpler architectures preserving this property.

## Hash-based embeddings

File: `phoenix/run_pipeline.py:76-114`, `phoenix/recsys_model.py:94-100`

Three entity types — users, posts, authors — each with its own hash space.

### The hash function

A linear congruential generator (LCG) per hash function:

```python
raw = (id * scale + bias) % modulus
out = 0 if id == 0 else int(raw % (num_buckets - 1)) + 1
```

- 0 is **reserved for padding** — any actual entity gets bucket `[1, num_buckets]`.
- Each entity type has `num_*_hashes` (default 2) functions with distinct `(scale, bias)` pairs.

### Collision mixing

After looking up `num_hashes` separate embeddings of dim `D`:

```
[B, S, num_hashes, D] → reshape → [B, S, num_hashes * D] → Linear → [B, S, D]
```

Concatenate-then-project, not sum/average. (`recsys_model.py:233, 260, 300, 326`)

### Hash params

Stored in the model's `config.json` under `hash_params`:

```
user_hash_scales, user_biases, user_modulus, user_vocab_size
item_hash_scales, item_biases, item_modulus, item_vocab_size
author_hash_scales, author_biases, author_modulus, author_vocab_size
```

The mini config: 1M vocab per entity, 2 hash functions each.

### Why this works

- **Memory-bounded**: vocab size is constant regardless of how many real users/posts/authors exist.
- **Collisions are graceful**: the projection layer learns to disambiguate via the combination of the 2 hash slots.
- **Pad=0 reserved everywhere** simplifies masking — anywhere an id is missing, the hash is 0, the lookup returns the pad embedding, and downstream mask logic just checks for zero.

## End-to-end inference (`phoenix/run_pipeline.py`)

```
1. Load artifacts
   - retrieval params + embeddings table (artifacts/retrieval/)
   - ranking   params + embeddings table (artifacts/ranker/)
   - sports_corpus.npz: pre-computed L2-normalized corpus embeddings + post_ids
   - example_sequence.json: {user_id, history: [{post_id, author_id, actions}]}

2. Hash user / history
   user_id     → [1, num_user_hashes]
   post_ids    → [1, S, num_item_hashes]
   author_ids  → [1, S, num_author_hashes]

3. Retrieval
   batch = build_batch_with_zero_candidates(user, history)
   embeds = lookup_embeddings(unified_table, batch.hashes)
   user_repr = retrieval_model.apply(retrieval_params, batch, embeds)  # [1, D]
   scores = corpus_repr @ user_repr[0]                                  # [N]
   top_k = argpartition(scores, K)[:K] sorted by score                  # [K]

4. Ranking
   # Re-hash top_k post_ids and author_ids using RANKER's hash params
   for chunk in batch_iter(top_k_ids, size=candidate_seq_len):
       batch  = build_batch(user, history, chunk)
       embeds = lookup_embeddings(ranker_table, batch.hashes)
       logits = ranking_model.apply(rank_params, batch, embeds)         # [1, C, 19]
       probs  = sigmoid(logits)
   probs_all = concat(chunks)

5. Final score
   final = 1.0*fav + 0.5*reply + 0.3*rt + 0.2*dwell   # demo weights only!
   ranked = sort(top_k_post_ids by final desc)
```

**Important:** the weights in step 5 of `run_pipeline.py` are **demo-only**. In production, weights come from feature switches per-request in home-mixer's `RankingScorer`. See `references/05-ranking-and-blending.md`.

## Non-obvious model features

### Right-anchored RoPE positions

Optional flag in `PhoenixModelConfig` (`recsys_model.py:362`). When enabled, the **newest** history item is pinned to a fixed RoPE position, so variable-length histories don't cause position drift across requests. Useful for streaming/online inference. Implementation: `grok.py:88-109`.

### Post age bucketing

Continuous post age → discrete bucket index (`recsys_model.py:36-55`):

- Bucket 0 = missing/invalid
- Buckets 1..N = age in `granularity_minutes` (default 60) increments
- Overflow bucket for posts older than `POST_AGE_MAX_MINUTES = 4800` (80h)

### Dwell time → embedding lift

Dwell time is a continuous input, but it's projected into embedding space via a 2-layer MLP (hidden=64) before being added to history embeddings. (`recsys_model.py:487-518`)

### IP address embeddings (optional)

If `use_ip_address=True`, IP hashes are looked up and **summed** into the user embedding. (`recsys_model.py:190-193, 360`)

### Product surface vocab

Small categorical (default vocab=16) representing the UI context (home timeline, search, notifications, etc.). Embedded, then concatenated with post + author features. (`recsys_model.py:350`)

### `mask_neg_feedback_on_negatives`

Config flag (`recsys_model.py:364`) — purpose not fully evident from code alone, likely a training-time regularization that zeroes positive-action labels for posts also marked as negative feedback.

## What the repo does NOT include

- The training loop (only inference + tests).
- The full production checkpoint (only 128-dim, 4-layer mini).
- The ANN index for corpus retrieval (only brute-force on flat L2-normalized table).
- The candidate-tower offline computation pipeline (sports_corpus.npz is pre-baked).
- The continuous-update online training infrastructure.
