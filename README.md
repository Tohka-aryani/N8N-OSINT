# Content Monitoring and Risk Classification Workflow

## Overview

This n8n workflow automates content monitoring and risk classification from both Telegram and Webhook sources. It searches Reddit and Google News, processes the results, performs AI-powered risk analysis, and saves categorized data to Google Sheets.

Initial workflow is created using the following Model Context Protocol (MCP): https://github.com/czlonkowski/n8n-mcp

## Workflow Architecture

### Trigger Nodes

1. **Telegram Trigger** (`telegram-trigger`)
   - **Type**: `n8n-nodes-base.telegramTrigger`
   - **Configuration**: 
     - Triggers on: `message`, `channel_post`
   - **Purpose**: Captures messages from Telegram channels or chats
   - **Credentials Required**: Telegram Bot Token

2. **Webhook Trigger** (`webhook-trigger`)
   - **Type**: `n8n-nodes-base.webhook`
   - **Configuration**:
     - HTTP Method: `POST`
     - Path: `content-monitor`
     - Response Mode: `lastNode`
   - **Purpose**: Receives data from external systems via webhook
   - **Webhook URL**: Generated when workflow is activated

### Data Extraction & Search

3. **Extract Search Query** (`extract-query-node`)
   - **Type**: `n8n-nodes-base.set`
   - **Purpose**: Extracts search query/keywords from incoming data
   - **Fields Extracted**:
     - `searchQuery`: From Telegram message text, channel post, or webhook body
     - `source`: Identifies trigger source (`telegram` or `webhook`)

4. **Search Reddit** (`reddit-search`)
   - **Type**: `n8n-nodes-base.httpRequest`
   - **Configuration**:
     - Method: `GET`
     - URL: `https://www.reddit.com/search.json?q={searchQuery}&limit=10&sort=relevance`
   - **Purpose**: Searches Reddit for content matching the query

5. **Search Google News** (`google-news-search`)
   - **Type**: `n8n-nodes-base.httpRequest`
   - **Configuration**:
     - Method: `GET`
     - URL: `https://news.google.com/rss/search?q={searchQuery}&hl=en&gl=US&ceid=US:en`
   - **Purpose**: Searches Google News RSS feed for matching articles

### Data Processing

6. **Process Reddit Results** (`process-reddit`)
   - **Type**: `n8n-nodes-base.set`
   - **Purpose**: Normalizes Reddit search results
   - **Output Fields**:
     - `title`: Post title
     - `source`: "Reddit"
     - `date`: Creation date (ISO format)
     - `url`: Post URL
     - `content`: Post content or title
     - `searchQuery`: Original search query

7. **Process Google News Results** (`process-google-news`)
   - **Type**: `n8n-nodes-base.set`
   - **Purpose**: Normalizes Google News RSS feed results
   - **Output Fields**:
     - `title`: Article title
     - `source`: "Google News"
     - `date`: Publication date
     - `url`: Article URL
     - `content`: Article description or title
     - `searchQuery`: Original search query

8. **Merge Results** (`merge-results`)
   - **Type**: `n8n-nodes-base.merge`
   - **Mode**: `append`
   - **Purpose**: Combines Reddit and Google News results into a single dataset

9. **Organize Data** (`organize-data`)
   - **Type**: `n8n-nodes-base.set`
   - **Purpose**: Consolidates data for AI analysis
   - **Output Fields**:
     - `consolidatedData`: JSON string of all results
     - `searchQuery`: Original search query
     - `sourceType`: Trigger source type

### Routing & AI Analysis

10. **Route by Source** (`route-by-source`)
    - **Type**: `n8n-nodes-base.switch`
    - **Purpose**: Routes data based on trigger source
    - **Outputs**:
      - **Telegram branch**: Routes to Agent Alpha
      - **Webhook branch**: Routes to Clean Webhook Data → Agent Bravo

11. **Clean Webhook Data** (`clean-webhook-data`)
    - **Type**: `n8n-nodes-base.set`
    - **Purpose**: Organizes webhook data into tables and removes markdown
    - **Output Fields**:
      - `cleanedContent`: Content with markdown removed
      - `tableData`: Formatted table data

12. **Agent Alpha** (`agent-alpha`)
    - **Type**: `@n8n/n8n-nodes-langchain.agent`
    - **Purpose**: AI agent for Telegram flow risk analysis
    - **Configuration**:
      - System Message: "You are Agent Alpha, a content risk analysis specialist..."
      - Prompt: Analyzes consolidated data and returns JSON with risk classification
    - **Connected To**: OpenAI Chat Model (via `ai_languageModel` connection)

13. **Agent Bravo** (`agent-bravo`)
    - **Type**: `@n8n/n8n-nodes-langchain.agent`
    - **Purpose**: AI agent for Webhook flow risk analysis
    - **Configuration**:
      - System Message: "You are Agent Bravo, a content risk analysis specialist..."
      - Prompt: Analyzes consolidated data and returns JSON with risk classification
    - **Connected To**: OpenAI Chat Model (via `ai_languageModel` connection)

14. **OpenAI Chat Model** (`openai-chat-model`)
    - **Type**: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - **Model**: `gpt-4o-2024-08-06`
    - **Purpose**: Provides language model capabilities to AI agents
    - **Credentials Required**: OpenAI API Key

### Risk Categorization & Storage

15. **Risk Categorization** (`risk-categorization`)
    - **Type**: `n8n-nodes-base.if`
    - **Purpose**: Separates High Risk from Medium/Low Risk items
    - **Condition**: `riskLevel === "High"`
    - **Outputs**:
      - **TRUE branch**: High Risk → Save High Risk
      - **FALSE branch**: Medium/Low Risk → Save Medium/Low Risk

16. **Save High Risk** (`save-high-risk`)
    - **Type**: `n8n-nodes-base.googleSheets`
    - **Operation**: `append`
    - **Sheet**: "High Risk"
    - **Purpose**: Saves high-risk items to Google Sheets
    - **Fields Saved**:
      - `searchQuery`, `title`, `source`, `date`, `url`
      - `riskLevel`, `riskCategory`, `reasoning`, `indicators`
      - `analyzedAt`
    - **Credentials Required**: Google Sheets OAuth2

17. **Save Medium/Low Risk** (`save-medium-low-risk`)
    - **Type**: `n8n-nodes-base.googleSheets`
    - **Operation**: `append`
    - **Sheet**: "Medium/Low Risk"
    - **Purpose**: Saves medium/low-risk items to Google Sheets
    - **Fields Saved**: Same as High Risk sheet
    - **Credentials Required**: Google Sheets OAuth2

### Notifications & Responses

18. **Telegram Notification** (`telegram-notification`)
    - **Type**: `n8n-nodes-base.telegram`
    - **Operation**: `sendMessage`
    - **Purpose**: Sends summary notification to Telegram
    - **Message Includes**:
      - Search query
      - Risk level and category
      - Reasoning
      - High risk alert (if applicable)
    - **Credentials Required**: Telegram Bot Token

19. **Respond to Webhook** (`respond-to-webhook`)
    - **Type**: `n8n-nodes-base.respondToWebhook`
    - **Purpose**: Returns JSON response to webhook caller
    - **Response Includes**:
      - `status`: "processed"
      - `searchQuery`: Original query
      - `riskLevel`: Assessed risk level
      - `analyzedAt`: Timestamp

## Workflow Flow

### Telegram Flow:
```
Telegram Trigger → Extract Search Query → [Search Reddit, Search Google News]
  → Process Reddit Results → Merge Results → Process Google News Results
  → Organize Data → Route by Source (Telegram) → Agent Alpha
  → OpenAI Chat Model → Risk Categorization
  → [Save High Risk | Save Medium/Low Risk]
  → Telegram Notification
```

### Webhook Flow:
```
Webhook Trigger → Extract Search Query → [Search Reddit, Search Google News]
  → Process Reddit Results → Merge Results → Process Google News Results
  → Organize Data → Route by Source (Webhook) → Clean Webhook Data
  → Agent Bravo → OpenAI Chat Model → Risk Categorization
  → [Save High Risk | Save Medium/Low Risk]
  → [Telegram Notification, Respond to Webhook]
```

## Setup Instructions

### 1. Prerequisites

- n8n instance (self-hosted or cloud)
- Telegram Bot Token (from @BotFather)
- OpenAI API Key
- Google Sheets API credentials
- Google Sheet with two sheets: "High Risk" and "Medium/Low Risk"

### 2. Google Sheets Setup

Create a Google Sheet with the following structure:

**Sheet: "High Risk"**
| searchQuery | title | source | date | url | riskLevel | riskCategory | reasoning | indicators | analyzedAt |

**Sheet: "Medium/Low Risk"**
| searchQuery | title | source | date | url | riskLevel | riskCategory | reasoning | indicators | analyzedAt |

### 3. Credentials Configuration

1. **Telegram Bot Token**:
   - Create a bot via @BotFather on Telegram
   - Copy the bot token
   - Add as credential in n8n: Telegram → Bot Token

2. **OpenAI API Key**:
   - Get API key from https://platform.openai.com/api-keys
   - Add as credential in n8n: OpenAI → API Key

3. **Google Sheets OAuth2**:
   - Set up OAuth2 credentials in Google Cloud Console
   - Add as credential in n8n: Google Sheets → OAuth2
   - Grant access to your Google Sheet

### 4. Workflow Configuration

1. Import the workflow JSON file into n8n
2. Update the following node configurations:

   **Google Sheets Nodes** (`save-high-risk`, `save-medium-low-risk`):
   - Set `documentId` to your Google Sheet ID
   - Verify sheet names match your Google Sheet

   **Telegram Notification** (`telegram-notification`):
   - Set `chatId` to your Telegram chat ID (use @get_id_bot to find it)
   - Or use environment variable: `$env.TELEGRAM_CHAT_ID`

3. Activate the workflow

### 5. Testing

**Test Telegram Flow:**
1. Send a message to your Telegram bot or channel
2. Message should trigger the workflow
3. Check Google Sheets for results
4. Verify Telegram notification received

**Test Webhook Flow:**
1. Get webhook URL from n8n (shown when workflow is active)
2. Send POST request:
   ```json
   {
     "query": "test search term",
     "body": {
       "query": "test search term"
     }
   }
   ```
3. Check Google Sheets for results
4. Verify webhook response received

## AI Agent Configuration

### Agent Alpha (Telegram Flow)
- **System Message**: Specialized for Telegram-sourced content analysis
- **Output Format**: JSON with `riskLevel`, `riskCategory`, `reasoning`, `indicators`

### Agent Bravo (Webhook Flow)
- **System Message**: Specialized for webhook-sourced content analysis
- **Output Format**: JSON with `riskLevel`, `riskCategory`, `reasoning`, `indicators`

### Risk Classification Output

The AI agents return JSON in this format:
```json
{
  "riskLevel": "High" | "Medium" | "Low",
  "riskCategory": "misinformation" | "sensitive_topic" | "reputation_threat" | "none",
  "reasoning": "Brief explanation of risk assessment",
  "indicators": ["indicator1", "indicator2", ...]
}
```

## Error Handling

The workflow includes basic error handling:
- Optional chaining replaced with explicit null checks
- Default values for missing data
- Fallback values for API failures

**Recommended Enhancements:**
- Add Error Trigger node for comprehensive error tracking
- Add retry logic for HTTP requests
- Add validation for search query input

## Customization

### Modify Search Sources
- Add additional HTTP Request nodes for other sources
- Update Merge Results node to include new sources
- Adjust Process nodes to normalize new data formats

### Adjust Risk Thresholds
- Modify If node condition in "Risk Categorization"
- Add additional risk levels (e.g., Critical, Low)
- Create additional Google Sheets for new categories

### Enhance AI Analysis
- Modify system messages in Agent Alpha/Bravo
- Add additional AI tools to agents
- Implement multi-step analysis with multiple AI calls

## Troubleshooting

### Common Issues

1. **Telegram messages not triggering**:
   - Verify bot token is correct
   - Check bot has access to channel/chat
   - Ensure workflow is activated

2. **Google Sheets not updating**:
   - Verify OAuth2 credentials
   - Check sheet names match exactly
   - Ensure sheet has write permissions

3. **AI analysis failing**:
   - Verify OpenAI API key is valid
   - Check API quota/limits
   - Review agent prompt formatting

4. **Webhook not responding**:
   - Verify webhook URL is correct
   - Check response mode configuration
   - Ensure Respond to Webhook node is connected

## Notes

- Reddit API: Uses public search endpoint (no authentication required)
- Google News: Uses RSS feed (no authentication required)
- Rate Limits: Be aware of API rate limits for production use
- Data Privacy: Ensure compliance with data protection regulations
- Cost Management: Monitor OpenAI API usage for cost control

## Support

For issues or questions:
1. Check n8n execution logs
2. Review node error messages
3. Validate credentials and permissions
4. Test individual nodes in isolation
