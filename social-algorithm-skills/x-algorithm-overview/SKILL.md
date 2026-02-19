---
name: x-algorithm-overview
description: Concise overview of the X For You feed algorithm codebase. Use when asking about how the X algorithm works, what this repository contains, where to find things, how the components connect, or as a starting point for understanding the system.
---

# X Algorithm Overview

The X "For You" feed algorithm takes a user and returns a ranked list of posts. It combines posts from accounts you follow with ML-discovered content, scores everything with a Grok-based transformer, and returns the top results.

## The 4 Components

| Component | Directory | Language | Purpose | Start Here |
|-----------|-----------|----------|---------|------------|
| **Home Mixer** | `home-mixer/` | Rust | Orchestration gRPC service -- wires everything together | `main.rs`, `server.rs` |
| **Thunder** | `thunder/` | Rust | Real-time in-memory post store, Kafka-fed, serves in-network posts | `posts/post_store.rs` |
| **Phoenix** | `phoenix/` | Python/JAX | ML brain -- two-tower retrieval + Grok-based transformer ranker | `recsys_model.py`, `grok.py` |
| **Candidate Pipeline** | `candidate-pipeline/` | Rust | Generic reusable pipeline framework (traits for Source, Filter, Scorer, etc.) | `candidate_pipeline.rs` |

## Data Flow

```
User Request
     |
     v
HOME MIXER (Rust gRPC)
     |
     |-- [Query Hydration] Fetch user context: engagement history, following list,
     |                     blocked/muted users, muted keywords
     |
     |-- [Candidate Sources] (parallel)
     |      |-- THUNDER (Rust) --> In-network posts from people you follow
     |      |-- PHOENIX Retrieval (Python/JAX) --> Out-of-network posts via ML similarity
     |
     |-- [Hydration] (parallel) Enrich with author info, media, subscriptions
     |-- [Filters] (sequential) Remove: duplicates, old, blocked, muted, already-seen
     |
     |-- [Scoring] (sequential)
     |      |-- Phoenix Scorer --> 19 engagement probabilities per post (Grok transformer)
     |      |-- Weighted Scorer --> Single score = SUM(weight * probability)
     |      |-- Author Diversity --> Decay repeated author scores
     |      |-- OON Scorer --> Adjust in-network vs discovered balance
     |
     |-- [Selection] Sort by score, pick top K
     |-- [Post-Selection] Visibility filtering, conversation dedup
     |
     v
Ranked Feed Response
```

## Why Rust + Python

- **Rust** (home-mixer, thunder, candidate-pipeline): Serving infrastructure needs low latency, high concurrency, and memory safety. gRPC servers, in-memory stores, Kafka consumers, and pipeline orchestration.
- **Python/JAX** (phoenix): ML models need the Python ecosystem for research velocity and JAX for hardware-accelerated inference. The transformer and retrieval models live here.

Home Mixer calls Phoenix and Thunder via **gRPC** -- they're separate services.

## 5 Key Design Decisions

1. **No hand-engineered features** -- The Grok transformer learns relevance entirely from engagement history sequences. No manual feature engineering.
2. **Candidate isolation mask** -- Candidates cannot attend to each other in the transformer, guaranteeing score independence regardless of batch composition.
3. **Multi-action prediction** -- The model predicts 19 engagement types (like, reply, block, mute...) instead of one score. Positive actions get positive weights, negative actions get negative weights.
4. **Two-stage retrieval + ranking** -- Retrieval (two-tower, dot product) narrows millions to thousands cheaply. Ranking (full transformer) orders thousands precisely.
5. **Composable pipeline** -- Every stage (Source, Filter, Scorer, Hydrator, Selector, SideEffect) is a trait. New behavior = implement a trait and plug it in.

## Running It

```bash
# From the phoenix/ directory:
uv run run_ranker.py                                              # Demo ranking model
uv run run_retrieval.py                                           # Demo retrieval model
uv run pytest test_recsys_model.py test_recsys_retrieval_model.py # Run tests
```

Note: Uses random weights (no trained model included). Tests verify architecture correctness, not prediction quality.

## Related Skills

For deeper dives, see these companion skills:

| Skill | What It Covers |
|-------|----------------|
| `social-media-algorithm` | General guide to building a social media recommendation algorithm from scratch |
| `transformer-attention-masks` | Deep dive into the candidate isolation attention mask and custom mask design |
| `engagement-scoring-weights` | How the 4-scorer chain works, the weighted formula, author diversity math, and tuning guide |
| `candidate-pipeline-framework` | The Rust pipeline framework: all 7 traits, design patterns, and how to add new components |
