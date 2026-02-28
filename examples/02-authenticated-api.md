# Authenticated API Calls via Browser Cookies

The killer feature of agent-browser: calling APIs that require authentication, using the cookies already in your browser. No API keys. No OAuth flows. No token management.

## The Pattern

1. Navigate to the correct subdomain (so cookies are attached)
2. Use `fetch()` via eval (browser attaches cookies automatically)
3. Parse and return the result

## Real-World Example: Fetching Watchlist Data

Suppose you use a financial platform at `trade.example.com` and need to pull your watchlist via their internal API. You're already logged in via Chrome.

### Step 1: Navigate to the correct subdomain

```bash
agent-browser open 'https://trade.example.com'
```

**This step is critical.** The browser only sends cookies for the current origin. If you try to `fetch()` from `google.com`, your `trade.example.com` cookies won't be attached.

### Step 2: Call the API

```bash
agent-browser eval --stdin <<'EOF'
(async () => {
  const resp = await fetch('https://trade.example.com/api/v1/watchlist', {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include'
  });

  if (!resp.ok) {
    return JSON.stringify({
      error: true,
      status: resp.status,
      message: resp.statusText
    });
  }

  const data = await resp.json();
  return JSON.stringify(data, null, 2);
})();
EOF
```

### Step 3: Use the data

The output is plain JSON that Claude Code can parse and act on:

```json
{
  "watchlist": [
    { "symbol": "600519", "name": "Kweichow Moutai", "price": 1580.00 },
    { "symbol": "000858", "name": "Wuliangye", "price": 142.50 }
  ]
}
```

## The CORS Gotcha

If you call an API on `api.example.com` while navigated to `www.example.com`, you may hit CORS errors even though both are "the same site."

**Solution:** Always navigate to the exact subdomain that hosts the API before calling `fetch()`.

```bash
# Wrong: navigated to www, calling api
agent-browser open 'https://www.example.com'
agent-browser eval "fetch('https://api.example.com/data')"  # CORS error

# Right: navigate to the API's subdomain first
agent-browser open 'https://api.example.com'
agent-browser eval "fetch('https://api.example.com/data')"  # Works
```

## Error Handling

### Rate Limiting (503 / 429)

Many internal APIs rate-limit aggressively. Build in retry logic:

```bash
agent-browser eval --stdin <<'EOF'
(async () => {
  const maxRetries = 3;
  for (let i = 0; i < maxRetries; i++) {
    const resp = await fetch('/api/v1/data');
    if (resp.status === 503 || resp.status === 429) {
      const wait = (i + 1) * 2000;
      await new Promise(r => setTimeout(r, wait));
      continue;
    }
    if (!resp.ok) {
      return JSON.stringify({ error: resp.status, message: resp.statusText });
    }
    return JSON.stringify(await resp.json());
  }
  return JSON.stringify({ error: 'rate_limited', message: 'Max retries exceeded' });
})();
EOF
```

### Auth Failures (401 / 403)

If you get a 401, your session has expired. The fix is manual -- log in again in Chrome, then retry.

```bash
# Check if we're still authenticated
agent-browser eval --stdin <<'EOF'
(async () => {
  const resp = await fetch('/api/v1/me');
  if (resp.status === 401) {
    return 'SESSION_EXPIRED: Please log in again in Chrome';
  }
  return JSON.stringify(await resp.json());
})();
EOF
```

## When to Use This vs. Regular API Keys

| Scenario | Use agent-browser | Use API keys |
|---|---|---|
| Internal tools with SSO | Yes | No keys available |
| Platform has no public API | Yes | N/A |
| Quick one-off data pull | Yes | Overkill |
| Production automation | No | Yes |
| CI/CD pipelines | No | Yes |

## Next Steps

- [Build a Claude Code skill](03-claude-code-skill.md) -- wrap this pattern into a reusable command
- [Gotchas](../docs/gotchas.md) -- all the edge cases from real usage
