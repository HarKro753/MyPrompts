---
name: engagement-scoring-weights
description: Guide to the engagement scoring and weight system that determines post ranking in the X feed. Use when asking about scoring, weights, feed tuning, author diversity, how posts get ranked, the weighted scorer, or how to adjust feed behavior.
---

# Engagement Scoring and Weights

This skill explains how X's algorithm turns raw ML predictions into final ranked scores. The scoring system is a **chain of 4 scorers** that run sequentially, each transforming the score further. This is where you tune the "feel" of the feed without retraining the ML model.

## The Scoring Chain

Scorers execute in this exact order. Each one reads from the previous scorer's output and writes to its own field:

```
Phoenix Scorer          Weighted Scorer         Author Diversity        OON Scorer
(ML predictions)   -->  (weighted sum)     -->  (repeated author    --> (in-network vs
                                                 decay)                  out-of-network)
                                                
Writes:                 Writes:                 Writes:                 Writes:
  phoenix_scores          weighted_score          score                   score
  (19 probabilities)      (single number)         (diversity-adjusted)    (source-adjusted)
```

The final `score` field is what the selector sorts by.

## Scorer 1: Phoenix Scorer (ML Predictions)

The Grok-based transformer outputs raw logits for each candidate. The Phoenix Scorer converts these to probabilities via `sigmoid()` and maps them to 19 named fields:

```
Positive engagement signals:          Negative signals:
  favorite_score      (like)            not_interested_score
  reply_score                           block_author_score
  retweet_score       (repost)          mute_author_score
  photo_expand_score                    report_score
  click_score
  profile_click_score
  vqv_score           (video view)
  share_score
  share_via_dm_score
  share_via_copy_link_score
  dwell_score         (time on post)
  quote_score
  quoted_click_score
  follow_author_score

Continuous value:
  dwell_time          (predicted seconds)
```

Each probability is between 0 and 1. These are stored in `candidate.phoenix_scores`.

**Code:** `home-mixer/scorers/phoenix_scorer.rs`

## Scorer 2: Weighted Scorer (The Core Formula)

Combines all 19 predictions into a single number:

```
weighted_score = SUM(weight_i * P(action_i))
```

From `home-mixer/scorers/weighted_scorer.rs`:

```rust
let combined_score =
      apply(s.favorite_score,           FAVORITE_WEIGHT)
    + apply(s.reply_score,              REPLY_WEIGHT)
    + apply(s.retweet_score,            RETWEET_WEIGHT)
    + apply(s.photo_expand_score,       PHOTO_EXPAND_WEIGHT)
    + apply(s.click_score,              CLICK_WEIGHT)
    + apply(s.profile_click_score,      PROFILE_CLICK_WEIGHT)
    + apply(s.vqv_score,                vqv_weight)        // conditional!
    + apply(s.share_score,              SHARE_WEIGHT)
    + apply(s.share_via_dm_score,       SHARE_VIA_DM_WEIGHT)
    + apply(s.share_via_copy_link_score, SHARE_VIA_COPY_LINK_WEIGHT)
    + apply(s.dwell_score,              DWELL_WEIGHT)
    + apply(s.quote_score,              QUOTE_WEIGHT)
    + apply(s.quoted_click_score,       QUOTED_CLICK_WEIGHT)
    + apply(s.dwell_time,               CONT_DWELL_TIME_WEIGHT)
    + apply(s.follow_author_score,      FOLLOW_AUTHOR_WEIGHT)
    + apply(s.not_interested_score,     NOT_INTERESTED_WEIGHT)    // negative weight
    + apply(s.block_author_score,       BLOCK_AUTHOR_WEIGHT)      // negative weight
    + apply(s.mute_author_score,        MUTE_AUTHOR_WEIGHT)       // negative weight
    + apply(s.report_score,             REPORT_WEIGHT);           // negative weight
```

### How `apply()` works

```rust
fn apply(score: Option<f64>, weight: f64) -> f64 {
    score.unwrap_or(0.0) * weight
}
```

If the ML model didn't produce a score (None), it defaults to 0.

### Special Case: Video Quality Views (VQV)

The `vqv_weight` is **conditional** -- it only applies to videos that exceed a minimum duration:

```rust
fn vqv_weight_eligibility(candidate: &PostCandidate) -> f64 {
    if candidate.video_duration_ms.is_some_and(|ms| ms > MIN_VIDEO_DURATION_MS) {
        VQV_WEIGHT
    } else {
        0.0  // no video or too short -- ignore vqv_score entirely
    }
}
```

### Score Offset Normalization

After the weighted sum, `offset_score()` remaps the score so all final values are positive and usable for ranking:

```rust
fn offset_score(combined_score: f64) -> f64 {
    if WEIGHTS_SUM == 0.0 {
        combined_score.max(0.0)
    } else if combined_score < 0.0 {
        // Negative scores: normalize into a small positive range
        (combined_score + NEGATIVE_WEIGHTS_SUM) / WEIGHTS_SUM * NEGATIVE_SCORES_OFFSET
    } else {
        // Positive scores: shift up by the offset
        combined_score + NEGATIVE_SCORES_OFFSET
    }
}
```

This ensures:
- Posts with purely negative signals still get a small positive score (they rank last, not at 0)
- The offset creates a clean separation between "bad" and "good" content
- All scores are positive, which simplifies downstream logic

**Code:** `home-mixer/scorers/weighted_scorer.rs`

## Scorer 3: Author Diversity Scorer

Prevents a single prolific author from dominating the feed. It applies an **exponential decay** to repeated authors:

```
multiplier = (1 - floor) * decay^position + floor
```

Where:
- `position` = how many posts from this author have already appeared (0-indexed)
- `decay` = exponential decay factor (configured via `AUTHOR_DIVERSITY_DECAY`)
- `floor` = minimum multiplier (configured via `AUTHOR_DIVERSITY_FLOOR`)

### How It Works

1. Sort all candidates by `weighted_score` (descending)
2. Walk through in order, tracking how many times each author has appeared
3. First post from an author: `position=0`, `multiplier=1.0` (no penalty)
4. Second post: `position=1`, `multiplier = (1-floor)*decay + floor`
5. Third post: `position=2`, `multiplier = (1-floor)*decay^2 + floor`
6. Apply: `score = weighted_score * multiplier`

### Worked Example

With `decay=0.5` and `floor=0.1`:

| Author's Nth Post | Position | Multiplier | Effect |
|--------------------|----------|------------|--------|
| 1st | 0 | `0.9 * 0.5^0 + 0.1 = 1.0` | Full score |
| 2nd | 1 | `0.9 * 0.5^1 + 0.1 = 0.55` | 55% of score |
| 3rd | 2 | `0.9 * 0.5^2 + 0.1 = 0.325` | 33% of score |
| 4th | 3 | `0.9 * 0.5^3 + 0.1 = 0.2125` | 21% of score |
| 5th+ | 4+ | Approaching `0.1` | Floor: minimum 10% |

The `floor` prevents posts from being completely suppressed -- even a 10th post from the same author retains some score.

**Code:** `home-mixer/scorers/author_diversity_scorer.rs`

## Scorer 4: Out-of-Network (OON) Scorer

Controls the balance between in-network content (from people you follow) and out-of-network content (discovered via Phoenix retrieval):

```rust
let updated_score = match c.in_network {
    Some(false) => base_score * OON_WEIGHT_FACTOR,  // out-of-network: scale
    _ => base_score,                                 // in-network: unchanged
};
```

- `OON_WEIGHT_FACTOR > 1.0` = more discovered content in the feed
- `OON_WEIGHT_FACTOR < 1.0` = more content from people you follow
- `OON_WEIGHT_FACTOR = 1.0` = no preference

**Code:** `home-mixer/scorers/oon_scorer.rs`

## The Split-Update Pattern

Each scorer only writes to **its own fields** via the `update()` method:

| Scorer | Reads | Writes |
|--------|-------|--------|
| PhoenixScorer | ML model response | `phoenix_scores` (19 fields) |
| WeightedScorer | `phoenix_scores` | `weighted_score` |
| AuthorDiversityScorer | `weighted_score` | `score` |
| OONScorer | `score`, `in_network` | `score` |

This pattern prevents one scorer from accidentally overwriting another's data. The pipeline enforces this by calling `scorer.score()` (returns new objects with only its fields set) and then `scorer.update()` (merges only those fields into the original candidate).

## Practical Tuning Guide

| Goal | What to Adjust |
|------|----------------|
| More conversation-starting content | Increase `REPLY_WEIGHT` and `QUOTE_WEIGHT` |
| Less outrage/controversy | Increase magnitude of `BLOCK_AUTHOR_WEIGHT`, `MUTE_AUTHOR_WEIGHT`, `REPORT_WEIGHT` (make more negative) |
| More diverse authors in feed | Decrease `AUTHOR_DIVERSITY_DECAY` (faster decay) or decrease `AUTHOR_DIVERSITY_FLOOR` |
| More discovered content | Increase `OON_WEIGHT_FACTOR` |
| Favor long-form engagement | Increase `DWELL_WEIGHT` and `CONT_DWELL_TIME_WEIGHT` |
| Boost video content | Increase `VQV_WEIGHT`, potentially lower `MIN_VIDEO_DURATION_MS` |
| Boost sharing behavior | Increase `SHARE_WEIGHT`, `SHARE_VIA_DM_WEIGHT`, `SHARE_VIA_COPY_LINK_WEIGHT` |

**All weight values are configured in `home-mixer/params/`** (not included in the open-source release for sensitivity reasons). The architecture lets you change feed behavior by adjusting these constants -- no model retraining required.

## Code References

| File | What |
|------|------|
| `home-mixer/scorers/phoenix_scorer.rs` | Maps ML model output to 19 named probability fields |
| `home-mixer/scorers/weighted_scorer.rs` | Weighted sum formula + offset normalization |
| `home-mixer/scorers/author_diversity_scorer.rs` | Exponential decay for repeated authors |
| `home-mixer/scorers/oon_scorer.rs` | In-network vs out-of-network multiplier |
| `phoenix/runners.py` lines 336-371 | Python-side: logits -> sigmoid -> probability extraction |
| `phoenix/runners.py` lines 202-222 | The 19 action names in canonical order |
