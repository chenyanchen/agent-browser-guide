# Gotchas from Real Usage

Hard-won lessons from using agent-browser in production workflows.

## 1. CORS Subdomain Issue

**The problem:** You navigate to `www.example.com` but call `fetch('https://api.example.com/data')`. The browser blocks it due to CORS -- even though both are "example.com."

**Why it happens:** CORS is enforced per-origin. `www.example.com` and `api.example.com` are different origins. When you run `fetch()` via eval, it executes in the context of the current page's origin.

**Detailed example:**

```bash
# You're logged into trade.example.com in Chrome.
# You navigate to the landing page:
agent-browser navigate 'https://www.example.com'

# Then try to call the trade API:
agent-browser eval "fetch('https://trade.example.com/api/portfolio')"
# ERROR: CORS policy blocks cross-origin request

# Fix: navigate to the API's subdomain first
agent-browser navigate 'https://trade.example.com'
agent-browser eval "fetch('/api/portfolio', { credentials: 'include' })"
# Works -- same origin, cookies attached
```

**Rule:** Always navigate to the exact subdomain that hosts the API before calling `fetch()`.

## 2. Shell Escaping

**The problem:** JavaScript strings contain quotes, backticks, and `${}` template literals that clash with shell quoting.

**The fix:** Always use heredocs with a quoted delimiter:

```bash
# WRONG: shell interprets $, backticks, and quotes
agent-browser eval "const x = `hello ${name}`"

# WRONG: single quotes break on apostrophes in JS
agent-browser eval 'it's broken'

# RIGHT: heredoc with quoted delimiter prevents all interpolation
agent-browser eval "$(cat <<'JSEOF'
const items = document.querySelectorAll('.item');
const result = Array.from(items).map(el => {
  return `${el.dataset.id}: ${el.textContent.trim()}`;
});
JSON.stringify(result);
JSEOF
)"
```

The `'JSEOF'` (with quotes around the delimiter) prevents the shell from interpreting `${}`, backticks, or any special characters inside the heredoc.

## 3. Rate Limiting Patterns

Many web APIs rate-limit aggressively, especially internal ones not designed for programmatic access.

**Symptoms:**
- HTTP 429 (Too Many Requests)
- HTTP 503 (Service Unavailable)
- Empty responses or HTML error pages instead of JSON

**Defense:**

```javascript
async function fetchWithRetry(url, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    const resp = await fetch(url, { credentials: 'include' });
    if (resp.status === 429 || resp.status === 503) {
      await new Promise(r => setTimeout(r, (i + 1) * 2000));
      continue;
    }
    return resp;
  }
  throw new Error('Rate limited after ' + maxRetries + ' retries');
}
```

**Also:** Add delays between batch operations. If you're updating 50 watchlist items, don't fire 50 requests simultaneously.

## 4. Chrome Restart Kills CDP

On Chrome 145+ (the `chrome://inspect` approach), CDP is enabled per-session. When Chrome restarts (update, crash, manual quit), CDP is disabled.

**Impact:** Your agent-browser commands silently fail with connection errors.

**Fix:** After any Chrome restart, re-visit `chrome://inspect/#remote-debugging` before running agent-browser commands.

**In skills:** Always start with a precondition check:

```bash
if ! agent-browser list-pages > /dev/null 2>&1; then
  echo "CDP unavailable. Re-enable at chrome://inspect/#remote-debugging"
  exit 1
fi
```

## 5. Chrome 136+ user-data-dir Requirement

If you use the older `--remote-debugging-port=9222` approach (instead of `chrome://inspect`):

- Chrome 136+ **silently ignores** `--remote-debugging-port` unless you also pass `--user-data-dir`
- No error, no warning -- Chrome just starts normally without CDP
- This breaks setups that worked for years

```bash
# Broken on Chrome 136+:
chrome --remote-debugging-port=9222

# Works:
chrome --remote-debugging-port=9222 --user-data-dir="$HOME/.chrome-cdp"
```

**Recommendation:** Use Chrome 145+ with `chrome://inspect` instead. It avoids this issue entirely.

## 6. Security: CDP = Full Browser Access

CDP gives agent-browser **complete control** over your browser:

- Read any page content (including banking, email, passwords)
- Execute arbitrary JavaScript
- Access all cookies and local storage
- Navigate anywhere
- Take screenshots

**Mitigations:**
- Only enable CDP when you need it
- Review what skills are doing before granting `Bash(agent-browser:*)` permission
- Don't run untrusted skills with agent-browser access
- Remember: anyone on localhost can connect to CDP when it's enabled

## 7. Tab Contention: Agent vs Human

**The problem:** You and the agent share the same Chrome. If the agent is working in one tab and you switch to another, does the agent get confused?

**The answer: No, as long as you don't close the agent's tab.**

We tested this thoroughly: once agent-browser connects to a tab, it **locks onto that tab** even if you switch to other tabs or windows. Over 100 seconds of continuous monitoring while actively switching tabs, agent-browser consistently returned the same URL and title.

**When it does break:** If you run `agent-browser close` (which disconnects), the next `--auto-connect` will grab **whichever tab is currently active** — which may not be the one you want.

**Two solutions:**

**Option A: Don't touch the agent's tab** (Recommended)

Simply leave the agent's tab alone. Use other tabs normally. The agent maintains its connection to its tab regardless of what you do in other tabs.

```
Your Chrome window:
  [Tab 1: Agent's tab — don't close this]  [Tab 2: Your browsing]  [Tab 3: ...]
                ↑                                    ↑
    agent-browser works here              You browse freely here
```

Rules:
- Don't close the agent's tab
- Don't navigate the agent's tab manually
- Avoid `agent-browser close` mid-workflow
- Everything else is fine — new tabs, other windows, no conflict

**Option B: Dedicated Chrome Profile** (For heavy usage)

Launch a separate Chrome instance on a fixed CDP port:

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.chrome-agent" \
  --no-first-run
```

Then configure agent-browser to use this port:

```json
// ~/.agent-browser/config.json
{ "cdp": 9222 }
```

Trade-offs:
- Complete isolation — you can't accidentally interfere
- But you need to **re-login** to all sites in the new profile
- Accounts with single-session limits will log you out of your main Chrome

**Our recommendation:** Start with Option A. It works for 95% of use cases. Only switch to Option B if you're running long, unattended automations where you can't guarantee you won't interact with the agent's tab.

## 8. agent-browser Daemon Lifecycle

agent-browser runs a background daemon to maintain the CDP connection.

**Key insight from testing:** The daemon holds onto its connected tab. This is what makes [tab contention](#7-tab-contention-agent-vs-human) a non-issue — but it also means stale daemons can cause problems.

**Best practices:**
- Run `agent-browser close` when you're done with browser automation
- If commands behave oddly, close and reconnect
- In skills, don't assume daemon state from a previous session

```bash
# Restart if something feels wrong
agent-browser close
agent-browser --auto-connect get url  # reconnects to active tab
```

## 10. Ref Invalidation After Navigation

After calling `agent-browser navigate`, any DOM references from previous `eval` calls are invalid.

**Problem:**

```bash
# Get a reference to an element
agent-browser eval 'window.__myEl = document.querySelector("#login")'

# Navigate away
agent-browser navigate 'https://other-page.com'

# This reference is now garbage
agent-browser eval 'window.__myEl.click()'  # ERROR: stale reference
```

**Rule:** After any navigation, re-query the DOM. Don't cache element references across navigations.

## 11. Large eval Output

agent-browser returns eval results as strings. If your JavaScript returns a huge object (e.g., the entire DOM), it can:

- Blow up Claude Code's context window
- Cause timeouts
- Return truncated results

**Fix:** Filter and summarize server-side:

```javascript
// WRONG: return everything
JSON.stringify(await resp.json());

// RIGHT: return only what you need
const data = await resp.json();
JSON.stringify(data.items.map(i => ({ id: i.id, name: i.name })));
```

## Next Steps

- [Setup guide](setup.md) -- make sure your installation is solid
- [Skill vs MCP comparison](skill-vs-mcp.md) -- architectural decisions
