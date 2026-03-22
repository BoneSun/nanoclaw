# Telegram Channel Triage and Known Issues

This document records known issues, debugging steps, and lessons learned from Telegram channel development.

---

## Issue: Messages Not Responding (SQL Filter Bug)

**Date:** 2026-03-22

### Symptoms

- Bot receives messages from Telegram
- Messages are stored in SQLite database
- But bot never responds
- Logs show `Telegram message stored` but never `Processing messages`

### Root Cause

The `getMessagesSince()` function in `src/db.ts` had a SQL filter that was too broad:

```sql
-- BEFORE (buggy)
WHERE chat_jid = ? AND timestamp > ?
  AND is_bot_message = 0 AND content NOT LIKE ?  -- ❌ Filtered user trigger messages!
```

The `content NOT LIKE '@Andy%'` was intended to filter out **bot responses** so they wouldn't be re-processed. But it ended up filtering out **all user trigger messages** that start with `@Andy` — which are exactly the messages that need to be processed!

### Evidence

```sql
-- Messages in database:
62|@Andy 测试|2026-03-22T05:45:41.000Z|0
61|@Andy 你好|2026-03-22T05:16:56.000Z|0

-- last_agent_timestamp: 2026-03-22T05:10:00.000Z
-- These messages SHOULD be processed (timestamp > last_agent_timestamp)
-- But getMessagesSince() returns empty because content NOT LIKE '@Andy%' filters them out
```

### Fix

Remove the `content NOT LIKE` filter from `getMessagesSince()`:

```sql
-- AFTER (fixed)
WHERE chat_jid = ? AND timestamp > ?
  AND is_bot_message = 0  -- ✅ Only filter messages already marked as bot responses
  AND content != '' AND content IS NOT NULL
```

The `is_bot_message` field is the correct way to identify bot responses. Adding a content-based filter was redundant and caused a logic error.

### Files Changed

- `src/db.ts`: Modified `getMessagesSince()` function (line 346-370)

### Lessons Learned

1. **Filter conditions must match field semantics**
   - `is_bot_message` is specifically designed to mark bot messages
   - Don't add redundant filters that can contradict the primary filter

2. **Test message retrieval functions independently**
   - `getNewMessages()` and `getMessagesSince()` have different use cases
   - Fixing one doesn't fix the other — always check all retrieval paths

3. **Log what you query**
   - Adding debug logs for SQL query results helps catch filtering bugs early

---

## Related Issues

### Telegram connect() Promise Never Resolves

**Symptom:** Bot connects but `await channel.connect()` hangs forever

**Root Cause:** Grammy's `bot.start()` starts polling but never resolves the Promise.

**Fix:** Wrap the start() call in a Promise that resolves in the `onStart` callback.

### Proxy Configuration for Telegram API

**Context:** Telegram API requires proxy in restricted network environments.

**Configuration:**

```bash
# Set in .env or environment
HTTP_PROXY=http://192.168.0.86:7890
HTTPS_PROXY=http://192.168.0.86:7890
```

**Implementation Details:**

1. **Dynamic Proxy Detection:**
   ```typescript
   const proxyUrl = process.env.HTTPS_PROXY || process.env.HTTP_PROXY;
   ```

2. **HttpsProxyAgent for HTTPS:**
   ```typescript
   const { HttpsProxyAgent } = await import('https-proxy-agent');
   agent = new HttpsProxyAgent(proxyUrl);
   ```

3. **Custom Fetch Function:**
   - Wraps Telegram's HTTP client with custom fetch
   - Passes proxy agent explicitly
   - Includes timeout (60s) for each request
   - Logs all API requests/responses

**Verification:**
```
[log] Telegram proxy check: "http://192.168.0.86:7890"
[log] Telegram using https-proxy-agent
[log] Telegram API request: "https://api.telegram.org/bot.../getUpdates..."
[log] Telegram API response: status 200
```

### Group Privacy Blocking Group Messages

**Symptom:** Bot only responds to @mentions in groups, not all messages

**Cause:** Telegram bot default is to only see @mentions and commands in groups.

**Fix:** In @BotFather, go to Bot Settings > Group Privacy > Turn off

---
