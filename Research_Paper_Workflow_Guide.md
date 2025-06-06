# Research Paper Monitoring Workflow Guide

## Overview
This n8n workflow automates the process of monitoring research papers from multiple sources, extracting and summarizing content, and distributing summaries through various channels.

## Workflow Components

### 1. 🕰️ Schedule Trigger
- **Purpose**: Runs the workflow every 6 hours
- **Configuration**: 
  - Interval: 6 hours
  - Can be adjusted based on your needs

### 2. 📚 Topic Watchers

#### ArXiv Watcher
- **Node Type**: HTTP Request
- **URL**: `http://export.arxiv.org/api/query`
- **Parameters**:
  ```
  search_query: cat:cs.AI OR cat:cs.CL OR cat:cs.LG
  start: 0
  max_results: 20
  sortBy: submittedDate
  sortOrder: descending
  ```

#### Semantic Scholar Watcher
- **Node Type**: HTTP Request
- **URL**: `https://api.semanticscholar.org/graph/v1/paper/search`
- **Parameters**:
  ```
  query: artificial intelligence machine learning
  limit: 10
  fields: paperId,title,abstract,authors,year,url,openAccessPdf
  ```

#### bioRxiv Watcher
- **Node Type**: HTTP Request  
- **URL**: `https://api.biorxiv.org/details/biorxiv/2024-01-01/2024-12-31`

### 3. 🔄 Data Parsers

#### ArXiv Parser (Code Node)
```javascript
// Parse ArXiv XML response
const xml2js = require('xml2js');
const parser = new xml2js.Parser();

if (items[0].xml) {
  return new Promise((resolve) => {
    parser.parseString(items[0].xml, (err, result) => {
      if (err) {
        resolve([{json: {error: 'Failed to parse XML'}}]);
        return;
      }
      
      const entries = result.feed?.entry || [];
      const papers = entries.map(entry => ({
        title: entry.title?.[0] || '',
        abstract: entry.summary?.[0] || '',
        authors: entry.author?.map(a => a.name?.[0]).join(', ') || '',
        published: entry.published?.[0] || '',
        pdf_url: entry.link?.find(l => l.$.type === 'application/pdf')?.$.href || '',
        source: 'arxiv'
      }));
      
      resolve(papers.map(paper => ({json: paper})));
    });
  });
}
return items;
```

#### Semantic Scholar Parser (Code Node)
```javascript
// Parse Semantic Scholar response
if (items[0].json?.data) {
  const papers = items[0].json.data.map(paper => ({
    title: paper.title || '',
    abstract: paper.abstract || '',
    authors: paper.authors?.map(a => a.name).join(', ') || '',
    published: paper.year?.toString() || '',
    pdf_url: paper.openAccessPdf?.url || paper.url || '',
    source: 'semantic_scholar'
  }));
  
  return papers.map(paper => ({json: paper}));
}
return items;
```

#### bioRxiv Parser (Code Node)
```javascript
// Parse bioRxiv response
if (items[0].json?.collection) {
  const papers = items[0].json.collection.slice(0, 10).map(paper => ({
    title: paper.title || '',
    abstract: paper.abstract || '',
    authors: paper.authors || '',
    published: paper.date || '',
    pdf_url: `https://www.biorxiv.org/content/10.1101/${paper.doi}v${paper.version}.full.pdf`,
    source: 'biorxiv'
  }));
  
  return papers.map(paper => ({json: paper}));
}
return items;
```

### 4. 📄 Paper Fetcher & Text Extraction

#### PDF Fetcher (HTTP Request)
- **URL**: `={{ $json.pdf_url }}`
- **Response Format**: File (binary)

#### GROBID Text Extractor (HTTP Request)
- **URL**: `http://localhost:8070/api/processFulltextDocument`
- **Method**: POST
- **Content-Type**: multipart/form-data
- **Body**: PDF file from previous step

#### Fallback PDF Parser (Code Node)
```javascript
// Fallback text extraction using simple PDF parsing
const pdf = require('pdf-parse');

if (items[0].binary?.data) {
  try {
    const data = await pdf(Buffer.from(items[0].binary.data, 'base64'));
    return [{
      json: {
        ...items[0].json,
        extracted_text: data.text.substring(0, 5000) // Limit to 5000 chars
      }
    }];
  } catch (error) {
    return [{
      json: {
        ...items[0].json,
        extracted_text: items[0].json.abstract || 'Could not extract text from PDF',
        extraction_error: error.message
      }
    }];
  }
}
return items;
```

### 5. 🤖 AI Summarizer

#### OpenAI Summarizer
- **Node Type**: OpenAI
- **Model**: gpt-4
- **System Prompt**: 
  ```
  You are a research paper summarizer. Create a concise, structured summary of the given research paper. Include: 1) Main contribution, 2) Key methods, 3) Results/findings, 4) Significance. Keep it under 300 words.
  ```
- **User Message**: 
  ```
  Title: {{ $json.title }}
  
  Abstract: {{ $json.abstract }}
  
  Extracted Text: {{ $json.extracted_text || $json.abstract }}
  ```

### 6. 📢 Output Channels

#### Notion Database
- **Operation**: Append to Database
- **Database ID**: Set via environment variable `NOTION_DATABASE_ID`
- **Properties**:
  - Title (title): `{{ $json.title }}`
  - Authors (rich text): `{{ $json.authors }}`
  - Source (select): `{{ $json.source }}`
  - Published (date): `{{ $json.published }}`
  - Summary (rich text): `{{ $('OpenAI Summarizer').item.json.choices[0].message.content }}`
  - PDF URL (url): `{{ $json.pdf_url }}`

#### Telegram Notification
- **Chat ID**: Set via environment variable `TELEGRAM_CHAT_ID`
- **Message**:
  ```
  📄 *New Research Paper*
  
  *{{ $json.title }}*
  
  👥 Authors: {{ $json.authors }}
  📅 Published: {{ $json.published }}
  🔍 Source: {{ $json.source }}
  
  📝 *Summary:*
  {{ $('OpenAI Summarizer').item.json.choices[0].message.content }}
  
  🔗 [Read PDF]({{ $json.pdf_url }})
  ```

#### Email Notification
- **Subject**: `New Research Paper: {{ $json.title }}`
- **HTML Body**:
  ```html
  <h2>{{ $json.title }}</h2>
  <p><strong>Authors:</strong> {{ $json.authors }}</p>
  <p><strong>Published:</strong> {{ $json.published }}</p>
  <p><strong>Source:</strong> {{ $json.source }}</p>
  
  <h3>Summary:</h3>
  <p>{{ $('OpenAI Summarizer').item.json.choices[0].message.content }}</p>
  
  <p><a href="{{ $json.pdf_url }}">Read Full Paper</a></p>
  ```

## Setup Requirements

### 1. External Services Setup

#### GROBID (Optional but Recommended)
```bash
# Run GROBID server for advanced PDF text extraction
docker run -d --rm --init --ulimit core=0 -p 8070:8070 lfoppiano/grobid:0.8.0
```

### 2. Environment Variables
Set these in n8n Settings > Variables:
```
NOTION_DATABASE_ID=your_notion_database_id
TELEGRAM_CHAT_ID=your_telegram_chat_id
EMAIL_RECIPIENT=your_email@example.com
```

### 3. Credentials Required
- **OpenAI API**: For paper summarization
- **Notion API**: For database storage
- **Telegram API**: For notifications
- **SMTP**: For email notifications

### 4. Node Dependencies
For the Code nodes, ensure these npm packages are available:
```bash
npm install xml2js pdf-parse
```

## Workflow Connections

```
Schedule Trigger
    ├── ArXiv Watcher → ArXiv Parser ┐
    ├── Semantic Scholar Watcher → Semantic Scholar Parser ├── Merge Papers
    └── bioRxiv Watcher → bioRxiv Parser ┘
                                         ↓
                                   Filter Papers with PDFs
                                         ↓
                                    PDF Fetcher
                                         ↓
                              GROBID Text Extractor
                                         ↓
                                  OpenAI Summarizer
                                         ↓
                              ┌─────────┼─────────┐
                              ↓         ↓         ↓
                        Notion    Telegram    Email
                      Database  Notification Notification
```

## Customization Options

### Search Queries
Modify the search queries in the watcher nodes:
- **ArXiv**: `cat:cs.AI OR cat:cs.CL OR cat:cs.LG`
- **Semantic Scholar**: `artificial intelligence machine learning`
- **bioRxiv**: Date range filtering

### Output Filtering
Add additional filtering nodes to:
- Filter by specific authors
- Filter by publication date
- Filter by keywords in title/abstract
- Remove duplicates

### Additional Output Channels
Consider adding:
- Slack notifications
- Discord webhooks
- RSS feed generation
- Airtable storage
- Google Sheets integration

## Testing and Deployment

1. **Test Individual Nodes**: Test each watcher and parser separately
2. **Verify Credentials**: Ensure all API credentials are working
3. **Test Full Workflow**: Run the complete workflow manually first
4. **Set Active**: Enable the schedule trigger for automated runs
5. **Monitor Logs**: Check execution logs for any errors

## Maintenance

- **Update Search Queries**: Adjust based on your research interests
- **Monitor API Limits**: Be aware of rate limits for external APIs
- **Review Summaries**: Periodically check AI summary quality
- **Update Dependencies**: Keep npm packages updated

This workflow provides a comprehensive automated research paper monitoring system that can be customized to your specific needs and interests. 