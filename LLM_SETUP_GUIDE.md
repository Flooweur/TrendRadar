# LLM Translation and Summarization Setup Guide

This feature allows the news aggregator to automatically translate Chinese news into English and present it in a friendly, conversational format using a Large Language Model (LLM).

## Features

- 🌐 **Automatic Translation**: Translates Chinese news headlines into English
- 📝 **Smart Summarization**: Summarizes and groups related news topics
- 📊 **Factual Reporting**: Presents news objectively and professionally
- 🔄 **Seamless Integration**: Works with all existing notification platforms

## Configuration

### 1. In `config/config.yaml`

Add or update the LLM configuration section:

```yaml
notification:
  # ... other notification settings ...
  
  # LLM翻译和摘要配置
  llm:
    enabled: true  # Set to true to enable LLM translation
    api_url: "https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent"
```

**Note**: The API key is NOT stored in the config file for security. It comes from the `GEMINI_API_KEY` environment variable or GitHub secret.

### 2. For GitHub Actions (Recommended)

Add these secrets to your repository (`Settings` → `Secrets and variables` → `Actions`):

- `LLM_ENABLED`: Set to `true`
- `LLM_API_URL`: Your LLM API endpoint (e.g., Google Gemini API URL)
- `GEMINI_API_KEY`: Your Google Gemini API key

### 3. For Docker Deployment

Update the `docker/.env` file:

```bash
# LLM Configuration
LLM_ENABLED=true
LLM_API_URL=https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent
GEMINI_API_KEY=your_api_key_here
```

### 4. For Local Deployment

Set the environment variable before running:

```bash
export GEMINI_API_KEY="your_api_key_here"
python main.py
```

## Supported APIs

The current implementation uses the **Google Gemini API** format. The API request structure is:

```json
{
  "contents": [
    {
      "parts": [
        {
          "text": "prompt here"
        }
      ]
    }
  ]
}
```

### Getting a Google Gemini API Key

1. Go to [Google AI Studio](https://makersuite.google.com/app/apikey)
2. Sign in with your Google account
3. Click "Create API Key"
4. Copy the API key and use it in your configuration

**Note**: The API format follows the example provided, which is compatible with Google Gemini. If you want to use a different LLM provider, you may need to modify the `translate_and_summarize_with_llm()` function in `main.py` to match their API format.

## How It Works

1. **Data Collection**: The system collects Chinese news as usual
2. **LLM Processing**: If LLM is enabled, it sends the news to the LLM API with instructions to:
   - Translate Chinese to English accurately
   - Summarize the main topics and key developments
   - Stay factual and objective
   - Present in a professional, readable format
3. **Notification**: The translated/summarized English text is sent to all configured notification platforms

## Example Output

Instead of receiving Chinese news like:
```
🔥 [1/3] AI ChatGPT : 2 条
1. [百度热搜] ChatGPT-5正式发布
2. [今日头条] AI芯片概念股暴涨
```

You'll receive:
```
📰 Daily Summary
🕐 2025-11-26 14:30:00

Summary of 2 news items:

AI & ChatGPT (2 items):
- ChatGPT-5 has been officially released (Baidu Hot Search)
- AI chip concept stocks are surging (Toutiao)
```

## Troubleshooting

### LLM translation not working

Check the logs for error messages:
- `⚠️ LLM功能已启用但未配置API URL或Token` - Check your configuration
- `❌ LLM API请求失败` - Verify your API key is valid
- `❌ LLM API请求超时` - The API took too long to respond (60s timeout)

### Fallback behavior

If LLM translation fails for any reason, the system will automatically fall back to sending the original Chinese news format. This ensures you always receive notifications even if the LLM service is unavailable.

## Customization

To modify the LLM prompt or output style, edit the `translate_and_summarize_with_llm()` function in `main.py`. You can change:

- The `system_instruction` to adjust the AI's behavior
- The `user_prompt` to change what information is sent
- The response parsing logic if using a different API provider

## Cost Considerations

LLM API calls may incur costs depending on your provider:
- Google Gemini has a free tier with usage limits
- Each news update makes one API call
- Monitor your API usage through your provider's dashboard

## Privacy Note

When LLM translation is enabled, your news data is sent to the LLM API provider for processing. Make sure you're comfortable with this before enabling the feature.
