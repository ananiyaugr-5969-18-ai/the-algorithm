# SKILL: Creating Highly Engaging Content on X

This document provides guidance on creating highly engaging content on X, derived from insights in the open-source recommendation algorithm. Understanding how X's algorithm evaluates and ranks content can help creators optimize their posts for maximum reach and engagement.

## How the Algorithm Works

X's recommendation algorithm selects content through a multi-stage pipeline:

1. **Candidate Generation** – Billions of posts are narrowed to thousands of candidates using signals from your network and beyond.
2. **Feature Hydration** – ~6,000 features are computed for each candidate post.
3. **Ranking** – A light ranker pre-filters candidates; a heavy neural network ranker produces final scores.
4. **Filtering & Mixing** – Diversity, safety, and relevance filters are applied before content is surfaced.

## Engagement Signals That Drive Reach

The algorithm uses the following user interaction signals as training labels and ranking features. Maximizing positive signals—and minimizing negative ones—directly improves how widely your content is distributed.

### Positive Signals (increase reach)
| Signal | Description |
|---|---|
| **Likes** | Users clicking the like button on your post |
| **Retweets** | Users reposting your content to their followers |
| **Quote Tweets** | Users reposting your content with added commentary |
| **Replies** | Users responding to your post (conversation depth matters) |
| **Bookmarks** | Users saving your post for later |
| **Post Clicks** | Users clicking through to read your post detail page |
| **Video Watch Time** | Users watching your video content (both duration and percentage matter) |
| **Profile Visits** | Users visiting your profile after seeing your post |
| **Follows** | Users following you after discovering your content |
| **Notification Opens** | Users engaging with push notifications about your content |

### Negative Signals (reduce reach)
| Signal | Description |
|---|---|
| **"Not interested"** | Users marking your post as not relevant |
| **Reports** | Users reporting your post |
| **Unfollows** | Users unfollowing you after seeing your post |
| **Mutes / Blocks** | Users muting or blocking your account |

## Content Quality Factors

The algorithm evaluates static content quality at index time. The following factors influence your content's quality score:

- **Readability** – Clear, well-structured writing scores higher. Avoid overly complex or convoluted phrasing.
- **No excessive capitalization ("shout" score)** – WRITING IN ALL CAPS is penalized. Use standard capitalization.
- **Content entropy** – Posts that contain coherent, meaningful information rank better than low-information or spammy text.
- **Absence of offensive content** – Toxic or offensive language is penalized by health models (`pToxicity` score).
- **Appropriate length** – Posts that are too short (low information) or excessively long can score lower. Aim for clarity and substance.

## Rich Content Features

Certain content types receive feature boosts because they drive higher engagement:

- **Images and videos** – Multimedia content has dedicated feature extraction and tends to drive higher click and watch-time signals.
- **Links and cards** – Posts with URLs and card previews (rich link previews) are treated as a distinct content type with dedicated features.
- **Trend words** – Posts that contain words aligning with current trends may receive a relevance boost during indexing.
- **Quotes** – Using the quote tweet format to add commentary to existing content is tracked as an engagement type.

## Author Credibility Signals

Your account's reputation directly affects how your content is distributed:

- **Tweepcred (Author Reputation)** – X uses a PageRank-style algorithm ([tweepcred](src/scala/com/twitter/graph/batch/job/tweepcred/README)) to compute author reputation based on your follower graph. A higher reputation score improves your content's baseline ranking.
- **Follower graph quality** – It is not just the number of followers that matters but how engaged and reputable those followers are.
- **Account age and consistency** – Long-standing accounts with consistent engagement history are treated more favorably.
- **Engagement rate** – The ratio of engagements to impressions on your past content is used as a signal. Consistently high engagement rates signal that your audience finds your content valuable.

## Network and Relevance

- **In-network content** (~50% of the For You Timeline) comes from accounts you follow. Build a relevant follower base by posting consistently within a clear topic area.
- **Out-of-network content** is surfaced when your posts are engaged with by users who share interests with the target viewer. Content that resonates across communities amplifies out-of-network reach.
- **SimClusters embeddings** – X maps users and content into interest communities. Posts that align with well-defined interest clusters surface more broadly to users in those clusters.
- **Real Graph score** – A higher likelihood of interaction between two users (based on past engagement history) increases the chance of your content being shown to users in your network.

## Recommendations for Creators

Based on the algorithm's design, the following practices are most likely to increase content reach:

1. **Post original, high-quality content** – Original posts outperform retweets for building author reputation and engagement rate.
2. **Encourage meaningful interaction** – Replies and quote tweets are strong ranking signals. Ask questions, share opinions, and invite discussion.
3. **Use media thoughtfully** – Images and especially videos generate additional engagement signals (watch time, clicks) that text-only posts cannot.
4. **Avoid engagement bait** – Algorithmically induced engagements from low-quality interactions can hurt your engagement rate ratio and may trigger spam signals.
5. **Be consistent with a topic** – Clustering your posts around clear topics improves your SimClusters alignment, making your content easier to surface to interested audiences.
6. **Post when your audience is active** – Realtime engagement signals (likes, retweets, replies in the first minutes/hours) heavily influence ranking. Early engagement velocity matters.
7. **Avoid behaviors that generate negative signals** – Content that drives "Not interested" clicks, reports, or unfollows actively reduces the reach of future posts.
8. **Engage with your community** – Replying to others and building mutual engagement relationships improves the Real Graph score between you and other users, increasing how often your content appears in their feeds.
9. **Keep your account in good standing** – Posts by accounts flagged with high `pToxicity` or `pBlock` scores are downranked. Maintaining respectful, constructive communication keeps your health scores low.
10. **Use links and cards** – Adding a URL with a rich card preview makes your post more visually prominent and drives click-through signals.

## Further Reading

- [X Recommendation Algorithm – Engineering Blog](https://blog.x.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm)
- [Candidate Sourcing Signals](RETREIVAL_SIGNALS.md)
- [For You Timeline Architecture](README.md)
- [SimClusters: Community Detection and Embeddings](src/scala/com/twitter/simclusters_v2/README.md)
- [Earlybird Light Ranker](src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/README.md)
- [Tweepcred: User Reputation](src/scala/com/twitter/graph/batch/job/tweepcred/README)
