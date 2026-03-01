---
name: candidate-pipeline-framework
description: Guide to the candidate pipeline framework used in the X algorithm. Use when asking about the pipeline architecture, adding new components (filters, scorers, sources, hydrators), trait interfaces, how stages are orchestrated, or how to extend the recommendation system.
---

# Candidate Pipeline Framework

This skill explains the generic, reusable Rust framework (`candidate-pipeline/`) that powers the X recommendation pipeline. It defines trait-based interfaces for every stage and orchestrates their execution with parallel processing, error handling, and request tracing.

## Overview

The pipeline processes candidates through 7 stages in a fixed order. Each stage is defined as a **trait** (interface) that you implement to add new behavior. The framework handles orchestration, parallelism, error recovery, and logging.

```
execute(query)
    |
    v
[1] hydrate_query()       -- QueryHydrator traits (parallel)
    |
    v
[2] fetch_candidates()    -- Source traits (parallel)
    |
    v
[3] hydrate()             -- Hydrator traits (parallel)
    |
    v
[4] filter()              -- Filter traits (sequential)
    |
    v
[5] score()               -- Scorer traits (sequential)
    |
    v
[6] select()              -- Selector trait (single)
    |
    v
[7] post-selection        -- Hydrators (parallel) then Filters (sequential)
    |
    v
[8] side_effects          -- SideEffect traits (fire-and-forget, parallel)
    |
    v
PipelineResult { retrieved, filtered, selected }
```

## The 7 Trait Interfaces

### 1. Source -- Fetch Candidates

```rust
#[async_trait]
pub trait Source<Q, C>: Send + Sync {
    fn enable(&self, query: &Q) -> bool { true }
    async fn get_candidates(&self, query: &Q) -> Result<Vec<C>, String>;
}
```

- **Execution**: All sources run in **parallel** via `join_all`
- **Contract**: Return a `Vec<C>` of candidates. Results from all sources are concatenated.
- **On failure**: Error is logged, source is skipped. Other sources still contribute.
- **Examples**: `ThunderSource` (in-network posts), `PhoenixSource` (ML retrieval)

### 2. QueryHydrator -- Enrich the Query

```rust
#[async_trait]
pub trait QueryHydrator<Q>: Send + Sync {
    fn enable(&self, query: &Q) -> bool { true }
    async fn hydrate(&self, query: &Q) -> Result<Q, String>;
    fn update(&self, query: &mut Q, hydrated: Q);
}
```

- **Execution**: All query hydrators run in **parallel**
- **Contract**: `hydrate()` returns a new query with only YOUR fields populated. `update()` merges those fields into the original.
- **Examples**: `UserActionSeqQueryHydrator` (engagement history), `UserFeaturesQueryHydrator` (blocked/muted lists)

### 3. Hydrator -- Enrich Candidates

```rust
#[async_trait]
pub trait Hydrator<Q, C>: Send + Sync {
    fn enable(&self, query: &Q) -> bool { true }
    async fn hydrate(&self, query: &Q, candidates: &[C]) -> Result<Vec<C>, String>;
    fn update(&self, candidate: &mut C, hydrated: C);
    fn update_all(&self, candidates: &mut [C], hydrated: Vec<C>);
}
```

- **Execution**: All hydrators run in **parallel**
- **Contract**: MUST return the same number of candidates in the same order. Only populate YOUR fields.
- **Safety**: If returned length doesn't match input length, the hydrator is **skipped** with a warning (not a crash)
- **Examples**: `CoreDataCandidateHydrator` (post text, author), `GizmoduckCandidateHydrator` (screen names), `VideoDurationCandidateHydrator`

### 4. Filter -- Remove Ineligible Candidates

```rust
pub struct FilterResult<C> {
    pub kept: Vec<C>,
    pub removed: Vec<C>,
}

#[async_trait]
pub trait Filter<Q, C>: Send + Sync {
    fn enable(&self, query: &Q) -> bool { true }
    async fn filter(&self, query: &Q, candidates: Vec<C>) -> Result<FilterResult<C>, String>;
}
```

- **Execution**: Filters run **sequentially** (order matters -- earlier filters reduce work for later ones)
- **Contract**: Partition candidates into `kept` (continue) and `removed` (excluded). Removed candidates are tracked, not silently dropped.
- **On failure**: Error is logged, **backup candidates are restored** (the filter is skipped, not the candidates)
- **Examples**: `DropDuplicatesFilter`, `AgeFilter`, `MutedKeywordFilter`, `AuthorSocialgraphFilter`

### 5. Scorer -- Compute Scores

```rust
#[async_trait]
pub trait Scorer<Q, C>: Send + Sync {
    fn enable(&self, query: &Q) -> bool { true }
    async fn score(&self, query: &Q, candidates: &[C]) -> Result<Vec<C>, String>;
    fn update(&self, candidate: &mut C, scored: C);
    fn update_all(&self, candidates: &mut [C], scored: Vec<C>);
}
```

- **Execution**: Scorers run **sequentially** (each builds on the previous scorer's output)
- **Contract**: MUST return same count in same order. Only populate YOUR score fields. **Never drop candidates** -- use a Filter for that.
- **Split-update pattern**: `score()` returns new objects with only scored fields set. `update()` merges just those fields into originals.
- **Examples**: `PhoenixScorer` -> `WeightedScorer` -> `AuthorDiversityScorer` -> `OONScorer`

### 6. Selector -- Sort and Truncate

```rust
pub trait Selector<Q, C>: Send + Sync {
    fn select(&self, query: &Q, candidates: Vec<C>) -> Vec<C>;
    fn score(&self, candidate: &C) -> f64;
    fn sort(&self, candidates: Vec<C>) -> Vec<C>;  // default: sort by score() descending
    fn size(&self) -> Option<usize>;                // default: None (no truncation)
}
```

- **Execution**: Single selector, runs once
- **Default implementation**: Sorts by `score()` descending, truncates to `size()` if set
- **Example**: `TopKScoreSelector`

### 7. SideEffect -- Fire-and-Forget Actions

```rust
#[async_trait]
pub trait SideEffect<Q, C>: Send + Sync {
    fn enable(&self, query: Arc<Q>) -> bool { true }
    async fn run(&self, input: Arc<SideEffectInput<Q, C>>) -> Result<(), String>;
}

pub struct SideEffectInput<Q, C> {
    pub query: Arc<Q>,
    pub selected_candidates: Vec<C>,
}
```

- **Execution**: All side effects run in **parallel** via `tokio::spawn` -- they do NOT block the response
- **Contract**: Receives the final query and selected candidates. Used for caching, logging, analytics.
- **Example**: `CacheRequestInfoSideEffect` (writes served post IDs for dedup on next request)

## Key Design Patterns

### Pattern 1: `enable()` Gating

Every component has an `enable(query)` method. This lets you conditionally disable components per-request:

```rust
// PhoenixSource is disabled for in-network-only requests
fn enable(&self, query: &ScoredPostsQuery) -> bool {
    !query.in_network_only
}
```

### Pattern 2: Split-Update

Scorers and hydrators use a two-step process to prevent data corruption:

1. `score()`/`hydrate()` returns **new objects** with only THIS component's fields set (everything else is Default)
2. `update()` copies **only its own fields** from the new object into the original candidate

This means multiple hydrators can run in parallel without overwriting each other's data.

### Pattern 3: Graceful Degradation

The pipeline never crashes from a single component failure:

- **Source fails**: Logged, skipped. Other sources still provide candidates.
- **Hydrator fails**: Logged, skipped. Candidates continue with un-hydrated fields.
- **Filter fails**: Logged, **backup restored**. Candidates pass through unfiltered rather than being lost.
- **Scorer fails**: Logged, skipped. Candidates keep their previous scores.
- **Length mismatch**: Hydrator/scorer returning wrong count is skipped with a warning.

### Pattern 4: Observability

- Every component auto-generates a `name()` from its Rust type name
- Every operation logs with `request_id` and `PipelineStage` for distributed tracing
- `PipelineResult` returns ALL three candidate sets (retrieved, filtered, selected) for debugging
- Side effects receive the final state for analytics/caching

## How to Add a New Component

### Adding a New Filter

1. Create a new file in `home-mixer/filters/`:

```rust
use xai_candidate_pipeline::filter::{Filter, FilterResult};

pub struct MyNewFilter;

#[async_trait]
impl Filter<ScoredPostsQuery, PostCandidate> for MyNewFilter {
    async fn filter(
        &self,
        query: &ScoredPostsQuery,
        candidates: Vec<PostCandidate>,
    ) -> Result<FilterResult<PostCandidate>, String> {
        let (kept, removed) = candidates
            .into_iter()
            .partition(|c| /* your condition here */ true);

        Ok(FilterResult { kept, removed })
    }
}
```

2. Register it in `phoenix_candidate_pipeline.rs`:

```rust
fn filters(&self) -> &[Box<dyn Filter<ScoredPostsQuery, PostCandidate>>] {
    &self.filters  // add Box::new(MyNewFilter) to the vec
}
```

### Adding a New Scorer

1. Create the scorer, implementing `score()` and `update()`:

```rust
pub struct MyScorer;

#[async_trait]
impl Scorer<ScoredPostsQuery, PostCandidate> for MyScorer {
    async fn score(&self, query: &ScoredPostsQuery, candidates: &[PostCandidate])
        -> Result<Vec<PostCandidate>, String>
    {
        let scored = candidates.iter().map(|c| {
            PostCandidate {
                score: Some(/* your scoring logic */),
                ..Default::default()  // only set YOUR fields
            }
        }).collect();
        Ok(scored)
    }

    fn update(&self, candidate: &mut PostCandidate, scored: PostCandidate) {
        candidate.score = scored.score;  // only copy YOUR fields
    }
}
```

2. Register it in the scorer chain (order matters -- it runs after previous scorers).

### Adding a New Source

1. Implement the `Source` trait with an async `get_candidates()`:

```rust
pub struct MySource { client: MyClient }

#[async_trait]
impl Source<ScoredPostsQuery, PostCandidate> for MySource {
    async fn get_candidates(&self, query: &ScoredPostsQuery)
        -> Result<Vec<PostCandidate>, String>
    {
        let posts = self.client.fetch(query.user_id).await?;
        Ok(posts.into_iter().map(|p| PostCandidate { /* ... */ }).collect())
    }
}
```

2. Register it in `sources()` -- it will automatically run in parallel with other sources.

## The Concrete Pipeline Wiring

`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` wires the actual X feed pipeline:

```
Query Hydrators:  UserActionSeqQueryHydrator, UserFeaturesQueryHydrator
Sources:          PhoenixSource, ThunderSource
Hydrators:        InNetwork, CoreData, VideoDuration, Subscription, Gizmoduck
Filters:          DropDuplicates, CoreDataHydration, Age, SelfTweet, RetweetDedup,
                  IneligibleSubscription, PreviouslySeen, PreviouslyServed,
                  MutedKeyword, AuthorSocialgraph
Scorers:          PhoenixScorer, WeightedScorer, AuthorDiversityScorer, OONScorer
Selector:         TopKScoreSelector
Post-Selection:   VFCandidateHydrator, VFFilter, DedupConversationFilter
Side Effects:     CacheRequestInfoSideEffect
```

## Code References

| File | What |
|------|------|
| `candidate-pipeline/candidate_pipeline.rs` | Core `CandidatePipeline` trait and `execute()` orchestration |
| `candidate-pipeline/source.rs` | `Source` trait definition |
| `candidate-pipeline/filter.rs` | `Filter` trait and `FilterResult` |
| `candidate-pipeline/scorer.rs` | `Scorer` trait with split-update pattern |
| `candidate-pipeline/hydrator.rs` | `Hydrator` trait |
| `candidate-pipeline/query_hydrator.rs` | `QueryHydrator` trait |
| `candidate-pipeline/selector.rs` | `Selector` trait with default sort implementation |
| `candidate-pipeline/side_effect.rs` | `SideEffect` trait and `SideEffectInput` |
| `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` | Concrete pipeline wiring all components together |
