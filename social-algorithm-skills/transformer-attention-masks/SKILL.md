---
name: transformer-attention-masks
description: Guide to designing custom attention masks for transformer models, focusing on the candidate isolation pattern used in recommendation systems. Use when asking about attention masks, candidate isolation, how candidates are scored independently, transformer masking strategies, or batch-invariant scoring.
---

# Transformer Attention Masks for Recommendation Systems

This skill explains how attention masks control information flow in transformers, with a deep focus on the **candidate isolation mask** -- the key architectural innovation in X's Phoenix ranking model.

## Background: What Attention Masks Do

In a transformer, every position can potentially attend to every other position. Attention masks restrict this by setting certain positions to "cannot attend" (masked out with `-inf` before softmax). This controls **what information flows where**.

### Standard Mask Types

| Mask Type | Pattern | Use Case |
|-----------|---------|----------|
| **Causal (autoregressive)** | Lower triangular -- position `i` can only see positions `<= i` | GPT-style language models, sequential prediction |
| **Bidirectional** | All ones -- every position sees every other | BERT-style encoding, classification |
| **Padding** | Zeros for pad tokens, ones for real tokens | Variable-length sequences in a batch |

```
Causal:              Bidirectional:       Padding (len=3, pad=2):
1 0 0 0 0            1 1 1 1 1            1 1 1 0 0
1 1 0 0 0            1 1 1 1 1            1 1 1 0 0
1 1 1 0 0            1 1 1 1 1            1 1 1 0 0
1 1 1 1 0            1 1 1 1 1            0 0 0 0 0
1 1 1 1 1            1 1 1 1 1            0 0 0 0 0
```

## The Candidate Isolation Mask

### The Problem

In a recommendation system, the transformer scores multiple candidate posts in a single forward pass for efficiency. But if candidates can attend to each other, then **adding, removing, or reordering candidates changes every other candidate's score**. This makes scores:
- **Non-deterministic**: Same post gets different scores depending on batch composition
- **Non-cacheable**: You can't cache a post's score because it depends on its batch neighbors
- **Hard to debug**: Score changes when you change something unrelated

### The Solution

The candidate isolation mask ensures each candidate can see the user context (user + engagement history) but **cannot see other candidates**. Each candidate only attends to itself in the candidate block.

### How It's Built (3 Steps)

From `phoenix/grok.py`:

```python
def make_recsys_attn_mask(seq_len, candidate_start_offset, dtype=jnp.float32):
    # Step 1: Start with a standard causal mask
    causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len), dtype=dtype))

    # Step 2: Zero out the entire candidate-to-candidate block (bottom-right)
    attn_mask = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)

    # Step 3: Add back self-attention on the diagonal for candidates
    candidate_indices = jnp.arange(candidate_start_offset, seq_len)
    attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)

    return attn_mask
```

### Visual: The Exact Mask

For a sequence `[user, h1, h2, c1, c2, c3]` where `candidate_start_offset = 3`:

```
         Keys (what we attend TO):
         user  h1   h2   c1   c2   c3
Queries:
  user  [  1    0    0    0    0    0  ]   <- causal: user sees only itself
  h1    [  1    1    0    0    0    0  ]   <- causal: h1 sees user + itself
  h2    [  1    1    1    0    0    0  ]   <- causal: h2 sees user + h1 + itself
  c1    [  1    1    1    1    0    0  ]   <- c1 sees ALL context + ONLY itself
  c2    [  1    1    1    0    1    0  ]   <- c2 sees ALL context + ONLY itself
  c3    [  1    1    1    0    0    1  ]   <- c3 sees ALL context + ONLY itself
```

Notice:
- **Top-left block** (user+history): Standard causal attention
- **Bottom-left block** (candidates -> context): Full attention -- candidates see ALL user+history
- **Bottom-right block** (candidates -> candidates): **Diagonal only** -- each candidate sees only itself, never other candidates

### How It's Applied in the Transformer

In `phoenix/grok.py`, the `Transformer.__call__()` method combines the structural mask with the padding mask:

```python
if candidate_start_offset is not None:
    # Recommendation system mode: candidate isolation
    attn_mask = make_recsys_attn_mask(seq_len, candidate_start_offset, fprop_dtype)
    mask = padding_mask * attn_mask  # combine both masks
else:
    # Standard mode: causal attention only
    causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len)))
    mask = padding_mask * causal_mask
```

The padding mask is `[B, 1, 1, T]` (broadcast over heads and query positions) and the structural mask is `[1, 1, T, T]` (broadcast over batch). Their element-wise product gives the final `[B, 1, T, T]` mask that the attention mechanism uses.

### Where `candidate_start_offset` Comes From

In `phoenix/recsys_model.py`, the input sequence is built by concatenating:

```python
embeddings = concat([user_embedding, history_embeddings, candidate_embeddings], axis=1)
#                    [B, 1, D]       [B, S, D]            [B, C, D]

candidate_start_offset = 1 + S  # user (1 position) + history (S positions)
```

This offset tells the mask builder where the candidate block begins.

## When to Use Which Mask

| Scenario | Mask | Why |
|----------|------|-----|
| Ranking candidates against user context | **Candidate isolation** | Score independence, cacheability |
| Encoding user history (retrieval user tower) | **Causal** | History is a sequence with temporal ordering |
| Encoding a single item (candidate tower) | **None / Bidirectional** | Item features have no order |
| Training the ranking model | **Candidate isolation** | Must match inference behavior |

In this codebase:
- The **ranking model** (`recsys_model.py`) passes `candidate_start_offset` to the transformer -- uses isolation mask
- The **retrieval user tower** (`recsys_retrieval_model.py`) passes `candidate_start_offset=None` -- uses standard causal mask

## Adapting for Other Use Cases

The candidate isolation pattern generalizes to any scenario where you need to **score multiple items independently against shared context**:

- **RAG (Retrieval-Augmented Generation)**: Score multiple retrieved documents against a query without documents influencing each other's relevance
- **Multi-turn conversation**: Isolate response candidates so each is scored against the conversation history independently
- **Ad ranking**: Score multiple ads against user context without ad-to-ad information leakage
- **Search result scoring**: Score search results against a query independently

The general recipe:
1. Define which positions are "shared context" (always visible to all)
2. Define which positions are "candidates" (isolated from each other)
3. Build the mask: context gets causal/bidirectional among themselves, candidates see all context + only themselves

## Code References

| File | What |
|------|------|
| `phoenix/grok.py` lines 39-71 | `make_recsys_attn_mask()` implementation |
| `phoenix/grok.py` lines 516-551 | `Transformer.__call__()` -- mask application |
| `phoenix/recsys_model.py` lines 428-437 | Input construction and `candidate_start_offset` calculation |
| `phoenix/recsys_model.py` lines 453-462 | Forward pass passing offset to transformer |
| `phoenix/test_recsys_model.py` lines 22-187 | 8 test cases verifying mask correctness |
