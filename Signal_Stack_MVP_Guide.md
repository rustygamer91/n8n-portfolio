# Signal Stack MVP - Daily AI Digest Setup Guide

## 🎯 Overview
Signal Stack MVP creates automated daily digests by monitoring multiple platforms (Reddit, HackerNews, Google News) for specific topics and delivering AI-powered summaries to Telegram and Notion.

## 📊 Workflow Architecture

```
📅 Daily Scheduler (24h intervals)
    ├── 📰 Google News RSS
    ├── 🔴 Reddit RSS Feed  
    └── 📊 HackerNews Algolia API
                ↓
        🔄 XML Parsers (Google News, Reddit)
                ↓
        💻 Data Processors (Clean & Structure)
                ↓
        🔗 Merge All Sources
                ↓
        🧹 Filter & Deduplicate (24hr window)
                ↓
        🤖 GPT Digest Generator
                ↓
        📱 Telegram + 📋 Notion Archive
```

## 🛠️ Component Breakdown

### 1. **Data Sources**

#### Google News Feed
- **URL**: `https://news.google.com/rss/search?q=generative+AI+ecommerce`
- **Format**: RSS/XML
- **Content**: News articles, press releases, formal announcements
- **Update Frequency**: Real-time

#### Reddit RSS Feed  
- **URL**: `https://www.reddit.com/search.rss?q=generative+AI+ecommerce&sort=new&t=day`
- **Format**: RSS/XML
- **Content**: Community discussions, opinions, informal updates
- **Filters**: Last 24 hours, sorted by newest

#### HackerNews API
- **URL**: `https://hn.algolia.com/api/v1/search?query=generative AI ecommerce&tags=story`
- **Format**: JSON
- **Content**: Tech-focused discussions, startup news
- **Filters**: Stories only, 10 results max

### 2. **Data Processing Pipeline**

#### XML Parsing (Google News & Reddit)
- Converts RSS/XML feeds to JSON
- Extracts: title, description, link, published date, author

#### JSON Processing (All Sources)
```javascript
// Standardized output format:
{
  title: "Article/Post Title",
  description: "Content summary/excerpt", 
  link: "URL to original content",
  published: "ISO date string",
  source: "Google News|Reddit|Hacker News",
  platform: "news|social|tech",
  author: "Author name (if available)"
}
```

#### Filter & Deduplication Logic
- **Time Filter**: Only content from last 24 hours
- **Deduplication**: Removes similar titles (first 5 words matching)
- **Prioritization**: Google News > HackerNews > Reddit
- **Limit**: Top 5-7 most relevant items

### 3. **AI Digest Generation**

#### GPT Prompt Strategy
```
System: You are Signal Stack, an AI that creates concise daily digests of developments in Generative AI for eCommerce. Create a 2-minute read digest that's insightful but accessible.

Structure:
🗞️ Signal Stack: GenAI x eCommerce — [Date]

[2-3 key items with bullets like:]
🛍️ [Headline] 
[1-2 sentence explanation of why it matters]

End with: Delivered daily. More insights → [link]

Keep it under 200 words total. Be engaging and focus on business impact.
```

#### Example Output
```
🗞️ Signal Stack: GenAI x eCommerce — June 5

🛍️ Amazon's new AI-generated product titles are live
Launching on US marketplace for softlines. Expect better CTRs and more A/B testing.

💰 YC-backed startup raises $8M for GenAI-driven eCom site search  
May disrupt Shopify's default search UX.

🧵 Reddit: "Is GenAI actually increasing conversions?" — top founder discussion
Key insights from r/Entrepreneur on real ROI measurements.

Delivered daily at 10AM PST. Archive here → [Notion]
```

## 🔧 Setup Instructions

### 1. **Import Workflow**
1. Download `workflows/signal-stack-mvp.json`
2. In n8n: Import from File
3. Configure credentials

### 2. **Required Credentials**
- **OpenAI API**: For digest generation
- **Telegram Bot**: For daily delivery
- **Notion API**: For archiving (optional)

### 3. **Configuration Options**

#### Change Topics/Keywords
Modify these URLs in the HTTP Request nodes:

**Google News:**
```
https://news.google.com/rss/search?q=[YOUR_KEYWORDS]
```

**Reddit:**
```  
https://www.reddit.com/search.rss?q=[YOUR_KEYWORDS]&sort=new&t=day
```

**HackerNews:**
```
https://hn.algolia.com/api/v1/search?query=[YOUR_KEYWORDS]&tags=story
```

#### Popular Topic Examples
- **AI Tools**: `AI+tools+productivity`
- **Climate Tech**: `climate+technology+sustainability`  
- **FinTech**: `fintech+banking+cryptocurrency`
- **HealthTech**: `healthtech+medical+AI`
- **EdTech**: `education+technology+learning`

### 4. **Output Channels**

#### Telegram Setup
1. Create Telegram bot via @BotFather
2. Get bot token
3. Add to channel/group and get Chat ID
4. Configure in Telegram node

#### Notion Archive Setup
1. Create Notion integration
2. Create database with properties:
   - Date (Title)
   - Digest (Rich Text)
   - Sources Count (Number)
   - Created (Date)
3. Add database ID to Notion node

### 5. **Scheduling Options**

#### Standard Schedule
- **Daily at 10AM**: `0 10 * * *`
- **Twice daily**: `0 10,18 * * *`
- **Weekdays only**: `0 10 * * 1-5`

#### Test Schedule (for setup)
- **Every hour**: Change to 1 hour intervals
- **Manual trigger**: Disable schedule, run manually

## 🎨 Customization Ideas

### **Multi-Topic Channels**
Create separate workflows for different interests:
- 🚗 `@ai_automotive_daily`
- 🏥 `@healthtech_digest`  
- 💰 `@fintech_signals`

### **Advanced Filtering**
Add Code nodes to filter by:
- **Sentiment analysis** (positive/negative news)
- **Company mentions** (track specific companies)
- **Funding announcements** (track VC activity)
- **Keyword density** (technical vs business content)

### **Enhanced Output Formats**

#### Telegram with Rich Formatting
```
🗞️ *Signal Stack: GenAI x eCommerce*
📅 June 5, 2024

🔥 *Top Signal*
🛍️ [Amazon's AI product titles go live](link)
_Why it matters: First major rollout could set industry standard_

💡 *Worth Watching*  
🚀 [Startup raises $8M for AI search](link)
_Potential Shopify disruptor with YC backing_

📊 *Community Pulse*
💬 ["Is GenAI ROI real?" - Reddit discussion](link)
_Founder insights on actual conversion data_

---
📈 Daily at 10AM PST | 📚 [Archive](notion-link)
```

#### Notion Database Enhancement
Add properties:
- **Sentiment** (Select: Positive/Neutral/Negative)
- **Category** (Select: Product/Funding/Discussion/Research)
- **Companies** (Multi-select: Amazon, Shopify, etc.)
- **Impact Score** (Number: 1-10)

## 📊 Analytics & Monitoring

### **Track Performance**
Monitor in Notion dashboard:
- **Daily source distribution** (Reddit vs HN vs News)
- **Topic trends** over time
- **Engagement patterns** (if using public channels)

### **Quality Metrics**
- **Relevance score** (manual review of digests)
- **Source diversity** (avoid single-source days)
- **Freshness** (ensure 24hr content only)

## 🚀 Scaling Options

### **Additional Sources**
- **ProductHunt**: Daily product launches
- **YCombinator**: Startup news  
- **TechCrunch**: Funding announcements
- **LinkedIn**: Professional discussions
- **Discord/Slack**: Community channels

### **Advanced Features**
- **Sentiment tracking** over time
- **Company mention alerts** for specific companies
- **Market impact correlation** (stock prices, funding)
- **Geographic filtering** (US vs Europe vs Asia news)

### **Distribution Expansion**
- **Email newsletters** (Substack/Mailchimp)
- **RSS feed** generation
- **Website integration** (embed digests)
- **Social media** auto-posting

## 🔧 Troubleshooting

### **Common Issues**

#### No Content Found
- Check RSS feed URLs manually
- Verify search keywords are popular enough
- Adjust time filters (expand to 48 hours)

#### Duplicate Content
- Review deduplication logic
- Adjust title similarity threshold
- Add domain-based filtering

#### Poor Digest Quality
- Refine GPT prompt for your topic
- Add context about target audience
- Include examples in system prompt

#### Rate Limiting
- Add delays between API calls
- Implement retry logic
- Monitor API usage

### **Testing Strategy**
1. **Test individual sources** first
2. **Run with 1-hour schedule** during setup
3. **Check output quality** before enabling daily schedule
4. **Monitor for first week** and adjust filters

## 📋 Maintenance Checklist

### **Weekly**
- [ ] Review digest quality
- [ ] Check for broken RSS feeds
- [ ] Monitor API usage/costs
- [ ] Adjust keywords if needed

### **Monthly**
- [ ] Analyze topic trends
- [ ] Update search keywords
- [ ] Review source performance
- [ ] Optimize GPT prompts

This Signal Stack MVP provides a solid foundation for automated content monitoring and can be easily customized for any topic or industry! 🚀 