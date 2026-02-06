# Why Your Post Isn't Doing Well — Algorithm Analysis

## Your Post

> **"Unpopular opinion: AI SaaS is just SaaS with better marketing..."**
> - Text-only opinion thread with an embedded diagram image
> - 17 impressions, no visible likes/replies/reposts
> - From a small account (@requique)

---

## The 7 Reasons the Algorithm Is Suppressing Your Post

Based on analysis of the actual X recommendation algorithm source code, here's exactly what's happening:

### 1. Out-of-Network Penalty (×0.75)

**File:** `home-mixer/.../scorer/RescoringFactorProvider.scala` (line 48-57)

Every post shown to users who don't follow you gets a **25% score reduction** automatically:

```scala
object RescoreOutOfNetwork extends RescoringFactorProvider {
  def selector(...) = !candidate.features.getOrElse(InNetworkFeature, false)
  def factor(...) = query.params(OutOfNetworkScaleFactorParam) // default: 0.75
}
```

**Impact on your post:** Since you have few followers, nearly 100% of potential viewers are out-of-network. Your post starts with a 25% handicap compared to posts from accounts people already follow.

### 2. Cold-Start Content Exploration Is Disabled by Default

**File:** `home-mixer/.../param/ScoredTweetsParam.scala` (line 66-70)

The pipeline specifically designed to surface posts from new/small accounts is **turned off**:

```scala
object EnableContentExplorationCandidatePipelineParam
    extends FSParam[Boolean](
      name = "scored_tweets_enable_content_exploration_candidate_pipeline",
      default = false  // <-- OFF by default
    )
```

Even when enabled, content exploration candidates get a **0.0001× multiplier** — effectively zeroing their scores:

```scala
// ContentExplorationListwiseRescoringProvider.scala, line 29
if (servedType matches ContentExploration types) 0.0001  // 99.99% penalty
```

### 3. Empty SimClusters Embedding (Cold Start Problem)

**File:** `src/scala/com/twitter/simclusters_v2/README.md`

X uses SimClusters to match posts to interested users by embedding both into ~145K interest communities. **Tweet embeddings are built from engagement:**

- Each like adds the liker's interest vector to your tweet's embedding
- Each retweet reinforces the embedding
- **With 0 likes, your tweet has no embedding** — it literally can't be matched to interested users

This creates a **chicken-and-egg problem**: you need engagement to get visibility, but you need visibility to get engagement.

### 4. UTEG Collaborative Filter Can't Find You

**File:** `src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`

The User-Tweet Entity Graph (UTEG) recommends out-of-network posts by traversing engagement graphs: "People you follow liked X, so you might like X too."

**Problem:** With no engagement on your post, UTEG has zero signal to work with. Your post never enters the recommendation graph.

### 5. Author Follower Count Is a Model Feature

**File:** `home-mixer/.../model/HomeFeatures.scala` (line 80)

```scala
object AuthorFollowersFeature extends Feature[TweetCandidate, Option[Long]]
```

Your follower count is fed directly into the ML ranking model. The model has learned that posts from accounts with very few followers generate less engagement on average, so it predicts lower engagement probability for your post — creating a self-reinforcing cycle.

### 6. Light Ranking Pre-Filter

**File:** `pushservice/src/main/python/models/light_ranking/`

Before posts even reach heavy ML scoring, a lightweight model pre-filters candidates. This model uses author reputation signals (followers, historical engagement rates). Posts from small accounts with no track record are more likely to be filtered out before they even get a chance to be scored.

### 7. No Feedback Loop Bootstrap

**File:** `unified_user_actions/README.md`

X's real-time pipeline (Unified User Actions) continuously updates tweet scores based on incoming engagement. When someone likes your post:

1. UUA streams the action in real-time
2. SimClusters TweetJob updates your tweet's embedding
3. This makes it matchable to more users
4. More users see it → more engagement → stronger signal

**With 17 impressions and 0 engagements, this virtuous cycle never starts.**

---

## How to Optimize Your Posts

Based on the algorithm's actual code, here are concrete strategies:

### Tier 1: Critical (Address These First)

| Strategy | Why It Works (Algorithm Mechanism) |
|----------|-----------------------------------|
| **Build your follow graph first** | Eliminates the 0.75× out-of-network penalty. In-network posts get full scoring. |
| **Get early engagement within first minutes** | Bootstraps SimClusters embeddings and UTEG graph entry. Even 2-3 likes from accounts with followers can cascade. |
| **Reply to large accounts in your niche** | Your replies become visible to their followers. The algorithm tracks reply engagement as a signal (`ReplyParam` model weight). |

### Tier 2: Content Optimization

| Strategy | Why It Works |
|----------|-------------|
| **Provoke replies, not just likes** | The model weights replies (`ReplyParam`), favorites (`FavParam`), AND "good clicks" (`GoodClickV1Param` = click + engagement). Replies generate the most downstream signals. |
| **Use images/media strategically** | Media features are tracked separately (`TweetMediaClusterIdsFeature`, `ClipImageClusterIdsFeature`). Media content gets separate hydration and scoring. But avoid repetitive media — the algorithm deduplicates similar images. |
| **Ask a question or create controversy** | The algorithm heavily weights `DwellParam` (time spent on tweet) and reply engagement. Controversial takes drive both. Your "unpopular opinion" framing is good — but needs more engagement hooks. |
| **Avoid being a reply** | Replies get a 0.75× score penalty (`ReplyScaleFactorParam`). Post as standalone threads, not replies. |

### Tier 3: Timing and Distribution

| Strategy | Why It Works |
|----------|-------------|
| **Post when your niche audience is active** | UTEG candidate generation uses a 24-48 hour window. Posts that get engagement quickly have more time in the recommendation pipeline. |
| **Engage authentically before posting** | The algorithm tracks bidirectional engagement. If you like/reply to others, their followers are more likely to see your content via UTEG collaborative filtering. |
| **Share your post to DMs/external** | Shares (`ShareParam`) and bookmarks (`BookmarkParam`) are positive engagement signals. External traffic → engagement → embedding → algorithmic reach. |

### Tier 4: Specific to Your Post

| Issue | Recommendation |
|-------|---------------|
| **"AI SaaS" is niche** | SimClusters needs to match your post to the AI/SaaS interest community. Tag/mention relevant accounts to seed the topic signal. |
| **Diagram image with no alt-text/context** | The algorithm uses CLIP embeddings for images. Ensure your image has clear, recognizable content that matches the topic. |
| **17 impressions = never entered recommendation** | Your post was likely only shown to your followers (in-network). It never crossed into out-of-network recommendation because no engagement signals existed to trigger UTEG or SimClusters. |
| **Thread structure** | Multi-line posts can be threaded — but the algorithm penalizes replies (0.75×). Structure as a single post with line breaks, not a self-reply thread. |

---

## The Core Problem

Your post's core issue isn't content quality — it's **distribution**. The algorithm is designed around a **rich-get-richer** feedback loop:

```
Followers → In-network impressions → Engagement → Embeddings → Out-of-network reach → More engagement
```

With few followers, you never enter this loop. The algorithm has multiple compounding penalties:
- 0.75× for out-of-network
- 0.0001× for content exploration candidates  
- Empty SimClusters embedding (unmatchable)
- No UTEG graph presence
- Low author reputation features

**Ironically, your post's thesis is proven by the algorithm itself**: "The model isn't the moat. Distribution + workflow ownership is." The algorithm rewards distribution (followers, engagement graphs) far more than content quality.

---

## Key Source Files Referenced

| File | What It Controls |
|------|-----------------|
| `ScoredTweetsParam.scala` | All scoring parameters and feature flags |
| `RescoringFactorProvider.scala` | Score multipliers (out-of-network, replies, MTL, feedback) |
| `NaviModelScorer.scala` | ML engagement prediction and weighted scoring |
| `HeuristicScorer.scala` | Rescoring pipeline orchestration |
| `ContentExplorationListwiseRescoringProvider.scala` | Cold-start content penalties |
| `AuthorBasedListwiseRescoringProvider.scala` | Author diversity decay |
| `ImpressedAuthorDecayRescoringProvider.scala` | Impression fatigue |
| `HomeFeatures.scala` | Feature definitions (follower count, engagement, etc.) |
| `RETREIVAL_SIGNALS.md` | Candidate sourcing signal mapping |
