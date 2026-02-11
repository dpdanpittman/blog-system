# Automated AI Blog System

A fully automated, 5-agent system that discovers trending topics, researches them, writes long-form blog posts, edits for quality, and publishes to your blog with intelligent learning capabilities.

## Requirements

Before you begin, you'll need:

1. **n8n instance** - Self-hosted or cloud n8n installation
2. **Local LLM via Ollama** - The system uses Ollama to run a local language model for content generation. You'll need:
   - Ollama installed and running on your network
   - A capable model (the workflows use `gpt-oss:20b` but you can substitute any model)
   - **Note: The Ollama endpoint is hardcoded in the workflows as `http://192.168.6.56:11434` - you must update this to your Ollama instance URL**
3. **Blog website with API** - A blog that accepts posts via REST API with Bearer token authentication. The workflows expect:
   - POST endpoint for creating posts (receives: slug, title, date, content, metaDescription, keywords, readingTime)
   - GET endpoint for fetching recent posts (returns array of posts with title and slug fields for duplicate detection)
   - **Note: The blog API URL is hardcoded as `https://mabus.ai/api/blog/posts` - you must update this to your blog's API**
4. **SerpApi account** - For web research (free tier available at serpapi.com)

## 🚀 System Overview

This system runs completely autonomously:
1. **Agent 1** discovers trending tech topics from HackerNews (with duplicate detection)
2. **Agent 2** researches topics using web search APIs
3. **Agent 3** writes 2500-5000 word blog posts (learns from past failures)
4. **Agent 4** edits for quality, grammar, and third-person journalistic style
5. **Agent 5** validates, publishes to your blog API, and logs metrics for continuous learning

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│             Automated Blog Generation Pipeline              │
└─────────────────────────────────────────────────────────────┘

Agent 1: Topic Discovery (Schedule Trigger - Daily)
   ├─ Fetch HackerNews top 30 stories
   ├─ Filter for AI/tech keywords
   ├─ Check for duplicates (last 30 blog posts)
   ├─ Enhanced duplicate detection (word + key entity overlap)
   └─ Select unique topic → Agent 2
        │
        ↓
Agent 2: Research (Webhook: /webhook/agent-2-research)
   ├─ Search via SerpApi
   ├─ Extract top 5 sources
   └─ Synthesize research summary → Agent 3
        │
        ↓
Agent 3: Writing (Webhook: /webhook/agent-3-writing)
   ├─ Load Data Tables (validation failures, quality metrics)
   ├─ Analyze learnings (common issues, quality benchmarks)
   ├─ Build enhanced prompt with learnings
   ├─ Write 2500-5000 word long-form post (third-person)
   └─ Generate markdown → Agent 4
        │
        ↓
Agent 4: Editing (Webhook: /webhook/agent-4-editing)
   ├─ Review for grammar, clarity, and flow
   ├─ Enforce third-person journalistic style
   ├─ Expand content to meet 2500-5000 word target
   └─ Output final edited post → Agent 5
        │
        ↓
Agent 5: Publishing (Webhook: /webhook/agent-5-publishing)
   ├─ Validate quality (word count, title, sections, POV)
   ├─ Log validation failures to Data Tables (for Agent 3 learning)
   ├─ Convert markdown to HTML
   ├─ Generate SEO metadata (description, keywords)
   ├─ Calculate quality score
   ├─ Write quality metrics to Data Tables
   ├─ Publish to Blog API
   └─ Success! (Logs published post data)
```

## 📋 Key Features

### Automated Topic Discovery
- **HackerNews Integration**: Fetches top 30 trending stories daily
- **Smart Filtering**: Filters for 35+ tech keywords (AI, blockchain, security, etc.)
- **Duplicate Detection**: Two-pronged approach
  - Word overlap check (40% threshold)
  - Key entity overlap (2+ shared proper nouns/technical terms)
- **Stops on All Duplicates**: Throws error if all topics are duplicates

### Intelligent Learning System
- **Agent 3 Learns from Past Failures**:
  - Loads validation failures from Data Tables before writing
  - Identifies common issues (e.g., word_count_low, first_person_language)
  - Adjusts writing prompt with "COMMON ISSUES TO AVOID"
  - Includes quality benchmarks from past posts
- **Agent 5 Tracks Everything**:
  - Logs all validation failures with severity levels
  - Records quality metrics (word count, sections, scores)
  - Monitors retry patterns
  - Tracks source reliability
  - Categories topic performance

### Long-Form Content
- **2500-5000 Word Posts**: Substantial, comprehensive articles
- **Third-Person Journalistic Style**: Professional news analysis perspective
- **Structured Content**: 6-10 main sections with subsections
- **Enhanced Prompts**: Both Agent 3 and Agent 4 enforce long-form depth

### Quality & SEO
- **Real-Time Validation**: Checks title, word count, sections, first-person language
- **Dynamic Quality Scoring**: 0-10 score based on multiple criteria
- **SEO Metadata**: Auto-generates meta descriptions and keywords
- **Severity Classification**: Critical, High, Medium issue levels

## 📦 Files

- `agent-1-topic-discovery.json` - Automated topic discovery with duplicate detection
- `agent-2-research.json` - SerpApi research agent
- `agent-3-writing.json` - Writing agent with Data Tables learning
- `agent-4-editing.json` - Editing agent with long-form enforcement
- `agent-5-publishing.json` - Publishing agent with validation, SEO, and Data Tables logging

## 🚀 Setup Instructions

### Prerequisites

- **n8n instance** running (self-hosted or cloud)
- **Ollama** running on your network with a capable LLM loaded
- **SerpApi account** with API key (get one at serpapi.com)
- **Blog with REST API** that supports Bearer token authentication

### Hardcoded Values You Must Change

The following values are hardcoded in the workflow JSON files and **must be updated** before use:

| Value | Current | Update To | Found In |
|-------|---------|-----------|----------|
| Ollama endpoint | `http://192.168.6.56:11434` | Your Ollama URL | Agent 3, Agent 4 |
| Ollama model | `gpt-oss:20b` | Your model name | Agent 3, Agent 4 |
| Blog API URL | `https://mabus.ai/api/blog/posts` | Your blog API | Agent 1, Agent 5 |
| n8n webhook base | `https://n8n.mabus.ai` | Your n8n URL | Agent 1, 2, 3, 4 |

You can find and replace these using your text editor before importing, or update them in the n8n UI after import.

### Step 1: Create Data Tables (MANUAL)

**IMPORTANT**: You must manually create these 3 Data Tables in n8n **before** importing the workflows.

Go to n8n → Data Tables → Create New Table:

#### Table 1: `blog_validation_failures`

Stores validation errors for Agent 3 to learn from.

| Column Name    | Type     | Description                               |
|---------------|----------|-------------------------------------------|
| `id`          | Number   | Auto-increment primary key                |
| `createdAt`   | DateTime | Timestamp of failure                      |
| `post_title`  | String   | Title of the failed post                  |
| `issue_type`  | String   | Type of issue (e.g., word_count_low)      |
| `severity`    | String   | critical, high, medium                    |
| `message`     | String   | Detailed error message                    |

#### Table 2: `blog_quality_metrics`

Stores quality scores and metrics for benchmarking.

| Column Name      | Type     | Description                            |
|-----------------|----------|----------------------------------------|
| `id`            | Number   | Auto-increment primary key             |
| `createdAt`     | DateTime | Timestamp of publication               |
| `post_title`    | String   | Title of published post                |
| `post_slug`     | String   | URL slug                               |
| `word_count`    | Number   | Total word count                       |
| `section_count` | Number   | Number of sections                     |
| `quality_score` | Number   | Quality score (0-10)                   |

#### Table 3: `blog_retry_patterns`

Stores retry information for monitoring.

| Column Name     | Type     | Description                             |
|----------------|----------|-----------------------------------------|
| `id`           | Number   | Auto-increment primary key              |
| `createdAt`    | DateTime | Timestamp of retry                      |
| `post_title`   | String   | Title of retried post                   |
| `retry_count`  | Number   | Number of retries attempted             |
| `final_status` | String   | success or failed                       |

**Note**: These tables can be empty initially. They will populate as the system runs and learns from experience.

### Step 2: Import Workflows

Import all 5 workflows into n8n:
1. `agent-1-topic-discovery.json`
2. `agent-2-research.json`
3. `agent-3-writing.json`
4. `agent-4-editing.json`
5. `agent-5-publishing.json`

### Step 3: Configure Agent 3 Data Table Nodes (MANUAL)

**Agent 3** has 3 Data Table nodes that need manual configuration after import.

Open **Agent 3** workflow and configure these nodes:

#### Node: "Get Validation Failures"
- **Operation**: Get
- **Table**: Select `blog_validation_failures` from dropdown
- **Options** → **Limit**: 100
- **Options** → **Sort**:
  - Field Name: `createdAt`
  - Order: DESC

#### Node: "Get Quality Metrics"
- **Operation**: Get
- **Table**: Select `blog_quality_metrics` from dropdown
- **Options** → **Limit**: 100
- **Options** → **Sort**:
  - Field Name: `createdAt`
  - Order: DESC

#### Node: "Get Retry Patterns"
- **Operation**: Get
- **Table**: Select `blog_retry_patterns` from dropdown
- **Options** → **Limit**: 100
- **Options** → **Sort**:
  - Field Name: `createdAt`
  - Order: DESC

### Step 4: Configure Agent 5 Data Table Nodes (MANUAL)

**Agent 5** has 3 Data Table nodes that need manual configuration after import.

Open **Agent 5** workflow and configure these nodes:

#### Node: "Log Validation Failure"
- **Operation**: Insert
- **Table**: Select `blog_validation_failures` from dropdown
- **Data Mode**: Define Below
- **Fields to Send**:
  - `createdAt` = `{{ $now.toISO() }}`
  - `post_title` = `{{ $json.topic }}`
  - `issue_type` = `{{ $json.issue }}`
  - `severity` = `{{ $json.severity }}`
  - `message` = `{{ $json.message }}`

#### Node: "Write Quality Metrics"
- **Operation**: Insert
- **Table**: Select `blog_quality_metrics` from dropdown
- **Data Mode**: Define Below
- **Fields to Send**:
  - `createdAt` = `{{ $now.toISO() }}`
  - `post_title` = `{{ $json.title }}`
  - `post_slug` = `{{ $json.slug }}`
  - `word_count` = `{{ $json.wordCount }}`
  - `section_count` = `{{ $json.sectionCount }}`
  - `quality_score` = `{{ $json.qualityScore }}`

#### Node: "Write Retry Pattern"
- **Operation**: Insert
- **Table**: Select `blog_retry_patterns` from dropdown
- **Data Mode**: Define Below
- **Fields to Send**:
  - `createdAt` = `{{ $now.toISO() }}`
  - `post_title` = `{{ $json.topic }}`
  - `retry_count` = `{{ $json.retryCount }}`
  - `final_status` = `{{ $json.status }}`

**Conditions Node: "Should Retry?"**

This node checks if the post failed validation and should retry:
- **IF**: `{{ $json.validationPassed }}` equals `false`
- **AND**: `{{ $json.retryCount }}` is less than `3`
- **THEN**: Route to retry (Agent 3)
- **ELSE**: Route to error logging

### Step 5: Configure Credentials

#### SerpApi (Agent 2)
1. Go to Agent 2 → "Search SerpApi" node
2. Add SerpApi credential with your API key

#### Blog API (Agent 5)
1. Set environment variable: `BLOG_API_KEY=your_bearer_token`
2. Verify Agent 5 → "Publish to Blog API" node uses:
   ```
   Authorization: Bearer {{ $env.BLOG_API_KEY }}
   ```

#### Ollama (Agent 3 & 4)
- **You must update the Ollama endpoint URL** - it's hardcoded as `http://192.168.6.56:11434`
- Find all HTTP Request nodes that call Ollama and update to your instance
- Model: Change `gpt-oss:20b` to your preferred model (e.g., `llama3:70b`, `mixtral:8x7b`, etc.)
- Recommended: Use a model with good long-form writing capabilities

### Step 6: Configure Webhook URLs

Update webhook URLs in each agent's "Call Agent X" node:

**Agent 1** → "Call Agent 2":
```
https://n8n.mabus.ai/webhook/agent-2-research
```

**Agent 2** → "Call Agent 3":
```
https://n8n.mabus.ai/webhook/agent-3-writing
```

**Agent 3** → "Call Agent 4":
```
https://n8n.mabus.ai/webhook/agent-4-editing
```

**Agent 4** → "Call Agent 5":
```
https://n8n.mabus.ai/webhook/agent-5-publishing
```

Replace `n8n.mabus.ai` with your actual n8n instance URL.

### Step 7: Set Schedule (Agent 1)

In **Agent 1**, configure the "Schedule Trigger" node:
- **Default**: Runs every 24 hours (daily)
- **Recommended**: Set to run at 8 AM your timezone
- Example cron: `0 8 * * *` (8 AM daily)

### Step 8: Activate Workflows

Activate workflows in **reverse order** (so webhooks are ready):

1. ✅ Agent 5 (Publishing)
2. ✅ Agent 4 (Editing)
3. ✅ Agent 3 (Writing)
4. ✅ Agent 2 (Research)
5. ✅ Agent 1 (Topic Discovery)

## 🧪 Testing

### Manual Test (Start from Agent 2)

Skip topic discovery and test the core pipeline:

```bash
curl -X POST https://n8n.mabus.ai/webhook/agent-2-research \
  -H "Content-Type: application/json" \
  -d '{
    "topic": "The Rise of AI Agents in 2026",
    "url": "https://example.com/article",
    "style": "accessible-explanation",
    "length": "long",
    "angle": "general overview"
  }'
```

### Full Automated Test

Let Agent 1 run on schedule:
1. Agent 1 discovers topic from HackerNews
2. Checks for duplicates against last 30 blog posts
3. If unique, triggers Agent 2 → 3 → 4 → 5
4. Blog post published to `https://mabus.ai/blog/[slug]`

### Verify Data Tables

After first few runs, check Data Tables to see learning in action:

```sql
-- In n8n Data Tables UI:

-- Check validation failures
SELECT * FROM blog_validation_failures ORDER BY createdAt DESC LIMIT 10;

-- Check quality metrics
SELECT * FROM blog_quality_metrics ORDER BY createdAt DESC LIMIT 10;

-- Check retry patterns
SELECT * FROM blog_retry_patterns ORDER BY createdAt DESC LIMIT 10;
```

### Expected Behavior

**First Run**:
- Word count might be low (1500-2000 words)
- Quality score: 6-7/10
- May have first-person language issues
- Validation failures logged to Data Tables

**After 5-10 Runs**:
- Word count increases to 2500-4000 words
- Quality score improves to 7-9/10
- First-person issues decrease
- Agent 3 learns from failures and adjusts

## 📊 Monitoring

### Quality Metrics Dashboard

Check `blog_quality_metrics` Data Table:
- **Average word count**: Should be 2500-5000
- **Average sections**: Should be 6-10
- **Average quality score**: Should improve to 7-9/10

### Validation Failures

Check `blog_validation_failures` Data Table:
- **Common issues**: First few posts may have word_count_low, first_person_language
- **Improvement**: These should decrease as Agent 3 learns
- **Severity**: Critical issues should be near zero

### Retry Patterns

Check `blog_retry_patterns` Data Table:
- **Retry rate**: Should be <15% after system learns
- **Final status**: Should show mostly "success"

## 🛠️ Customization

### Change Topic Keywords

**Agent 1** → "Filter for AI/Tech Topics" node:

```javascript
const keywords = [
  'ai', 'ml', 'machine learning', 'artificial intelligence',
  'llm', 'gpt', 'model', 'neural', 'deep learning',
  'data science', 'automation', 'robotics', 'quantum',
  'tech', 'software', 'programming', 'algorithm',
  'computing', 'developer', 'api', 'open source',
  'security', 'privacy', 'crypto', 'blockchain',
  'startup', 'cloud', 'database', 'web', 'mobile',
  'app', 'code', 'language', 'framework', 'tool', 'platform'
];
```

Add or remove keywords based on your blog's focus.

### Adjust Duplicate Detection Threshold

**Agent 1** → "Check for Duplicates" node:

```javascript
// Word overlap threshold (default: 40%)
const isDuplicateByWords = wordOverlap > 0.4;

// Key entity threshold (default: 2+ shared entities)
const isDuplicateByKeys = keyMatches.length >= 2;
```

Increase thresholds for stricter duplicate detection.

### Change Word Count Targets

**Agent 3** → "Build Writing Prompt" node:

```javascript
const prompt = `...
LENGTH: LONG-FORM (aim for 2500-5000 words - this is critical!)
...`;
```

**Agent 4** → "Build Edit Prompt" node:

```javascript
const prompt = `...
- DO NOT shorten or summarize - EXPAND and add detail
- Target length: 2500-5000 words
...`;
```

### Modify Quality Validation

**Agent 5** → "Validate Quality" node:

```javascript
// Word count validation
if (wordCount < 1500) {
  issues.push({
    issue: 'word_count_low',
    severity: 'high',
    message: `Post is too short (${wordCount} words). Target: 2500-5000.`
  });
}
```

Adjust thresholds based on your requirements.

## 🚨 Troubleshooting

### Agent 1: No topics found
- **Check**: HackerNews API is accessible
- **Check**: Keyword filter isn't too restrictive
- **Fix**: Broaden keyword list or increase story fetch count

### Agent 1: All topics are duplicates
- **Expected**: If running frequently (hourly/daily), HN top stories don't change much
- **Fix**: Reduce duplicate detection threshold or increase fetch count to 50 stories
- **Note**: System throws error and stops (prevents duplicate posts)

### Agent 3: Posts too short
- **Check**: Data Tables have quality metrics to learn from
- **Check**: Ollama model has enough context window
- **Fix**: Wait for system to learn after 5-10 runs, or manually adjust prompts

### Agent 5: Validation failures
- **Normal**: First few posts may have issues (system is learning)
- **Check**: Validation failures are being logged to Data Tables
- **Expected**: Failures decrease after 10-15 posts as Agent 3 learns

### Data Tables empty
- **Check**: Agent 3 and Agent 5 Data Table nodes are configured
- **Check**: Tables were created with correct column names
- **Fix**: Verify "Write Quality Metrics" and "Log Validation Failure" nodes in Agent 5

### Ollama timeouts
- **Check**: Model is loaded: `ollama list`
- **Check**: Enough GPU memory for model
- **Fix**: Increase timeout in HTTP Request nodes (default: 600000ms = 10min)

## 📈 Expected Results

### First Week
- **Blog posts**: 7 posts (1 per day)
- **Quality**: 6-7/10 average
- **Word count**: 1500-2500 words average
- **Issues**: Some first-person language, occasional short posts

### After One Month
- **Blog posts**: 30 posts
- **Quality**: 7-9/10 average
- **Word count**: 2500-4000 words average
- **Issues**: Minimal validation failures
- **Learning**: Agent 3 consistently avoids common mistakes

### Long-Term Benefits
- **Consistency**: Professional, third-person journalistic style
- **SEO**: Rich, long-form content with proper metadata
- **Efficiency**: <15% retry rate
- **Quality**: Continuous improvement from Data Tables learning

## 🏆 Success Criteria

Your system is working well when:
- ✅ 85%+ of posts pass validation on first try
- ✅ Average quality score is 7.5+/10
- ✅ Average word count is 2500-4000 words
- ✅ Zero first-person language issues
- ✅ Duplicate detection prevents re-publishing same topics
- ✅ Data Tables show decreasing validation failures over time

## 🤝 Contributing

To improve this system:
1. Tune Agent 3 writing prompts based on your blog style
2. Adjust validation rules in Agent 5 based on your quality standards
3. Add new keywords to Agent 1 based on trending topics
4. Modify duplicate detection thresholds based on your publishing frequency

---

**Questions?** Check n8n execution logs for detailed error messages and debug information. Each agent logs its progress with emojis (✅, 📊, 🔍, etc.) for easy tracking.

**Need help?** Review the "Build Writing Prompt" and "Validate Quality" nodes - they contain the core logic and are well-commented.
