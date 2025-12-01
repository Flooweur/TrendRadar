# TrendRadar Project Process Documentation

This document provides a comprehensive explanation of all processes implemented in the TrendRadar project, formatted for easy understanding by Large Language Models (LLMs).

## Table of Contents

1. [Project Overview](#project-overview)
2. [News Hotness Calculation](#news-hotness-calculation)
3. [News Grouping Process](#news-grouping-process)
4. [Keyword Matching System](#keyword-matching-system)
5. [Sorting and Ranking Algorithm](#sorting-and-ranking-algorithm)
6. [Data Flow and Processing Pipeline](#data-flow-and-processing-pipeline)
7. [Report Generation Modes](#report-generation-modes)
8. [Configuration System](#configuration-system)

---

## Project Overview

**TrendRadar** is a news aggregation and monitoring system that:
- Crawls trending news from multiple platforms (Weibo, Zhihu, Baidu, etc.)
- Filters news based on user-defined keywords
- Calculates hotness scores for news items
- Groups news by keyword matches
- Generates reports in multiple formats (HTML, text, notifications)

**Core Components:**
- `main.py`: Main processing logic and report generation
- `mcp_server/`: AI analysis server using Model Context Protocol
- `config/config.yaml`: System configuration
- `config/frequency_words.txt`: User-defined keyword groups

---

## News Hotness Calculation

### Function: `calculate_news_weight()`

**Location:** `main.py`, line 1043

**Purpose:** Calculates a weighted hotness score for each news item to determine its importance and ranking.

**Algorithm:**

The hotness score is calculated using three weighted components:

```
Total Weight = (Rank Weight × RANK_WEIGHT) + 
               (Frequency Weight × FREQUENCY_WEIGHT) + 
               (Hotness Weight × HOTNESS_WEIGHT)
```

**Default Weights (configurable in `config.yaml`):**
- `RANK_WEIGHT`: 0.6 (60%)
- `FREQUENCY_WEIGHT`: 0.3 (30%)
- `HOTNESS_WEIGHT`: 0.1 (10%)

### Component 1: Rank Weight

**Calculation:**
```python
rank_scores = []
for rank in ranks:
    score = 11 - min(rank, 10)  # Higher rank = higher score
    rank_scores.append(score)

rank_weight = sum(rank_scores) / len(ranks)
```

**Explanation:**
- Converts ranking positions (1-10) to scores (1-10)
- Rank 1 → Score 10, Rank 2 → Score 9, ..., Rank 10 → Score 1
- Ranks above 10 are capped at 10
- Takes the average of all rank scores for the news item

**Example:**
- News appears at ranks [1, 3, 5] across different crawls
- Scores: [10, 8, 6]
- Rank Weight = (10 + 8 + 6) / 3 = 8.0

### Component 2: Frequency Weight

**Calculation:**
```python
frequency_weight = min(count, 10) × 10
```

**Explanation:**
- `count` = number of times the news appeared across all crawls
- Capped at 10 appearances (max frequency_weight = 100)
- Rewards news that appears consistently

**Example:**
- News appears 5 times → Frequency Weight = 5 × 10 = 50
- News appears 15 times → Frequency Weight = 10 × 10 = 100 (capped)

### Component 3: Hotness Weight

**Calculation:**
```python
high_rank_count = sum(1 for rank in ranks if rank <= rank_threshold)
hotness_ratio = high_rank_count / len(ranks)
hotness_weight = hotness_ratio × 100
```

**Explanation:**
- `rank_threshold`: Default is 5 (configurable)
- Counts how many times the news appeared in top N positions
- Calculates the ratio of high-rank appearances
- Converts to a 0-100 score

**Example:**
- News appears 10 times with ranks: [1, 2, 3, 6, 7, 8, 1, 2, 4, 5]
- rank_threshold = 5
- high_rank_count = 7 (ranks ≤ 5: 1, 2, 3, 1, 2, 4, 5)
- hotness_ratio = 7 / 10 = 0.7
- Hotness Weight = 0.7 × 100 = 70

### Final Weight Calculation Example

Given:
- Ranks: [1, 2, 3, 6, 7]
- Count: 5
- rank_threshold: 5
- Weights: RANK=0.6, FREQUENCY=0.3, HOTNESS=0.1

**Step 1: Rank Weight**
- Scores: [10, 9, 8, 5, 5]
- Rank Weight = (10 + 9 + 8 + 5 + 5) / 5 = 7.4

**Step 2: Frequency Weight**
- Frequency Weight = min(5, 10) × 10 = 50

**Step 3: Hotness Weight**
- high_rank_count = 3 (ranks ≤ 5: 1, 2, 3)
- hotness_ratio = 3 / 5 = 0.6
- Hotness Weight = 0.6 × 100 = 60

**Step 4: Total Weight**
- Total = (7.4 × 0.6) + (50 × 0.3) + (60 × 0.1)
- Total = 4.44 + 15.0 + 6.0 = **25.44**

---

## News Grouping Process

### Function: `count_word_frequency()`

**Location:** `main.py`, line 1175

**Purpose:** Groups news items by matching keyword groups and calculates statistics for each group.

### Grouping Logic

**Step 1: Load Keyword Groups**

Keyword groups are loaded from `config/frequency_words.txt`:
- Groups are separated by empty lines
- Each group can contain:
  - Normal words: Basic keywords
  - Required words (prefix `+`): Must all be present
  - Filter words (prefix `!`): Exclude if present
  - Max count (prefix `@`): Limit number of news items per group

**Example Configuration:**
```
AI
ChatGPT
+技术

特斯拉
马斯克
@10

苹果
!水果
```

**Step 2: Match News to Groups**

For each news title, the system:
1. Checks if title contains any filter words → If yes, exclude
2. For each keyword group:
   - Checks if all required words are present
   - Checks if any normal words are present
   - If both conditions met, assign to that group

**Matching Function:** `matches_word_groups()` (line 1079)

**Step 3: Collect Statistics**

For each matched news item:
- Records which group it belongs to
- Tracks platform source
- Records ranks, URLs, timestamps
- Marks as "new" if first appearance

**Step 4: Sort Within Groups**

Within each group, news items are sorted by:
1. **Primary:** Negative hotness weight (descending)
2. **Secondary:** Minimum rank (ascending)
3. **Tertiary:** Negative count (descending)

**Code:**
```python
sorted_titles = sorted(
    all_titles,
    key=lambda x: (
        -calculate_news_weight(x, rank_threshold),
        min(x["ranks"]) if x["ranks"] else 999,
        -x["count"],
    ),
)
```

**Step 5: Apply Display Limits**

- If group has `@N` limit, only show top N items
- If global `MAX_NEWS_PER_KEYWORD` is set, apply as fallback
- Priority: Group-specific limit > Global limit > No limit

**Step 6: Sort Groups**

Groups are sorted based on `SORT_BY_POSITION_FIRST` configuration:

**Option A: Sort by Hotness First (default)**
```python
stats.sort(key=lambda x: (-x["count"], x["position"]))
```
- Groups with more news items appear first
- Secondary sort by configuration order

**Option B: Sort by Configuration Order First**
```python
stats.sort(key=lambda x: (x["position"], -x["count"]))
```
- Groups appear in the order defined in `frequency_words.txt`
- Secondary sort by number of news items

---

## Keyword Matching System

### Function: `matches_word_groups()`

**Location:** `main.py`, line 1079

**Purpose:** Determines if a news title matches any keyword group rules.

### Matching Rules

**Rule 1: Filter Words (Highest Priority)**
- If title contains ANY filter word (prefix `!`), immediately return `False`
- Filter words apply globally across all groups

**Rule 2: Required Words**
- If a group has required words (prefix `+`), ALL must be present in the title
- Case-insensitive matching

**Rule 3: Normal Words**
- If a group has normal words, AT LEAST ONE must be present in the title
- Case-insensitive matching

**Rule 4: Empty Configuration**
- If no keyword groups are configured, all news items match

### Matching Algorithm

```python
def matches_word_groups(title, word_groups, filter_words):
    # Step 1: Check filter words
    if any(filter_word in title.lower() for filter_word in filter_words):
        return False
    
    # Step 2: Check each group
    for group in word_groups:
        required_words = group["required"]
        normal_words = group["normal"]
        
        # Step 2a: Check required words
        if required_words:
            if not all(req_word in title.lower() for req_word in required_words):
                continue  # Try next group
        
        # Step 2b: Check normal words
        if normal_words:
            if not any(normal_word in title.lower() for normal_word in normal_words):
                continue  # Try next group
        
        # Both conditions met
        return True
    
    return False
```

### Examples

**Example 1:**
- Group: `["AI", "+技术"]`
- Title: "AI技术突破"
- Result: ✅ Matches (has "AI" AND has "技术")

**Example 2:**
- Group: `["AI", "+技术"]`
- Title: "AI应用广泛"
- Result: ❌ No match (has "AI" but missing "技术")

**Example 3:**
- Group: `["苹果", "!水果"]`
- Title: "苹果公司发布新iPhone"
- Result: ✅ Matches (has "苹果", no "水果")

**Example 4:**
- Group: `["苹果", "!水果"]`
- Title: "苹果水果价格下跌"
- Result: ❌ No match (contains filter word "水果")

---

## Sorting and Ranking Algorithm

### Multi-Level Sorting

News items are sorted at multiple levels:

### Level 1: Within Groups

**Sort Key:** `(-weight, min_rank, -count)`

1. **Primary:** Negative hotness weight (higher weight = higher priority)
2. **Secondary:** Minimum rank (lower rank = higher priority)
3. **Tertiary:** Negative count (more appearances = higher priority)

**Example:**
```
News A: weight=25.44, min_rank=1, count=5
News B: weight=20.00, min_rank=1, count=8
News C: weight=25.44, min_rank=3, count=3

Sorted: A, C, B
Reason: A and C have same weight, but A has lower min_rank
```

### Level 2: Between Groups

**Option A: Hotness First (Default)**
```python
stats.sort(key=lambda x: (-x["count"], x["position"]))
```

Groups sorted by:
1. Number of news items (descending)
2. Configuration position (ascending)

**Option B: Configuration Order First**
```python
stats.sort(key=lambda x: (x["position"], -x["count"]))
```

Groups sorted by:
1. Configuration position (ascending)
2. Number of news items (descending)

### Rank Display Formatting

**Function:** `format_rank_display()` (line 1135)

**Rules:**
- If minimum rank ≤ `rank_threshold` (default: 5):
  - Highlighted format: `**[rank]**` or `**[min-max]**`
- If minimum rank > `rank_threshold`:
  - Normal format: `[rank]` or `[min-max]`

**Example:**
- Ranks: [1, 2, 3], threshold: 5 → `**[1-3]**` (highlighted)
- Ranks: [7, 8, 9], threshold: 5 → `[7-9]` (normal)

---

## Data Flow and Processing Pipeline

### Complete Processing Flow

```
1. Crawl News
   ↓
2. Parse and Store (output/YYYY年MM月DD日/txt/*.txt)
   ↓
3. Load Historical Data (if exists)
   ↓
4. Identify New News (compare with previous crawl)
   ↓
5. Load Keyword Groups (config/frequency_words.txt)
   ↓
6. Match News to Groups (matches_word_groups)
   ↓
7. Calculate Hotness Scores (calculate_news_weight)
   ↓
8. Group and Sort News (count_word_frequency)
   ↓
9. Generate Report (prepare_report_data)
   ↓
10. Format Output (HTML, Text, Notifications)
   ↓
11. Send Notifications (if enabled)
```

### Data Structures

**News Item Structure:**
```python
{
    "title": str,
    "ranks": List[int],  # [1, 2, 3] - ranks across different crawls
    "count": int,        # Number of appearances
    "url": str,
    "mobileUrl": str,
    "first_time": str,   # "09时15分"
    "last_time": str,    # "10时45分"
    "source_name": str,  # Platform name
    "is_new": bool       # First appearance flag
}
```

**Group Statistics Structure:**
```python
{
    "word": str,              # Group key (e.g., "AI ChatGPT")
    "count": int,             # Number of matched news items
    "position": int,          # Position in configuration
    "titles": List[Dict],     # List of news items
    "percentage": float       # Percentage of total news
}
```

### Report Modes

**Mode 1: Daily Summary (`daily`)**
- Processes ALL news from the current day
- Includes news that appeared in previous crawls
- Shows cumulative statistics

**Mode 2: Current Ranking (`current`)**
- Processes only news in the latest crawl batch
- Shows current trending status
- May show same news multiple times if it stays in rankings

**Mode 3: Incremental Monitoring (`incremental`)**
- Processes ONLY new news items
- Excludes previously seen news
- Zero duplication

---

## Report Generation Modes

### Mode Comparison

| Mode | Data Source | Duplication | Use Case |
|------|------------|-------------|----------|
| `daily` | All news from today | Yes (includes previous) | Daily summary reports |
| `current` | Latest crawl batch | Yes (if still trending) | Real-time trend tracking |
| `incremental` | Only new news | No | Change detection |

### Mode-Specific Processing

**Daily Mode:**
```python
results_to_process = results  # All news
all_news_are_new = False
```

**Current Mode:**
```python
# Find latest timestamp
latest_time = max(all timestamps)

# Filter news with last_time == latest_time
results_to_process = filter_by_latest_time(results)
all_news_are_new = False
```

**Incremental Mode:**
```python
if is_first_today:
    results_to_process = results  # All news (first time)
    all_news_are_new = True
else:
    results_to_process = new_titles  # Only new
    all_news_are_new = True
```

---

## Configuration System

### Key Configuration Files

**1. `config/config.yaml`**

**Weight Configuration:**
```yaml
weight:
  rank_weight: 0.6      # 60% weight on ranking
  frequency_weight: 0.3 # 30% weight on frequency
  hotness_weight: 0.1   # 10% weight on hotness ratio
```

**Report Configuration:**
```yaml
report:
  mode: "incremental"           # daily|current|incremental
  rank_threshold: 5              # Highlight threshold
  sort_by_position_first: false  # Sort groups by position or hotness
  max_news_per_keyword: 0        # Global limit (0 = unlimited)
```

**2. `config/frequency_words.txt`**

**Syntax:**
- Normal words: `keyword`
- Required words: `+keyword` (must all be present)
- Filter words: `!keyword` (exclude if present)
- Max count: `@N` (limit to N items per group)
- Group separator: Empty line

**Example:**
```
AI
ChatGPT
+技术
@10

特斯拉
马斯克
!广告
```

---

## Summary

### Key Processes

1. **Hotness Calculation**: Combines rank, frequency, and hotness ratio with configurable weights
2. **Grouping**: Matches news to keyword groups using required/normal/filter word rules
3. **Sorting**: Multi-level sorting (within groups by weight, between groups by count/position)
4. **Filtering**: Applies display limits (group-specific or global)
5. **Mode Processing**: Handles three report modes (daily, current, incremental)

### Algorithm Characteristics

- **Weighted Scoring**: Three-component hotness score (60% rank, 30% frequency, 10% hotness)
- **Flexible Grouping**: Supports complex keyword matching rules
- **Configurable Sorting**: Can prioritize hotness or configuration order
- **Display Limits**: Per-group and global limits to control output size
- **Mode Flexibility**: Three distinct processing modes for different use cases

### Data Persistence

- News data stored in: `output/YYYY年MM月DD日/txt/*.txt`
- Each file contains platform-specific news with ranks and URLs
- Historical data used for trend analysis and new item detection

---

**Document Version:** 1.0  
**Last Updated:** Based on TrendRadar v3.4.0 codebase
