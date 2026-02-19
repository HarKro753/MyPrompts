---
name: social-media-algorithm
description: Guide for designing and building a social media recommendation algorithm. Use when the user asks about building a feed ranking system, content recommendation engine, for-you feed, timeline algorithm, or social media ranking pipeline. Covers retrieval, ranking, filtering, and scoring architecture.
---

# How to Build a Social Media Recommendation Algorithm

This skill describes the high-level architecture for building a social media feed algorithm, based on the patterns used in X's "For You" feed (the Phoenix/Thunder system in this repository).

## Overview

A social media algorithm answers one question: **given a user, which posts should they see and in what order?**

The system has two fundamental problems to solve:

1. **Retrieval**: From millions of posts, find the ~1000 that _might_ be relevant
2. **Ranking**: From those ~1000, predict exactly how likely the user is to engage with each one, then sort by score

Everything else (filtering, diversity, safety) wraps around these two core operations.

## Architecture: The Candidate Pipeline

The entire system is a **pipeline** that processes candidates through stages. Each stage has a single responsibility and a clean interface:

```
User Request
    |
    v
[1. Query Hydration]     -- Fetch user context (who they follow, engagement history)
    |
    v
[2. Candidate Sourcing]  -- Get posts from multiple sources in parallel
    |
    v
[3. Candidate Hydration] -- Enrich posts with metadata (author info, media, etc.)
    |
    v
[4. Pre-Scoring Filters] -- Remove ineligible posts (blocked authors, duplicates, etc.)
    |
    v
[5. Scoring]             -- ML model predicts engagement, then weighted combination
    |
    v
[6. Selection]           -- Sort by score, pick top K
    |
    v
[7. Post-Selection]      -- Final safety/visibility checks
    |
    v
Ranked Feed
```

### Key Design Principle: Composability

Each stage is defined as a **trait/interface** (Source, Hydrator, Filter, Scorer, Selector, SideEffect). New behavior is added by implementing a trait and plugging it in -- not by modifying existing code. Sources and hydrators run in **parallel**. Filters and scorers run **sequentially** (order matters).

## Stage 1: Query Hydration

Before fetching any posts, gather everything you know about the requesting user:

- **Engagement history**: The last N posts they interacted with (liked, replied, reposted, etc.)
- **Social graph**: Who they follow, who they've blocked/muted
- **Preferences**: Muted keywords, content settings
- **Session context**: What posts were already shown in this session

This context becomes the input to the ML model later.

## Stage 2: Candidate Sourcing

Posts come from **two sources** that run in parallel:

### In-Network Source (like Thunder)

Posts from accounts the user follows. This is a simple lookup:

- Maintain an **in-memory store** of recent posts per user
- Ingest post create/delete events in real-time (e.g., from Kafka)
- When a user requests their feed, look up posts from everyone they follow
- Sub-millisecond latency since everything is in memory

### Out-of-Network Source (like Phoenix Retrieval)

Posts from accounts the user does NOT follow, discovered via ML. This uses a **two-tower model**:

```
   User Tower                    Candidate Tower
   (Transformer)                 (MLP)
       |                              |
       v                              v
   User Embedding [D]          Post Embeddings [N, D]
       |                              |
       +---------> Dot Product <------+
                      |
                      v
                Top-K Post IDs
```

- **User tower**: Encodes user features + engagement history through a transformer, then mean-pools and L2-normalizes to get a single user embedding vector
- **Candidate tower**: Projects each post's features (post content + author info) through an MLP, L2-normalizes
- **Retrieval**: Dot product similarity between user embedding and all candidate embeddings, return top-K
- Both embeddings are L2-normalized so dot product = cosine similarity
- In production, use **approximate nearest neighbor** (ANN) search for speed at scale

## Stage 3: Candidate Hydration

Enrich each candidate post with additional data by calling various services in parallel:

- Core post metadata (text, media, timestamps)
- Author information (username, verification status)
- Video duration (for video-specific scoring)
- Subscription status (for paywalled content)
- Visibility/safety labels

Each hydrator only writes to its own fields -- they don't conflict, so they can run in parallel.

## Stage 4: Filtering (Pre-Scoring)

Remove posts that should never be shown, **before** spending compute on ML scoring:

| Filter                   | Purpose                                       |
| ------------------------ | --------------------------------------------- |
| Duplicate removal        | Same post ID appearing twice                  |
| Age filter               | Posts older than threshold                    |
| Self-post filter         | User's own posts                              |
| Blocked/muted authors    | Posts from accounts user has blocked or muted |
| Muted keywords           | Posts containing user's muted terms           |
| Previously seen          | Posts already shown to this user              |
| Previously served        | Posts already served in this session          |
| Failed hydration         | Posts where metadata fetch failed             |
| Subscription eligibility | Paywalled content user can't access           |

## Stage 5: Scoring (The Heart of the Algorithm)

Scoring runs as a chain of scorers, each building on the previous:

### Scorer 1: ML Engagement Prediction (Phoenix Ranking Model)

This is the core ML model -- a **transformer** that predicts engagement probabilities.

#### Inputs (what the model sees)

The model receives a rich representation of the user and candidates:

| Input              | Description                                                                                    |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| User identity      | Hash-based embedding of the user                                                               |
| Engagement history | Last N posts the user interacted with (post embeddings + author embeddings)                    |
| Actions taken      | What the user did with each history post (liked, replied, blocked, etc.) -- a multi-hot vector |
| Product surface    | Where the user saw each post (home feed, search, notifications)                                |
| Candidate posts    | The posts to score (post embeddings + author embeddings)                                       |

All inputs are projected to the same embedding dimension and concatenated into a single sequence:

```
[User] [History_1] [History_2] ... [History_N] [Candidate_1] [Candidate_2] ... [Candidate_C]
```

#### The Candidate Isolation Attention Mask (Critical Design Decision)

The transformer uses a **special attention mask** that prevents candidates from attending to each other:

- User + History: Normal causal attention among themselves
- Candidates -> User/History: Each candidate CAN attend to all user/history positions
- Candidates -> Candidates: Each candidate can ONLY attend to itself (NOT other candidates)

```
         User  History  Candidates
User   [  ok    --       --     ]
Hist   [  ok    ok       --     ]
Cands  [  ok    ok    self-only ]
```

**Why this matters**: It guarantees that a candidate's score is **independent** of which other candidates are in the batch. Without this, reordering or adding/removing a candidate would change every other candidate's score -- making the system unpredictable and uncacheable.

#### Outputs (what the model predicts)

For each candidate, the model outputs **probabilities for ~19 engagement types**:

```
Positive signals:              Negative signals:
  P(favorite/like)               P(not_interested)
  P(reply)                       P(block_author)
  P(repost)                      P(mute_author)
  P(quote)                       P(report)
  P(click)
  P(profile_click)
  P(video_view)
  P(photo_expand)
  P(share)
  P(dwell/time_spent)
  P(follow_author)
```

#### Why Multi-Action Prediction Matters

Predicting many actions instead of a single "relevance" score gives you:

- **Tunability**: Change feed behavior by adjusting weights, not retraining the model
- **Transparency**: You can see WHY a post scores high (engagement vs. outrage)
- **Negative signals**: Content that triggers blocks/mutes/reports gets penalized directly

### Scorer 2: Weighted Combination

Combine the ML predictions into a single score:

```
Final Score = SUM(weight_i * P(action_i))
```

- Positive actions (like, share, reply) get **positive** weights
- Negative actions (block, mute, report) get **negative** weights
- This is where you tune the "feel" of the feed without retraining the model
- Example: Increase reply weight to favor conversation-starting posts

### Scorer 3: Author Diversity

Attenuate scores for repeated authors. If someone you follow posts 20 times, you don't want 20 posts from them at the top. Apply a decay factor for each subsequent post from the same author.

### Scorer 4: Source-Specific Adjustments

Optionally adjust scores based on where the candidate came from (in-network vs. out-of-network). This lets you tune the balance between content from people you follow vs. discovered content.

## Stage 6: Selection

Sort all candidates by their final score and select the top K.

## Stage 7: Post-Selection Processing

Final safety checks on the selected posts:

- Visibility filtering (deleted, spam, violence, gore)
- Conversation deduplication (don't show multiple branches of the same thread)

## Key Architecture Decisions

### 1. No Hand-Engineered Features

Let the transformer learn relevance from raw engagement sequences. No manual feature engineering for "what makes a good post." This dramatically simplifies data pipelines and makes the system easier to maintain.

### 2. Hash-Based Embeddings

Use multiple hash functions to map user IDs, post IDs, and author IDs to embedding table indices. This handles the open vocabulary problem (new users/posts) without maintaining a mapping table, and provides collision-based regularization.

### 3. Separation of Retrieval and Ranking

Retrieval is fast and approximate (dot product over precomputed embeddings). Ranking is slow and precise (full transformer forward pass). This two-stage design lets you cover a huge candidate space while spending expensive compute only on promising candidates.

### 4. Pipeline as Framework

The candidate pipeline is a generic framework. Business logic (specific filters, scorers, sources) plugs in via traits/interfaces. This means:

- Adding a new filter = implement one trait, register it
- Adding a new signal = add a scorer to the chain
- Changing behavior = adjust weights or swap components, don't rewrite the pipeline

## How to Get Started Building Your Own

1. **Start with the pipeline framework**: Define your stage interfaces (Source, Filter, Scorer, Selector)
2. **Build the in-network source first**: Simple in-memory post store for followed accounts
3. **Add basic filters**: Duplicates, blocked users, age cutoff
4. **Build the ranking model**: Start with a simpler model (even logistic regression on hand-crafted features), then graduate to a transformer
5. **Add the retrieval model**: Two-tower architecture for out-of-network discovery
6. **Tune with weights**: Adjust engagement type weights to shape feed behavior
7. **Add diversity and safety**: Author diversity scorer, visibility filtering

The ranking model is the hardest part. Everything else is engineering plumbing around it. Start simple and iterate.
