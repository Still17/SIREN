# Simple App - System Overview

## What the App Does

A cryptocurrency sentiment analysis system that:
1. Collects tweets about specific cryptocurrencies
2. Analyzes tweet sentiment and quality using AI
3. Calculates weighted sentiment signals for each coin
4. Tracks sentiment trends over multiple time windows

**Goal**: Identify trending coins based on social media sentiment to inform trading decisions.

---

## Collection Frequency

**Every 10 minutes** (configurable via `WORKER_INTERVAL_MINUTES`)

Each round:
- Fetches ~10 pages per coin-specific query
- Processes 200-700 tweets per round
- Updates existing tweets' engagement metrics
- Calculates fresh NWSF metrics for all coins

---

## Funnel Stages & Filtering

### Stage 1: Tweet Fetching
**Source**: Twitter via ScrapeStorm API

**Search Strategy**: Coin-specific queries only
- Generates queries for 50 tracked coins (e.g., `($BTC OR $ETH OR $SOL) AND (coin OR token OR crypto) lang:en min_faves:3`)
- Batches coins into groups of 3 per query
- Fetches 10 pages per query with 0.5s delay between requests

**Initial Filters**:
- Language: English only (`lang:en`)
- Minimum engagement: 3+ likes (`min_faves:3`)
- Excludes quotes and retweets (`-filter:quote -filter:retweets`)
- Spam detection: Filters out tweets with spam indicators (excessive hashtags, promotional patterns)

### Stage 2: Deduplication
- Checks database for existing tweet IDs
- **New tweets**: Sent through full LLM classification pipeline
- **Existing tweets**: Only engagement metrics updated (likes, retweets, views, etc.)

### Stage 3: Author Quality Scoring
**Only for new tweets**

Calculates 0-100 score based on:
- **Views Sanity** (0-30): Views-to-followers ratio (tier-adjusted)
- **Content Originality** (0-10): Original tweets vs retweets ratio
- **Account Age** (0-20): >2 years = 20, >1 year = 15, >6 months = 10
- **Engagement Sanity** (0-30): Engagement-to-followers ratio (detects fake followers)
- **Follower Ratio** (0-5): Followers/following ratio
- **Verification** (0-5): Legacy verified = 5, Twitter Blue = 0

Cached for 7 days per author.

### Stage 4: LLM Classification
**Model**: GPT-4o-mini (temperature=0)

**Extracts**:
- `coin_name`: Ticker symbol (e.g., "BTC", "ETH")
- `sentiment_score`: -10 (extreme bearish) to +10 (extreme bullish)
- `depth_score`: 0 (noise) to 10 (deep analysis)
- `keywords`: 2-3 topic keywords

**Coin Detection Priority**:
1. Explicit tickers ($BTC, #SOL, BTC/USDT)
2. Clear coin names (Bitcoin, Ethereum)
3. Ignores generic terms unless specific coin mentioned
4. Prefers main topic coin over incidental mentions

**Filtered Out**: Tweets with no coin or empty coin_name

### Stage 5: Penalty Calculation
Reduces signal quality for spam indicators:
- **Hashtags**: 0-0.15 (>3 hashtags penalized)
- **Mentions**: 0-0.10 (>2 mentions penalized)
- **URLs**: 0.05 (any URL)
- **Spam Keywords**: 0.10 (giveaway, signal, pump, etc.)
- **Total**: Capped at 0.3

### Stage 6: WSS Calculation
**WSS (Weighted Sentiment Signal)** = Primary tweet-level metric

**Formula**:
```
WSS = sentiment_score × (depth_score/10) × (author_quality/100)
```

**Range**: -10 to +10 (in practice, typically -2 to +2)

**Example**:
- Sentiment: +8 (strong bullish)
- Depth: 6 (multi-point analysis)
- Author Quality: 75
- WSS = 8 × 0.6 × 0.75 = **3.6**

**Filtered Out**: Tweets with null WSS (missing any component)

### Stage 7: Database Storage
**Saved**:
- New tweets with all metrics
- New authors with quality scores
- Updated engagement metrics for existing tweets

---

## Metrics

### WSS (Weighted Sentiment Signal)
**Per-tweet metric** combining sentiment, analysis depth, and author credibility.

**Components**:
- **Sentiment** (-10 to +10): Emotional tone
  - +9/+10: Extreme conviction ("never seeing these prices again")
  - +5/+8: Strong hype/FOMO ("buy now or regret forever")
  - +1/+4: Mild bullish ("accumulating, looks undervalued")
  - 0: Neutral/informational
  - -1/-4: Skeptical/cautious
  - -5/-8: Strong bearish/shorting
  - -9/-10: Panic/scam warnings

- **Depth** (0-10): Analytical quality
  - 0: Pure noise ("gm", emojis)
  - 1-2: Hype with zero reasoning
  - 3-4: Simple TA/FA (single indicator)
  - 5-7: Multi-point analysis (multiple indicators, on-chain data)
  - 8-10: Deep research (original discovery, mathematical proof)

- **Author Quality** (0-100): Credibility score (see Stage 3)

### NWSF (Net Weighted Sentiment Flow)
**Per-coin metric** aggregating sentiment over time windows.

**Formula**:
```
NWSF = SUM(WSS) for all tweets about the coin in time window
```

**Time Windows**:
- `nwsf_15min`: Last 15 minutes
- `nwsf_30min`: Last 30 minutes
- `nwsf_1h`: Last 1 hour
- `nwsf_3h`: Last 3 hours
- `nwsf_6h`: Last 6 hours
- `nwsf_12h`: Last 12 hours
- `nwsf_24h`: Last 24 hours

**Interpretation**:
- **Positive NWSF**: Net bullish sentiment
- **Negative NWSF**: Net bearish sentiment
- **Magnitude**: Strength of sentiment (higher absolute value = stronger signal)
- **Trend**: Compare shorter vs longer windows to detect momentum shifts

**Example**:
```
BTC:
  nwsf_1h: +12.5   (strong bullish in last hour)
  nwsf_24h: +45.2  (sustained bullish over day)
  
PEPE:
  nwsf_1h: -3.2    (recent bearish turn)
  nwsf_24h: +8.1   (was bullish, now reversing)
```

---

## AI Model & Prompt

### Model
- **Provider**: OpenAI
- **Model**: `gpt-4o-mini`
- **Temperature**: 0 (deterministic)
- **Output**: Structured JSON via Pydantic schema

### Prompt Structure
**System Role**: "You are a crypto-savvy analyst"

**Task**: Extract structured information from ONE crypto tweet

**Key Instructions**:
1. Detect specific coin (prefer explicit tickers, ignore generic terms)
2. Select most important coin if multiple mentioned (ignore SEO hashtags, portfolio lists)
3. Score sentiment -10 to +10 (emotional tone, not fundamentals)
4. Score depth 0-10 (analytical value, not sentiment)
5. Extract 2-3 topic keywords (no sentiment words, no emojis)
6. Return JSON only

**Prompt Length**: ~2,500 tokens (includes detailed examples for sentiment and depth scoring)

---

## Tracked Coins

**Total**: 50 coins (from `coins_list_top50.txt`)

**Categories**:
- **Top Market Cap** (9): BTC, ETH, SOL, BNB, XRP, ADA, DOGE, AVAX, DOT
- **Memecoins** (11): PEPE, SHIB, FLOKI, BONK, WIF, MEME, DEGEN, BRETT, MOG, WOJAK, POPCAT
- **DeFi** (6): UNI, AAVE, MKR, CRV, COMP, RUNE
- **Layer 1/2** (6): MATIC, ARB, OP, IMX, BLAST, BASE
- **AI/Data** (5): FET, AGIX, RNDR, GRT, AR
- **Gaming** (6): AXS, SAND, MANA, GALA, BEAM, PORTAL
- **New/Trending** (5): GRASS, JUP, WEN, MYRO, MEW
- **Ecosystem** (8): LINK, LDO, APT, SUI, SEI, INJ, TIA, PYTH

**Selection Criteria**:
- High liquidity
- Active trading volume
- No stablecoins
- Mix of established and trending coins

**Note**: System also captures coins mentioned by LLM that aren't in the list (allows discovery of new trending coins).

---

## Database Schema

### Table: `authors`
**Purpose**: Store Twitter users with quality metrics

**Key Fields**:
- `twitter_user_id` (unique): Twitter user ID
- `screen_name`: Twitter handle
- `followers_count`, `following_count`, `tweet_count`: Account metrics
- `quality_score`: 0-100 calculated score
- `is_verified`: Legacy verification status
- `last_fetched_at`: Last update timestamp

**Indexes**: `twitter_user_id`, `quality_score`, `updated_at`

### Table: `trending_tweets`
**Purpose**: Store analyzed tweets with all metrics

**Key Fields**:
- `twitter_id` (unique): Tweet ID
- `author_twitter_user_id` (FK): Links to authors table
- `text`: Tweet content
- `created_at_twitter`: Original tweet timestamp
- `coin_name`: Detected coin ticker
- `like_count`, `retweet_count`, `reply_count`, `view_count`: Engagement metrics
- `sentiment_score`: -10 to +10 from LLM
- `depth_score`: 0-10 from LLM
- `keywords`: Array of topic keywords
- `author_quality_score`: Denormalized from authors table
- `hashtag_penalty`, `mention_penalty`, `url_penalty`, `spam_penalty`: Individual penalties
- `total_penalty`: Sum of penalties (capped at 0.3)
- `wss_score`: Calculated WSS
- `fetched_at`: When tweet was collected
- `batch_id`: UUID for this collection round

**Indexes**: `twitter_id`, `coin_name`, `created_at_twitter`, `sentiment_score`, `wss_score`, `fetched_at`, `batch_id`

### Table: `coin_metrics`
**Purpose**: Store NWSF and engagement metrics per coin per round

**Key Fields**:
- `coin_name`: Coin ticker
- `circle_start_time`: Round timestamp
- `batch_id`: UUID for this round
- `nwsf_15min`, `nwsf_30min`, `nwsf_1h`, `nwsf_3h`, `nwsf_6h`, `nwsf_12h`, `nwsf_24h`: NWSF for each time window
- `total_likes`, `total_retweets`, `total_replies`, `total_views`: Aggregated engagement
- `tweet_count`: Total tweets for this coin in this round
- `new_tweet_count`: New tweets processed
- `updated_tweet_count`: Existing tweets updated
- `calculated_at`: Metric calculation timestamp

**Indexes**: `coin_name`, `circle_start_time`, `batch_id`
**Unique Constraint**: `(coin_name, batch_id)`

### Table: `coin_prices`
**Purpose**: Cache coin prices from CoinGecko

**Key Fields**:
- `coin_name`: Coin ticker
- `price_usd`: USD price
- `batch_id`: Round UUID
- `circle_start_time`: Round timestamp
- `fetched_at`: Price fetch timestamp
- `source`: Price source (default: 'coingecko')

**Indexes**: `coin_name`, `batch_id`, `fetched_at`
**Unique Constraint**: `(coin_name, batch_id)`

---

## Data Flow Summary

```
1. Every 10 minutes
   ↓
2. Fetch tweets (coin-specific queries, 5 pages each)
   ↓
3. Filter spam, deduplicate
   ↓
4. Split: Existing tweets → Update engagement only
         New tweets → Continue to step 5
   ↓
5. Calculate author quality (0-100)
   ↓
6. LLM classification (sentiment, depth, coin)
   ↓
7. Calculate penalties (hashtags, mentions, URLs, spam)
   ↓
8. Calculate WSS = sentiment × depth × author_quality
   ↓
9. Save to database (tweets, authors, metrics)
   ↓
10. Calculate NWSF for all coins (7 time windows)
    ↓
11. Save coin_metrics (NWSF + engagement)
```

---

## Key Design Decisions

### Why Coin-Specific Queries Only?
- Higher signal-to-noise ratio
- More relevant tweets about actual coins
- Reduces spam and generic crypto discussion

### Why Update Existing Tweets?
- Track engagement growth over time
- Viral tweets gain more weight in later rounds
- Enables engagement-based filtering/ranking

### Why 7 NWSF Time Windows?
- Detect momentum shifts (compare 15min vs 24h)
- Different trading strategies need different timeframes
- Identify sudden spikes vs sustained trends

### Why Cache Author Quality?
- Expensive to calculate (requires multiple tweets)
- Author quality changes slowly
- 7-day TTL balances freshness vs performance

### Why Separate WSS and NWSF?
- **WSS**: Individual tweet quality (for filtering, ranking)
- **NWSF**: Aggregate coin sentiment (for trading signals)
- Allows analysis at both micro (tweet) and macro (coin) levels
