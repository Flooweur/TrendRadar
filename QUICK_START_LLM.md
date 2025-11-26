# Quick Start: Enable LLM Translation (5 minutes)

Transform your Chinese news notifications into friendly English summaries!

## Step 1: Get Your API Key (2 minutes)

1. Visit [Google AI Studio](https://makersuite.google.com/app/apikey)
2. Sign in with your Google account
3. Click **"Create API Key"**
4. Copy the API key (looks like: `AIzaSy...`)

> 💡 Google Gemini has a generous free tier!

## Step 2: Enable the Feature (3 minutes)

### Option A: Using config.yaml (Local/Docker)

Open `config/config.yaml` and update the LLM section:

```yaml
notification:
  # ... other settings ...
  
  llm:
    enabled: true  # Change from false to true
    api_url: "https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent"
    api_token: "AIzaSy..."  # Paste your API key here
```

### Option B: Using GitHub Actions

Add these three secrets to your repository:

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **"New repository secret"** and add:

| Name | Value |
|------|-------|
| `LLM_ENABLED` | `true` |
| `LLM_API_URL` | `https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent` |
| `LLM_API_TOKEN` | `AIzaSy...` (your API key) |

### Option C: Using Docker

Edit `docker/.env`:

```bash
LLM_ENABLED=true
LLM_API_URL=https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent
LLM_API_TOKEN=AIzaSy...
```

Then restart your Docker container:
```bash
docker restart trend-radar
```

## Step 3: Test It!

### GitHub Actions:
- Go to **Actions** tab → **"Hot News Crawler"** → **"Run workflow"**

### Local/Docker:
- Just wait for the next scheduled run, or run `python main.py` manually

## What to Expect

**Before:**
```
📊 热点词汇统计
🔥 [1/2] 特斯拉 马斯克 : 3 条
  1. [微博] 特斯拉降价促销活动开始
  2. [抖音] 马斯克宣布新产品计划
```

**After:**
```
📰 Hourly Update
🕐 2025-11-26 14:00:00

Hey! Quick update on what's trending 📱

🚗 Tesla & Elon Musk making moves:
Tesla just kicked off a price cut promotion (Weibo), 
and Elon announced plans for a new product (Douyin).

Looks like Tesla's trying to boost sales! 💰
```

## Troubleshooting

### Not working?

Check the logs for these messages:

✅ **Good signs:**
- `🤖 正在使用LLM翻译和总结新闻...`
- `✅ LLM翻译完成`
- `✅ 将使用LLM翻译后的内容发送通知`

❌ **Problems:**
- `⚠️ LLM功能已启用但未配置API URL或Token` → Check your config
- `❌ LLM API请求失败: 401` → Invalid API key
- `❌ LLM API请求失败: 429` → Rate limit exceeded (wait a bit)

### Still receiving Chinese?

The system automatically falls back to Chinese if:
- LLM API is unavailable
- API key is invalid
- Request times out (60 seconds)

This ensures you **always** receive notifications, even if the LLM service has issues.

## Cost Info

- **Free tier**: Up to 60 requests per minute
- **Each news update = 1 request**
- Running hourly = ~720 requests/month = **FREE** ✨

## Need Help?

See the full guide: [`LLM_SETUP_GUIDE.md`](./LLM_SETUP_GUIDE.md)

---

**That's it!** Enjoy your news in natural English! 🎉
