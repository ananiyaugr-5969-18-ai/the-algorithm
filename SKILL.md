# SKILL: Creating Highly Engaging Content on X

This document provides guidance on creating highly engaging content on X, derived directly from the open-source recommendation algorithm code in this repository. Each section describes what the algorithm actually does—based on source code and READMEs—so you can understand the mechanics rather than just follow vague tips.

---

## Table of Contents

1. [How the Algorithm Works – Full Pipeline](#1-how-the-algorithm-works--full-pipeline)
2. [Candidate Sources: How Your Content Gets Discovered](#2-candidate-sources-how-your-content-gets-discovered)
3. [Engagement Signals That Drive Reach](#3-engagement-signals-that-drive-reach)
4. [Ranking: Light Ranker and Heavy Ranker](#4-ranking-light-ranker-and-heavy-ranker)
5. [Content Quality Factors (Static Signals)](#5-content-quality-factors-static-signals)
6. [Health and Safety Filtering](#6-health-and-safety-filtering)
7. [Rich Content Features](#7-rich-content-features)
8. [Author Credibility: Tweepcred and UserMass](#8-author-credibility-tweepcred-and-usermass)
9. [Network Graphs: Real Graph and SimClusters](#9-network-graphs-real-graph-and-simclusters)
10. [Out-of-Network Discovery: UTEG, UUG, and FRS](#10-out-of-network-discovery-uteg-uug-and-frs)
11. [Engagement Quality Filters (Ratio-Based)](#11-engagement-quality-filters-ratio-based)
12. [Language Matching for Out-of-Network Content](#12-language-matching-for-out-of-network-content)
13. [Feedback Fatigue: How Negative Feedback Suppresses Future Posts](#13-feedback-fatigue-how-negative-feedback-suppresses-future-posts)
14. [Notifications: A Separate Recommendation Pipeline](#14-notifications-a-separate-recommendation-pipeline)
15. [Diversity and Content Balance Filters](#15-diversity-and-content-balance-filters)
16. [Visibility Filtering: What Can Get Your Content Suppressed](#16-visibility-filtering-what-can-get-your-content-suppressed)
17. [Recommendations for Creators](#17-recommendations-for-creators)
18. [Further Reading](#18-further-reading)

---

## 1. How the Algorithm Works – Full Pipeline

X's For You Timeline recommendation pipeline is described in [`home-mixer/README.md`](home-mixer/README.md). The full pipeline runs as follows:

### Stage 1: Candidate Generation

Multiple candidate sources are queried in parallel to retrieve thousands of candidate tweets from billions available. Candidate sources include:

- **Earlybird Search Index** – Fetches in-network tweets (from accounts you follow) using a real-time distributed inverted index. About 50% of For You Timeline posts come from this source.
- **CR-Mixer** (Candidate Retrieval Mixer) – A coordination layer that fetches out-of-network tweet candidates from compute services including SimClusters ANN, UTEG, and others. Acts as a lightweight delegator that caches results and performs light filtering.
- **User Tweet Entity Graph (UTEG)** – Surfaces out-of-network tweets that users in your network have liked. This is the source behind the "X Liked" posts in your feed.
- **Follow Recommendations Service (FRS)** – Recommends tweets from accounts you might want to follow in the future (FutureGraph tweets), scored on the probability of following and engaging.

### Stage 2: Feature Hydration

Each candidate tweet has approximately **6,000 features** computed for it. These features are pulled from multiple sources:

- Static features computed at index time (URL presence, media types, trend words, reply/retweet flag)
- Real-time engagement counts updated continuously (likes, retweets, replies, quote counts)
- Author features: Tweepcred (reputation score), follower count, engagement history
- Viewer-specific features: Real Graph score between viewer and author, language match, SimClusters embedding similarity
- Health model scores: toxicityScore, pBlockScore, pReportedTweetScore, pSpammyTweetScore, spammyTweetContentScore

### Stage 3: Ranking

- **Light Ranker (Earlybird)** – A logistic regression model that scores the larger pool of candidates. Predicts likelihood of engagement. Runs on the order of hundreds of thousands of candidates.
- **Heavy Ranker** – A deep neural network (implemented via the heavy-ranker model at [the-algorithm-ml](https://github.com/twitter/the-algorithm-ml)) that produces the final ranking scores. Receives the output of the light ranker as input.

### Stage 4: Filters and Heuristics

After scoring, a set of filters and heuristics is applied before content reaches your feed:

- **Author diversity** – Limits how many posts from the same author appear in your feed in a single refresh.
- **Content balance** – Controls the ratio of in-network vs. out-of-network content.
- **Feedback fatigue** – Suppresses content from authors or accounts whose content you previously marked "See Fewer".
- **Deduplication** – Removes posts you have already seen.
- **Visibility filtering** – Applies safety, legal compliance, and user-preference filters (muted/blocked accounts, NSFW settings).

### Stage 5: Mixing

Scored, filtered tweets are combined with non-tweet content:

- Ads
- Who-To-Follow (WTF) user recommendation modules (powered by FRS)
- Prompts and conversation modules

---

## 2. Candidate Sources: How Your Content Gets Discovered

Your posts can enter the recommendation pipeline through several distinct candidate sources. Understanding each one helps you understand how your content can reach new audiences.

### 2.1 Earlybird Search Index (In-Network)

Described in [`src/java/com/twitter/search/README.md`](src/java/com/twitter/search/README.md).

Earlybird is a real-time distributed search system based on Apache Lucene. It maintains three clusters:

- **Realtime cluster** – Indexes all public tweets posted in approximately the last 7 days.
- **Protected cluster** – Indexes protected tweets for the same 7-day window.
- **Archive cluster** – Indexes all tweets ever posted, up to about 2 days ago.

The index is sharded across multiple partitions, replicated for fault tolerance, and updated incrementally in real time as new tweets are posted. A **Feature Update Service** continuously pushes updated engagement counts (likes, retweets, replies) into the index, which means your post's like/retweet count is reflected in ranking within seconds or minutes.

When the timeline is requested, Earlybird Roots fan out the request to all partitions simultaneously, merge the results, and return ranked candidates to Home Mixer. This is the primary source for posts from accounts you follow.

**What this means for you:** Your in-network posts are indexed in real time. Their engagement counts (likes, retweets, replies) are also updated in real time inside the index, so a post that gains early engagement rapidly rises in relevance scores for viewers who follow you.

### 2.2 User Tweet Entity Graph (UTEG) – "X Liked" Out-of-Network

Described in [`src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`](src/scala/com/twitter/recos/user_tweet_entity_graph/README.md).

UTEG is a GraphJet-based in-memory service that maintains a live graph of user-tweet engagement relationships. It performs **collaborative filtering**: given a viewer's follow graph, it finds tweets that users in that follow graph have recently liked. These are the out-of-network posts labeled "X Liked" in your For You feed.

UTEG maintains a sliding window of **24–48 hours** of engagement data. Events older than this are evicted from memory.

**What this means for you:** If people who follow your audience like your post within the past 24–48 hours, your post will be surfaced to that audience through UTEG even if they don't follow you. High, fast like velocity directly drives UTEG reach.

### 2.3 User User Graph (UUG) – Based on Who Your Follows Followed

Described in [`src/scala/com/twitter/recos/README.md`](src/scala/com/twitter/recos).

UUG is a GraphJet-based service that surfaces user recommendations based on who people in your follow graph have recently followed. It maintains a sliding window of roughly **the past week** of follow events.

**What this means for you:** If many accounts that follow your audience have recently started following you, UUG will surface your account as a "Who to Follow" recommendation to that audience.

### 2.4 User Video Graph (UVG) – Video Recommendation

Also GraphJet-based, UVG maintains user-video engagement relationships with a **24–48 hour** window. It provides video-specific tweet recommendations via random walks over the engagement graph. This is specifically used for video content recommendations on the Home timeline.

**What this means for you:** Video content has its own dedicated recommendation pathway. Videos that accumulate watch events quickly will be spread through UVG to other viewers with similar video engagement histories.

### 2.5 CR-Mixer (Candidate Retrieval Mixer)

Described in [`cr-mixer/README.md`](cr-mixer/README.md).

CR-Mixer is a centralized candidate generation layer that aggregates output from compute services including SimClusters ANN, UTEG, and others. Its pipeline is:

1. **Source Signal Extraction** – Pulls user signals from UserProfileService, Real Graph, and other sources.
2. **Candidate Generation** – Calls external compute services and caches results to reduce latency.
3. **Filtering** – Deduplication, health filtering (using `isPassTweetHealthFilterStrict`), and pre-ranking filters.
4. **Light Ranking** – A lightweight ranking step before handing off to the main Home Mixer pipeline.

### 2.6 Follow Recommendations Service (FRS) – FutureGraph Tweets

Described in [`follow-recommendations-service/README.md`](follow-recommendations-service/README.md).

FRS generates both account recommendations (Who To Follow) and **FutureGraph tweet recommendations** (posts from accounts you might follow in the future). The ranking score is a weighted combination of:

- `p(follow | recommendation)` – Probability the viewer will follow the author
- `p(positive engagement | follow)` – Probability the viewer will engage with the author's posts after following

FRS uses ML features hydrated from multiple sources, social proof (e.g., "followed by X user"), and multiple candidate generation algorithms to personalize these recommendations per display location.

**What this means for you:** Growing your follower base among accounts that are broadly influential causes FRS to surface your content as FutureGraph tweets to their audiences.

### 2.7 SimClusters ANN (Community-Based Retrieval)

Described in [`src/scala/com/twitter/simclusters_v2/README.md`](src/scala/com/twitter/simclusters_v2/README.md).

SimClusters ANN finds tweets whose community embeddings are similar to a user's interest embedding. It is powered by pre-built index jobs run in BigQuery:

- **PushOpenBased SimClusters ANN Index** – Built from push notification open history. Used as a candidate source for Notifications.
- **VideoViewBased SimClusters ANN Index** – Built from video view history. Used for video recommendations on Home.

---

## 3. Engagement Signals That Drive Reach

The algorithm uses these signals as both training labels (to train the ranking models) and real-time features (evaluated at serving time). All signals are collected and centralized by the **User Signal Service (USS)**, described in [`user-signal-service/README.md`](user-signal-service/README.md), and unified as a real-time event stream by the **Unified User Actions (UUA)** pipeline, described in [`unified_user_actions/README.md`](unified_user_actions/README.md).

### 3.1 Positive Signals (increase reach)

The table below shows each signal and which systems use it, based on [`RETREIVAL_SIGNALS.md`](RETREIVAL_SIGNALS.md):

| Signal | Description | USS | SimClusters | TwHIN | UTEG | FRS | Light Ranking |
|---|---|---|---|---|---|---|---|
| **Likes (Favorites)** | Clicking the like button | Features | Features | Features + Labels | Features | Features + Labels | Features + Labels |
| **Retweets** | Reposting to followers | Features | — | Features + Labels | Features | Features + Labels | Features + Labels |
| **Quote Tweets** | Repost with commentary | Features | — | Features + Labels | Features | Features + Labels | Features + Labels |
| **Replies** | Responding to a post | Features | — | Features | Features | Features + Labels | Features |
| **Bookmarks** | Saving a post | Features | — | — | — | — | — |
| **Post Clicks** | Viewing the detail page | Features | — | — | — | Features | Labels |
| **Video Watch** | Watching video content | Features | Features | — | — | — | Labels |
| **Notification Opens** | Opening a push notification | Features | Features | Features | — | Features | — |
| **Ntab Clicks** | Clicking on Notifications tab | Features | Features | Features | — | Features | — |
| **Profile Visits** | Visiting the author's profile | (tracked via USS) | — | — | — | — | — |
| **Follows** | Following after seeing content | Features | Features + Labels | Features + Labels | Features | Features + Labels | — |

**Notes on signal weight:**
- Signals marked as **Labels** are used directly to train the ranking models—they are what the model is optimizing to predict. Signals marked as **Features** are inputs the model uses to make its predictions.
- Likes are the most broadly used signal across all systems. Getting likes quickly after posting is the single highest-leverage action.
- Tweet Clicks (reading the full post) are used as a **label** for the light ranker—meaning the model is specifically trying to predict whether a viewer will click through to read your post in full. Writing a compelling hook that makes people want to read more directly optimizes for this label.
- Video Watch time is also a **label** for the light ranker, meaning the model is trained to predict how long people will watch your video. Longer average watch time increases your video's ranking score.
- **Notification Opens** feed the SimClusters push-open-based ANN index. Getting people to open notifications about your posts strengthens your profile in that index.

### 3.2 Negative Signals (reduce reach)

| Signal | Description | Effect |
|---|---|---|
| **"Not interested"** (`SeeFewer`) | User clicks "Not interested in this tweet" | Triggers Feedback Fatigue filter (see Section 13); suppresses your content for that user for a configurable period |
| **Reports** | User reports your post | Adds to your `pReportedTweetScore`; if score crosses a threshold, your content is filtered from recommendations |
| **Unfollows** | User unfollows you after seeing your post | Recorded in USS; negative training signal for FRS |
| **Mutes** | User mutes your account | Your posts are filtered from that user's timeline |
| **Blocks** | User blocks your account | Your posts are permanently excluded from that user's recommendations; also counts against `pBlockScore` |
| **Tweet Unfavorite** | User removes a like | Tracked in USS and SimClusters as a negative signal |

---

## 4. Ranking: Light Ranker and Heavy Ranker

### 4.1 The Earlybird Light Ranker

Described in [`src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/README.md`](src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/README.md).

The Earlybird light ranker is a **logistic regression model** that predicts the probability a user will engage with a tweet. It is designed to be fast enough to score hundreds of thousands of candidates before passing a smaller subset to the more expensive heavy ranker.

There are two separate light ranker models:

- **`recap_earlybird`** – Used for in-network tweets (posts from accounts you follow)
- **`rectweet_earlybird`** – Used for out-of-network tweets (UTEG candidates)

The feature pipeline for the light ranker includes four types of inputs:

1. **Static features** (computed once at index time, stored in the index):
   - Whether the tweet is a retweet
   - Whether the tweet contains a link (URL)
   - Whether the tweet has any trend words at ingestion time
   - Whether the tweet is a reply
   - Text quality score: computed from offensiveness, content entropy, "shout" score (all-caps), length, and readability
   - Tweepcred (author reputation score)

2. **Realtime features** (updated continuously via Feature Update Service):
   - Number of likes, replies, retweets (updated within seconds/minutes)
   - pToxicity and pBlock scores from health models

3. **User table features** (per-author features propagated to the tweet being scored):
   - Author language preferences
   - Author engagement patterns
   - Author reputation signals

4. **Search context features** (per-viewer, per-request context):
   - Viewer's UI language
   - Viewer's consumed and produced language
   - Current time (to capture recency effects)

### 4.2 The Heavy Ranker

The heavy ranker is a deep neural network trained as a multi-task learning model. It takes the ~6,000 hydrated features as input and predicts the probability of multiple engagement outcomes simultaneously. The final ranking score is a weighted sum of these predicted probabilities.

### 4.3 Notification Heavy Ranker

Described in [`pushservice/src/main/python/models/heavy_ranking/README.md`](pushservice/src/main/python/models/heavy_ranking/README.md).

Notifications have their own separate 4-stage pipeline:

1. **Candidate generation** – Multiple candidate sources queried
2. **Light ranking** – A lightweight MLP model pre-selects highly-relevant candidates from the large candidate pool, reducing cost before heavy ranking
3. **Heavy ranking** – A multi-task learning model (ClemNet, implemented in `pushservice/src/main/python/models/heavy_ranking/lib/model.py`) predicts two outcomes:
   - **p(notification open | send)** – Will the user open the notification?
   - **p(engagement | notification open)** – Will the user engage after opening?
4. **Quality control** – Post-ranking filters including engagement ratio filters, language match, health score thresholds

---

## 5. Content Quality Factors (Static Signals)

These features are computed at **index time** (when your tweet is first ingested) and stored in the Earlybird index. They are evaluated once and affect every ranking decision for the lifetime of the tweet.

### 5.1 Text Quality Score

The text quality score is computed by `TweetTextScorer.java` in the Earlybird Ingester. It evaluates:

- **Offensiveness** – Language that may violate community guidelines or be perceived as offensive
- **Content entropy** – A measure of how much meaningful, varied information is in the text. Low-entropy posts (e.g., single words, repeated characters, random content) score lower
- **"Shout" score** – Proportion of characters in UPPERCASE. Excessive use of all-caps is penalized because it correlates with lower-quality content
- **Length** – Both very short posts (low information) and very long posts can receive lower scores
- **Readability** – Sentence structure, vocabulary, and clarity

### 5.2 Static Structural Features

- **`isReply`** – Whether the tweet is a reply to another tweet. Replies are scored differently from top-level posts; top-level posts generally receive wider distribution.
- **`isRetweet`** – Retweets are tracked distinctly from original posts. Original content receives stronger author credibility signals.
- **`hasUrl`** – Whether the tweet contains a URL. Posts with links have dedicated features in both the light ranker and in tweet_info thrift schema.
- **`hasImage` / `hasVideo` / `hasGif`** – Media presence flags stored in the `TspTweetInfo` thrift struct and used throughout the pipeline.
- **`hasMultipleMedia`** – Whether the tweet contains multiple media items.
- **`isHighMediaResolution`** – High-resolution media is tracked as a distinct flag.
- **`isVerticalAspectRatio`** – Vertical video/image is tracked separately, relevant for mobile-optimized content.
- **Trend words at ingestion time** – Whether any of the tweet's words match current trending topics at the moment of indexing.

### 5.3 Realtime Engagement Signals

These are fed into the Earlybird index continuously by the Feature Update Service:

- **`retweetCount`** – Number of retweets
- **`favCount`** – Number of likes
- **`replyCount`** – Number of replies
- **`quoteCount`** – Number of quote tweets

These counts are reflected in the light ranker's scoring within seconds to minutes of the engagement event occurring.

---

## 6. Health and Safety Filtering

The algorithm applies multiple layers of health and safety filtering at different points in the pipeline. Understanding these helps you avoid having your content silently suppressed.

### 6.1 Health Score Suite

Content is evaluated against the following health models, as seen in [`cr-mixer/server/src/main/scala/com/twitter/cr_mixer/filter/UtegHealthFilter.scala`](cr-mixer/server/src/main/scala/com/twitter/cr_mixer/filter/UtegHealthFilter.scala) and [`home-mixer/server/src/main/scala/com/twitter/home_mixer/util/earlybird/EarlybirdResponseUtil.scala`](home-mixer/server/src/main/scala/com/twitter/home_mixer/util/earlybird/EarlybirdResponseUtil.scala):

| Score | Description | Effect if High |
|---|---|---|
| **`toxicityScore`** | From the `pToxicity` model; detects toxic language including insults and certain types of harassment | Content filtered from out-of-network recommendations |
| **`pBlockScore`** | Predicts probability that users will block the author | Content filtered from recommendations |
| **`pReportedTweetScore`** | Probability that the tweet will be reported | Content filtered from recommendations |
| **`pSpammyTweetScore`** | Probability that the tweet is spam | Content filtered via `isPassTweetHealthFilterStrict` |
| **`spammyTweetContentScore`** | Content-based spam score (distinct from behavioral spam) | Content filtered via `isPassTweetHealthFilterStrict` |

The **strict health filter** (`isPassTweetHealthFilterStrict`) checks all five of these scores together. Out-of-network content must pass this filter to be surfaced in the For You timeline at all.

### 6.2 Trust and Safety Models

The Trust and Safety team trains and maintains the following models, described in [`trust_and_safety_models/README.md`](trust_and_safety_models/README.md):

- **`pNSFWMedia`** – Detects NSFW images (adult/porn content)
- **`pNSFWText`** – Detects NSFW text content (adult/sexual topics)
- **`pToxicity`** – Detects toxic tweets (insults, certain types of harassment; not necessarily a TOS violation)
- **`pAbuse`** – Detects abusive content, including hate speech, targeted harassment, and abusive behavior (TOS violations)

Accounts flagged by these models have their content suppressed across the recommendation pipeline.

### 6.3 Author Safety Flags

From the `TspTweetInfo` thrift schema and author feature hydration:

- **`isNsfwAuthor`** – Whether the author's account is marked NSFW. Posts from NSFW authors are filtered in contexts without explicit content enabled.
- **`isKGODenylist`** – Whether the author is on the Known Good/Official denylist.
- **Restricted accounts** – A `restricted` safety flag reduces a user's mass score to 10% of its normal value in the Tweepcred pipeline.
- **Suspended accounts** – Suspended accounts receive a mass of 0 in the Tweepcred pipeline. Their content is not indexed or distributed.

---

## 7. Rich Content Features

### 7.1 Media Type Detection

The following media flags are stored in the `TspTweetInfo` thrift struct and used throughout the ranking and filtering pipeline:

- `hasImage` – Photo present
- `hasVideo` – Video present
- `hasGif` – GIF present
- `hasMultipleMedia` – Multiple media items attached
- `isHighMediaResolution` – High-resolution media
- `isVerticalAspectRatio` – Vertical-format content (optimized for mobile)
- `videoDurationSeconds` – Video length in seconds

### 7.2 Video Duration Filtering

The Home Mixer pipeline implements configurable minimum and maximum video duration filters:

- **`MinVideoDurationThresholdParam`** – Default: 0 seconds. Can be tuned to exclude very short video clips.
- **`MaxVideoDurationThresholdParam`** – Default: up to 7 days (604,800,000 ms). In practice, much shorter thresholds may be active in production.

Videos within the configured duration range are surfaced through both the main Home timeline and the User Video Graph (UVG) recommendation pathway.

**Media tweets may also skip the language filter** for out-of-network content (see the `skipLanguageFilterForMediaTweets` flag in Section 12), giving media posts a broader distribution than text-only posts.

### 7.3 Link Cards

Posts with URLs trigger card rendering (rich link previews). The `hasUrl` field is tracked as a distinct feature. Posts with link cards are treated as a distinct content type in the feature pipeline, and the visual prominence of the card drives higher click-through rates (which are directly used as Labels for the light ranker).

### 7.4 Trend Words

At index time, Earlybird checks whether any words in your tweet match currently trending topics. Trend matches boost relevance scores for users who are engaging with those trends. This is a one-time check at indexing—tweets that happen to contain trend words at the exact moment of posting benefit from this.

### 7.5 Quote Tweets

Quote tweets are tracked separately from regular retweets. They carry both the original author's content and the quoter's commentary. The number of quote tweets your post receives (`quoteCount`) is a real-time feature used in ranking, and the **QT-to-NtabClick ratio** is also used as a quality filter (see Section 11). Quote tweets therefore represent a double-edged engagement: they boost visibility but can also trigger quality filters if the ratio of quote tweets to notification opens becomes anomalously high (which can indicate controversial content).

---

## 8. Author Credibility: Tweepcred and UserMass

### 8.1 Tweepcred

Tweepcred is X's PageRank-based author reputation system, implemented in [`src/scala/com/twitter/graph/batch/job/tweepcred/`](src/scala/com/twitter/graph/batch/job/tweepcred/README). It runs as a batch job on Hadoop using the MapReduce framework.

The pipeline has three stages:

1. **PreparePageRankData** – Constructs the user-interaction graph and sets initial PageRank scores based on UserMass.
2. **WeightedPageRank** – Iteratively updates PageRank scores until convergence. Supports both weighted (interaction-strength-based) and unweighted variants.
3. **ExtractTweepcred** – Converts the final PageRank values to a Tweepcred score (0–100) using a log-linear formula.

The **conversion formula** (from [`Reputation.scala`](src/scala/com/twitter/graph/batch/job/tweepcred/Reputation.scala)):
```
tweepcred = round(130 + 5.21 * ln(pagerank))
clamped to [0, 100]
```

This means that the tweepcred score is a logarithmic transformation of PageRank. Doubling your effective PageRank increases your tweepcred by about `5.21 * ln(2) ≈ 3.6` points.

### 8.2 UserMass: Initial PageRank Seed

The `UserMass` class ([`UserMass.scala`](src/scala/com/twitter/graph/batch/job/tweepcred/UserMass.scala)) computes the initial mass (seed weight) for each user in the PageRank computation. The formula is:

1. **Base score:**
   - `score = 0.05` (base from `deviceWeightAdditive * 0.1`)
   - `+ 0.5` if the account has a valid messaging device registered
2. **Age normalization:** `score *= min(1.0, log(1 + accountAgeDays / 15))`
   - An account that is 30+ days old receives the full age multiplier (1.0)
   - Very new accounts (< 30 days) have their score reduced logarithmically
3. **Restriction penalty:** If account is restricted, `score *= 0.1` (90% reduction)
4. **Suspended / deactivated accounts:** `mass = 0` (excluded from the graph entirely)
5. **Legacy verified accounts:** `mass = 100` (maximum seed weight)

### 8.3 Post-Calculation Follower Ratio Adjustment

After the PageRank converges, the final score is adjusted for accounts with a disproportionately high following-to-follower ratio (i.e., accounts that follow many people but are followed by few). The penalty in [`Reputation.scala`](src/scala/com/twitter/graph/batch/job/tweepcred/Reputation.scala):

- Applies when `numFollowings > 2500` (absolute threshold)
- If `(1 + numFollowings) / (1 + numFollowers) > 0.6`:
  - `divFactor = exp(3.0 * (ratio - 0.6) * ln(ln(numFollowings)))`
  - `adjustedMass = mass / min(divFactor, 50)` — can reduce mass by up to 50×

**Practical implication:** Following thousands of accounts while only being followed by a few hundred causes a severe reputation penalty. Organic follower growth—where people follow you because they find your content valuable—produces a much healthier following-to-follower ratio and preserves your Tweepcred score.

There is a similar but more aggressive penalty in UserMass itself (during the seed mass calculation):

- Applies when `numFollowings > 500` AND `(1 + numFollowings) / (1 + numFollowers) > 0.6`
- `adjustedMass = mass / exp(5.0 * (ratio - 0.6))`

---

## 9. Network Graphs: Real Graph and SimClusters

### 9.1 Real Graph

Described in [`src/scala/com/twitter/interaction_graph/README.md`](src/scala/com/twitter/interaction_graph).

Real Graph is a machine learning system (gradient boosted tree classifier, trained in BigQuery ML) that predicts the probability that User A will interact with User B. It is used to weight the follow graph when traversing candidate sources.

**Features used to train the model include:**

- `num_days` – Number of days over which the interaction pair has been active
- `num_tweets` – Number of tweets from User B that User A has seen
- `num_follows` – Direct follow relationship
- `num_favorites` – Number of likes from A on B's posts
- Retweet counts, reply counts, click counts
- Profile view history
- Address book matches (if user opted in to share contacts)
- Decayed rollup sums of all the above (more recent interactions weighted higher)

**Two model variants:**

- **`prod`** – Predicts probability of any interaction the next day
- **`prod_explicit`** – Predicts probability of explicit interactions (likes, retweets, replies) the next day

The Real Graph score is used throughout the pipeline to weight how strongly candidates sourced from a user's follow graph are promoted. A viewer with a high Real Graph score toward an author is much more likely to see that author's content.

**How this affects you:** Any authentic interaction between you and another user—likes, replies, retweets—raises the Real Graph score between you in both directions. This increases the probability that your content will appear in their For You feed and vice versa. Consistent engagement between two accounts, over time, is what drives this score up.

### 9.2 Graph Feature Service (GFS)

Described in [`graph-feature-service/README.md`](graph-feature-service/README.md).

GFS serves graph features for directed pairs of users. Example features it can answer:

- "How many of User A's followings have liked User C's posts?"
- "How many of User A's followings follow User C?"
- "How similar is User C to the accounts that User A has liked?"

These features are used in ranking to measure how well a candidate author fits within a viewer's social neighborhood.

### 9.3 SimClusters: Community-Based Embeddings

Described in [`src/scala/com/twitter/simclusters_v2/README.md`](src/scala/com/twitter/simclusters_v2/README.md). Published in KDD'2020 Applied Data Science Track.

SimClusters is a general-purpose representation layer that maps users and content into overlapping interest communities. The algorithm works as follows:

**Step 1: Follow Graph as a Bipartite Graph**

The follow graph is represented as a bipartite graph: Producers (followed accounts) on one side, Consumers (followers) on the other.

**Step 2: Community Detection ("Known For")**

- Producer-producer similarity is computed as cosine similarity between their follower sets.
- A producer-producer similarity graph is built and edges below a noise threshold are removed.
- Metropolis-Hastings sampling-based community detection is run to find K communities.
- **In production:** K ≈ 145,000 communities, covering the top 20 million producers.
- Each producer is assigned to at most one "Known For" community in the maximal-sparsity KnownFor matrix.

**Step 3: Consumer Embeddings ("InterestedIn")**

Each consumer's InterestedIn embedding is computed by multiplying the follow graph matrix by the KnownFor matrix. This gives a sparse vector representing which communities the consumer has interest in, based on who they follow. These embeddings capture **long-term interests**.

**Step 4: Producer Embeddings**

Producer embeddings capture a richer representation than KnownFor alone—a producer can be known across multiple communities. Producer embeddings are used for producer-based tweet recommendations (e.g., when you follow a new account, similar accounts' tweets are recommended).

**Step 5: Tweet Embeddings (Real-Time)**

Tweet embeddings are particularly important for content distribution:

- When a tweet is posted, its embedding starts as an **empty vector**.
- Every time a user likes the tweet, the liker's InterestedIn vector is **added to the tweet's embedding vector**.
- This is processed in real time via a Heron streaming job (Storm).
- The more a tweet is liked by users with strong community embeddings, the more it aligns with those communities.
- Tweet embeddings are also persisted to Manhattan for longer-term storage.

**What this means for you:** Your tweet's SimClusters embedding is shaped by who likes it. If influential members of a specific interest community (e.g., tech, sports, politics) like your post in the first few hours, your tweet's embedding will align strongly with that community and be recommended broadly to other members of it. Early likes from relevant community members are disproportionately valuable.

**Step 6: Topic Embeddings**

Topic embeddings are computed from how users who follow a topic interact with annotated tweets. The Semantic Core Entity Embeddings map topics to community clusters—when your post is annotated with a topic (e.g., "NBA," "Golden State Warriors") by the Topic Social Proof Service (TSPS), it inherits that topic's community embedding.

---

## 10. Out-of-Network Discovery: UTEG, UUG, and FRS

Out-of-network content represents a major opportunity for creators to reach new audiences. Here's how each pathway works in detail.

### 10.1 UTEG Collaborative Filtering (24–48 Hour Window)

UTEG ([`src/scala/com/twitter/recos/user_tweet_entity_graph/README.md`](src/scala/com/twitter/recos/user_tweet_entity_graph/README.md)) works by:

1. Taking the viewer's weighted follow graph (a list of followed user IDs with Real Graph weights).
2. Performing a traversal over the user-tweet engagement graph to find tweets that users in the follow graph have recently liked.
3. Aggregating across multiple hops: the more highly-weighted users in the follow graph who liked a given tweet, the higher that tweet's candidate score.
4. Returning the top-weighted tweets as out-of-network candidates.

This means: for your post to reach a new user via UTEG, at least one user in that new user's follow graph must have liked your post within the past 24–48 hours.

### 10.2 UTG Multi-Hop Traversal (24–48 Hour Window)

User Tweet Graph (UTG) performs 1-hop, 2-hop, and 3+hop traversals over a bidirectional user-tweet engagement graph. This enables it to find tweets that are 2 or 3 engagement steps away from a seed user or tweet—providing wider but more indirect reach.

### 10.3 FRS FutureGraph Tweets

FRS surfaces tweets from accounts you might follow in the future, based on who your existing follows have recently followed (UUG) and a variety of other signals. The ranking score is:

```text
score = w1 * p(follow | recommendation) + w2 * p(positive engagement | follow)
```

FRS adds **social proof annotations** (e.g., "Followed by [accounts you follow]") to these recommendations, which further increase click-through rates.

### 10.4 Representation Scorer (Embedding Similarity)

Described in [`representation-scorer/README.md`](representation-scorer/README.md).

The Representation Scorer (RSX) computes pairwise similarity scores between user and content embeddings (SimClusters, TwHIN, and others). These scores are used as ML features at both the candidate retrieval and ranking stages. A high embedding similarity between your post and a viewer's interest profile is a strong positive signal for surfacing your post to that viewer.

---

## 11. Engagement Quality Filters (Ratio-Based)

The algorithm uses engagement ratios to detect content that may be generating superficial or controversy-driven engagement. These filters are applied primarily to out-of-network content.

### 11.1 Reply-to-Like Ratio Filter

Implemented in [`pushservice/src/main/scala/com/twitter/frigate/pushservice/predicate/TweetEngagementRatioPredicate.scala`](pushservice/src/main/scala/com/twitter/frigate/pushservice/predicate/TweetEngagementRatioPredicate.scala):

```scala
val ratio = replyCount / likeCount.max(1)
```

For out-of-network tweet candidates, if:
- `replyCount > TweetReplytoLikeRatioReplyCountThreshold` (must have enough replies to be evaluated)
- `TweetReplytoLikeRatioThresholdLowerBound < ratio < TweetReplytoLikeRatioThresholdUpperBound`

...the tweet is **filtered out** and not sent as a notification.

**What this means:** A tweet with many replies but few likes suggests controversy or a debate rather than genuine appreciation. The ratio filter is applied in a band—not just a simple "too many replies" rule—so extremely viral posts with many both likes and replies are not penalized. The concern is specifically posts where the reply count is disproportionately higher than the like count.

### 11.2 Quote-Tweet-to-NtabClick Ratio Filter

Also in [`TweetEngagementRatioPredicate.scala`](pushservice/src/main/scala/com/twitter/frigate/pushservice/predicate/TweetEngagementRatioPredicate.scala):

```scala
lazy val quoteRate = if (ntabClickCount > 0) quoteCount / ntabClickCount else 1.0
```

For out-of-network candidates:
- Requires `ntabClickCount >= 1000` (sufficient volume to evaluate)
- If `quoteRate >= QTtoNtabClickRatioThreshold`: tweet is filtered

**What this means:** A very high ratio of quote tweets to notification opens suggests a post is being widely mocked or ratio'd rather than genuinely appreciated. Posts that spread primarily through controversial quote tweets are filtered from the notifications candidate pool.

---

## 12. Language Matching for Out-of-Network Content

Implemented in [`pushservice/src/main/scala/com/twitter/frigate/pushservice/predicate/TweetLanguagePredicate.scala`](pushservice/src/main/scala/com/twitter/frigate/pushservice/predicate/TweetLanguagePredicate.scala).

For out-of-network tweet candidates, the system checks whether the tweet's language matches the viewer's language profile. The viewer's language profile is built from five sources:

1. **`user.language.user.preferred_contents`** – Explicitly preferred content languages (set by the user in settings)
2. **`user.language.user.engagements`** – Languages of content the user has engaged with (weighted by engagement frequency, threshold-filtered)
3. **`user.language.user.following_accounts`** – Languages used by accounts the user follows
4. **`user.language.user.produced_tweets`** – Languages the user has tweeted in themselves
5. **`user.language.user.recent_devices`** – Device language settings

A tweet must match at least one language in this combined set to pass the filter.

**Exception for media content:** The `skipLanguageFilterForMediaTweets` flag allows image and video tweets to bypass the language filter. This is why viral video content can spread across language barriers more easily than text posts.

**Practical implication:** If you post in a language that the viewer does not consume content in, engage with, follow, or produce, your out-of-network content will be filtered out before it can appear in their recommendations—regardless of how engaging it is.

---

## 13. Feedback Fatigue: How Negative Feedback Suppresses Future Posts

Implemented in [`home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/filter/FeedbackFatigueFilter.scala`](home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/filter/FeedbackFatigueFilter.scala).

The Feedback Fatigue filter tracks instances where a user has clicked "See Fewer" (the `SeeFewer` feedback type) on content. It then suppresses future content from associated accounts for a configurable duration (`FeedbackFatigueFilteringDurationParam`).

The filter tracks four types of feedback engagement:

1. **Tweet feedback** – User clicked "See Fewer" directly on a tweet from Author X → future tweets by Author X are filtered from that user's timeline
2. **Like feedback** – User clicked "See Fewer" on content liked by Account L → future content liked by Account L is filtered (unless it has other likers who haven't been filtered)
3. **Follow feedback** – User clicked "See Fewer" on content from someone newly followed by Account F → future content from that pathway is suppressed
4. **Retweet feedback** – User clicked "See Fewer" on content retweeted by Account R → future retweets by Account R are filtered

**Practical implication:** "See Fewer" clicks have a persistent, multi-pathway suppression effect that is much stronger than a single "not interested" might suggest. If your posts consistently cause "See Fewer" clicks, your reach will be progressively reduced for those users until the feedback duration expires.

---

## 14. Notifications: A Separate Recommendation Pipeline

Described in [`pushservice/README.md`](pushservice/README.md).

The Notifications recommendation system (`pushservice`) runs a completely separate 4-stage pipeline from the Home timeline. It involves:

1. **Candidate generation** – Multiple candidate sources queried using target user signals.
2. **Candidate hydration** – Batch calls to downstream services to fill in features.
3. **Light ranking** – A lightweight MLP model pre-selects high-relevance candidates (described in [`pushservice/src/main/python/models/light_ranking/README.md`](pushservice/src/main/python/models/light_ranking/README.md)).
4. **Heavy ranking** – The ClemNet multi-task model predicts `p(notification open)` and `p(engagement after open)`.
5. **Quality control / heavy filtering** – Post-ranking checks including health scores, engagement ratio filters, and language matching.

The final candidate is delivered as a **push notification** and/or an **in-app notification** (Ntab).

### Notification SimClusters Indexes

Two specialized SimClusters ANN indexes power notification candidate generation:

- **PushOpenBased index** – Built from user push notification open history. Finds tweets that users with similar notification-open behavior have engaged with.
- **VideoViewBased index** – Built from video view history. Powers video content recommendations.

**Practical implication for creators:** Getting your post included in notifications to new users requires passing all four stages of the notification pipeline independently—the light ranker, heavy ranker, and all quality filters. Notification engagement (notification opens, Ntab clicks) feeds back into the SimClusters ANN indexes, giving you a positive flywheel effect if your content consistently drives notification opens.

---

## 15. Diversity and Content Balance Filters

After ranking, several filters ensure your feed doesn't become repetitive:

### 15.1 Author Diversity

The pipeline limits how many posts from a single author can appear in a single feed refresh. This protects the feed from being dominated by any one account, even highly-ranked ones.

**Implication for you:** Even if your posts all score very highly for a given viewer, only a limited number will appear per refresh. Consistent posting over time gives more opportunities to be included in successive refreshes rather than saturating a single one.

### 15.2 Content Balance (In-Network vs. Out-of-Network)

The `ForYouScoredTweetsMixerPipelineConfig` balances in-network and out-of-network content. The search index (Earlybird) is described as providing approximately **50% of For You Timeline content**, with the remainder coming from out-of-network sources.

### 15.3 TwHIN and Category Diversity Rescoring

The Home Mixer params include `TwhinDiversityRescoringParam` and `CategoryDiversityRescoringParam`, which apply diversity rescoring based on TwHIN embeddings and content categories respectively. This means the system actively prevents too many similar posts (by embedding similarity or topic) from appearing in a single feed refresh.

### 15.4 Previously Seen Content Removal

Candidates that a viewer has already seen are deduplicated and removed from the pool before final ranking.

---

## 16. Visibility Filtering: What Can Get Your Content Suppressed

Described in [`visibilitylib/README.md`](visibilitylib/README.md).

The Visibility Filtering library is a rule engine that applies per-content, per-viewer filtering. It supports several action types:

- **Hard filtering (`Drop`)** – Content is completely removed from the viewer's response.
- **Soft filtering (`Labels` / `Interstitials`)** – Content is shown with a warning label or click-through interstitial.
- **Ranking clues (downranking)** – Content is retained but its ranking score is reduced.

Filtering is applied based on:

- **Safety labels** on content (tweets, users, media, spaces, DMs)
- **Viewer's settings** (NSFW content enabled/disabled, sensitive media settings)
- **Relationships** between viewer and content (block, mute, report)
- **Legal compliance** requirements in specific regions

Key safety label types that affect content distribution:

- Posts by users with high `pAbuse` or `pNSFW` scores receive safety labels that trigger interstitials or hard drops.
- Accounts flagged as NSFW author (`isNsfwAuthor`) have their content filtered in non-explicit contexts.
- Reported content accumulates `pReportedTweetScore` signals which can lead to SafetyLabel application.

---

## 17. Recommendations for Creators

The following recommendations are derived directly from the algorithm's mechanics described above.

### 17.1 Optimize for Early Engagement Velocity

The Earlybird Feature Update Service pushes like/retweet/reply counts into the ranking index in near-real-time. UTEG and UTG maintain a 24–48 hour sliding window. SimClusters tweet embeddings are built incrementally from likes. All of these converge on the same conclusion: **the first hours after posting are the most critical**.

- Post at times when your most engaged followers are active.
- Engage with your audience immediately after posting (reply to comments, like replies) to keep the conversation active.
- Consider using threads to link related posts—a user who engages with one post in a thread has features tied to the full conversation.

### 17.2 Build a Healthy Follower-to-Following Ratio

The Tweepcred system imposes severe penalties on accounts where following dramatically outnumbers followers:

- The UserMass penalty kicks in at **500 followings** if the following/follower ratio exceeds **0.6**.
- The post-PageRank penalty kicks in at **2,500 followings** and can reduce your score by **up to 50×**.

Aggressively following accounts to gain follows-back will directly harm your Tweepcred and reduce the baseline ranking score of all your future posts.

### 17.3 Keep Your Account Age Healthy

The UserMass age normalization function `min(1.0, log(1 + accountAgeDays / 15))` means:
- A 15-day-old account has roughly 69% of the age multiplier of a mature account.
- A 30-day-old account effectively reaches the maximum age multiplier.
- Brand new accounts start with very low seed mass in the PageRank graph, which means their Tweepcred starts low even with good engagement.

### 17.4 Post Original Top-Level Content (Not Retweets)

The light ranker explicitly tracks `isRetweet` as a static feature. Retweets are scored differently from original posts. Original top-level posts:
- Build author credibility (Tweepcred is based on interaction graph, not retweet behavior)
- Generate stronger SimClusters tweet embedding signals when liked
- Do not inherit the safety or quality scores of the original tweet author

### 17.5 Use Video Content Strategically

Videos have a dedicated candidate pathway (User Video Graph), a dedicated SimClusters ANN index (VideoViewBased), and their watch time is directly used as a **label** for the light ranker. Additionally, media tweets can bypass the language filter for out-of-network recommendations.

- Videos should be long enough to generate meaningful watch-time signals.
- Vertical aspect ratio (`isVerticalAspectRatio`) is tracked—portrait/vertical video is optimized for mobile consumption.
- High-resolution media (`isHighMediaResolution`) is also tracked as a feature.

### 17.6 Write Content That Gets Clicked Through (Not Just Liked)

Tweet clicks (viewing the full detail page) are used as **Labels** for the light ranker—meaning the model directly optimizes to predict click-through. A post that many people click through to read in full gets a higher light ranker score. Write posts with:
- Compelling hooks that create curiosity or inform quickly
- First sentences that make the reader want to see the rest

### 17.7 Understand the Reply-to-Like Ratio Filter

If your post is polarizing and generates many replies with few likes (ratio'd), it may be filtered from notifications and out-of-network recommendations. This isn't about avoiding all controversy—it's about ensuring your posts generate genuine appreciation (likes) alongside the discussion (replies).

### 17.8 Post in Your Audience's Language

Out-of-network distribution relies on language matching across five user language profile dimensions. Text-only posts in a minority language will fail to reach most out-of-network viewers whose language profile doesn't include that language. Media posts (images, video) can skip this filter.

### 17.9 Become a Core Member of a SimClusters Community

Your post's SimClusters embedding is shaped by who likes it. If you want your content to spread within a specific interest community:
- Engage consistently with the top producers in that community (they have well-established KnownFor assignments)
- Get your early likes from active members of that community
- Post content that is topically aligned with the community (so that when community members see and like it, the embedding signals are coherent)

### 17.10 Avoid the Feedback Fatigue Trap

"See Fewer" clicks generate persistent, multi-pathway suppression. If users repeatedly see your content via retweeters, likers, or follows-of-follows and click "See Fewer" on it, those pathways are blocked for a duration. Focus on content that your audience genuinely wants to see, not content designed to maximize impressions at the cost of satisfaction.

### 17.11 Maintain a Clean Health Score Profile

The five health scores (toxicityScore, pBlockScore, pReportedTweetScore, pSpammyTweetScore, spammyTweetContentScore) are evaluated against out-of-network content in particular. A pattern of reports, blocks, or toxic language classifications will cause your content to fail the `isPassTweetHealthFilterStrict` check, removing you from out-of-network candidate pools entirely—regardless of your engagement metrics.

### 17.12 Use Topic Annotations to Your Advantage

The Topic Social Proof Service (TSPS) annotates tweets with semantic topic labels (e.g., "NBA," "Machine Learning," "Climate") based on the tweet's text and SimClusters embedding similarity. Posts that earn a topic annotation are surfaced to users who follow that topic, even if they don't follow you. Write posts that are clearly topically focused so they earn accurate topic annotations.

---

## 18. Further Reading

Internal repository references:

- [Candidate Sourcing Signals](RETREIVAL_SIGNALS.md) – Full signal usage matrix across all candidate sources
- [For You Timeline Architecture](README.md) – Top-level system overview
- [SimClusters: Community Detection and Embeddings](src/scala/com/twitter/simclusters_v2/README.md) – Full algorithm description (KDD'2020 paper)
- [Earlybird Light Ranker](src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/README.md) – Feature pipeline and model details
- [Tweepcred: User Reputation](src/scala/com/twitter/graph/batch/job/tweepcred/README) – PageRank-based reputation system
- [CR-Mixer: Candidate Retrieval Mixer](cr-mixer/README.md) – Out-of-network candidate coordination
- [Follow Recommendations Service](follow-recommendations-service/README.md) – FutureGraph tweet recommendations
- [Home Mixer](home-mixer/README.md) – Main For You Timeline pipeline
- [Pushservice (Notifications)](pushservice/README.md) – Notification recommendation pipeline
- [User Signal Service](user-signal-service/README.md) – Centralized user signal platform
- [Unified User Actions](unified_user_actions/README.md) – Real-time action event stream
- [Trust and Safety Models](trust_and_safety_models/README.md) – pNSFW, pToxicity, pAbuse models
- [Topic Social Proof Service](topic-social-proof/README.md) – Topic annotation system
- [Visibility Filtering](visibilitylib/README.md) – Safety and compliance filtering
- [Real Graph](src/scala/com/twitter/interaction_graph/README.md) – User-user interaction prediction
- [Graph Feature Service](graph-feature-service/README.md) – Directed user pair graph features
- [Representation Scorer](representation-scorer/README.md) – Embedding-based scoring features

External references:

- [X Recommendation Algorithm – Engineering Blog](https://blog.x.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm)
- [SimClusters KDD'2020 Paper](https://www.kdd.org/kdd2020/accepted-papers/view/simclusters-community-based-representations-for-heterogeneous-recommendatio)
- [GraphJet: High-Performance In-Memory Storage](https://github.com/twitter/GraphJet)
