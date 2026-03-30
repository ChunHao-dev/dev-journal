# agent-browser CDP Connection Bug Fix: Memory Saver Frozen Tabs Cause Hang

> **Version Info**
> - agent-browser: 0.23.0
> - Related Issue: [#1036](https://github.com/vercel-labs/agent-browser/issues/1036)
> - Related PR: [#1080](https://github.com/vercel-labs/agent-browser/pull/1080)
> - Updated: 2026-03-30

---

## TL;DR

- **Problem**: `--cdp` connection hangs indefinitely when Chrome has Memory Saver frozen tabs
- **Root cause**: `Page.enable` sent to a discarded tab with no renderer process — never responds
- **Fix**: Add 3-second timeout to `enable_domains()`, fall back to a new tab on timeout
- **Scope**: Single function change in `discover_and_attach_targets()`
- **Risk**: Low — normal connection behavior unchanged, fallback only triggers on timeout

---

## The Problem

### Scenario

Connect to your daily Chrome browser via `agent-browser --cdp 9222` for automation.

**Expected:**
- Connects to Chrome, ready to operate

**Actual:**
- Command hangs forever — no output, no error message
- Must `Ctrl+C` to kill

**Trigger:**
- Chrome has Memory Saver enabled (on by default in modern Chrome)
- Some tabs have been automatically frozen (discarded)
- Using `--cdp` to connect

### Who This Affects

Almost everyone connecting to their daily browser via `--cdp`. The more tabs you have open, the higher the chance some are discarded.

---

## Core Concepts

### agent-browser Architecture

agent-browser uses a CLI + Daemon architecture, controlling the browser via CDP (Chrome DevTools Protocol):

```
┌──────────┐      IPC       ┌──────────┐    WebSocket     ┌─────────┐
│          │   (Unix        │          │     (CDP)        │         │
│   CLI    │───socket)────> │  Daemon  │────────────────> │ Chrome  │
│          │                │          │                  │         │
│ Short-   │                │ Long-    │                  │ The     │
│ lived    │                │ running  │                  │ browser │
│ process  │                │ process  │                  │         │
└──────────┘                └──────────┘                  └─────────┘

 You type                   Maintains                    The actual
 commands                   connection &                 browser you
 (get title)                tab state                    see on screen
```

### CDP Connection Flow

When the daemon connects to Chrome, it goes through these steps:

```
1. WebSocket connection
   Daemon ──ws://127.0.0.1:9222──> Chrome
                                     │
2. Discover all targets              │
   Target.getTargets ───────────────>│
   <── returns tab list ────────────│
                                     │
3. Attach to each tab                │
   Target.attachToTarget ──────────>│  (establishes session)
   <── returns session_id ─────────│
                                     │
4. Enable domains (start listening)  │
   Page.enable ────────────────────>│  ← THIS STEP HANGS!
   Runtime.enable ─────────────────>│
   Network.enable ─────────────────>│
```

### Attach vs Enable

These are two different operations:

| | Attach | Enable |
|---|---|---|
| What it does | Establishes a CDP session to a tab | Activates event listening for a domain |
| On discarded tabs | ✅ Works (no renderer needed) | ❌ Hangs (renderer required) |
| What you get | session_id, title, url | Ability to operate the page, run JS, monitor network |

**Attach** just creates a session at the CDP level — it doesn't need the tab's renderer process.

**Enable** tells Chrome "start sending me events for this tab":
- `Page.enable` → page load, navigation events
- `Runtime.enable` → JS execution context (needed for `eval`)
- `Network.enable` → network request events

Without enabling, you only know a tab exists (title, url) but can't interact with it.

### Why Only One Tab Is Enabled at a Time

agent-browser's design: **only enable the first tab (active tab) at startup; other tabs get enabled on `tab switch`.**

This is not a CDP limitation — CDP allows enabling multiple tabs simultaneously. It's a design choice:
- Faster startup (don't wait for all tabs to enable)
- Lower resource usage (don't listen to events from all tabs)

### What Is a Discarded Tab?

Chrome's Memory Saver feature automatically freezes inactive tabs to save memory:

```
Active tab                       Discarded tab
┌─────────────────┐              ┌─────────────────┐
│  Tab: GitHub    │              │  Tab: YouTube   │
│                 │              │                 │
│  [Renderer ✅]  │              │  [Renderer ❌]  │
│  - DOM in memory│              │  - Killed       │
│  - JS running   │              │  - Memory freed │
│  - Operable     │              │  - Only title/  │
│                 │              │    URL remain   │
└─────────────────┘              └─────────────────┘
  Page.enable → instant response   Page.enable → never responds
```

When a user clicks a discarded tab, Chrome reloads the page (restarts the renderer).

You can view tab states at `chrome://discards` and manually discard tabs for testing.

---

## Bug Analysis

### Where the Problem Is

In `cli/src/native/browser.rs`, the `discover_and_attach_targets()` function:

```rust
// 1. Attach all tabs (no problem)
for target in &page_targets {
    let attach_result = self.client
        .send_command_typed("Target.attachToTarget", ...)
        .await?;
    self.pages.push(...);
}

// 2. Enable the first tab (BUG IS HERE!)
self.active_page_index = 0;
let session_id = self.pages[0].session_id.clone();
self.enable_domains(&session_id).await?;  // ← if pages[0] is discarded → hangs forever
```

The order of `pages[0]` is determined by `Target.getTargets`, which is not sorted by activity. If the first tab happens to be discarded, `Page.enable` never responds.

### Why No Timeout?

The CDP client does have a 30-second timeout:

```rust
let response = match tokio::time::timeout(
    std::time::Duration::from_secs(30), rx
).await { ... };
```

But `enable_domains` sends 3 commands sequentially (`Page.enable`, `Runtime.enable`, `Network.enable`). The first one hangs, and the daemon is completely blocked for 30 seconds. During this time, CLI commands sent via IPC also get no response.

### Reproduction Steps

```bash
# 1. Launch Chrome with remote debugging
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222

# 2. Open several tabs, go to chrome://discards, Urgent Discard a few

# 3. Connect — will hang
agent-browser --cdp 9222 get title
# (never responds, must Ctrl+C)
```

---

## The Fix

### Strategy

Add a 3-second timeout to `enable_domains()`. If the first tab doesn't respond (likely discarded), fall back to creating a new `about:blank` tab.

### Code Change

Only `cli/src/native/browser.rs` — the `discover_and_attach_targets()` function:

```rust
// Before: direct enable, no timeout
self.active_page_index = 0;
let session_id = self.pages[0].session_id.clone();
self.enable_domains(&session_id).await?;

// After: 3-second timeout + fallback
self.active_page_index = 0;
let session_id = self.pages[0].session_id.clone();
match tokio::time::timeout(
    std::time::Duration::from_secs(3),
    self.enable_domains(&session_id),
).await {
    Ok(Ok(())) => {}  // Success, continue normally
    _ => {
        // Timeout — create a new tab as fallback
        let result = self.client.send_command_typed(
            "Target.createTarget",
            &CreateTargetParams { url: "about:blank".to_string() },
            None,
        ).await?;
        // ... attach + enable the new tab
    }
}
```

### Behavior After Fix

```
Scenario 1: All tabs are active
  pages[0] enable → instant success → normal operation
  (identical to before)

Scenario 2: pages[0] is a discarded tab
  pages[0] enable → 3s timeout → create new about:blank → enable succeeds
  (before: hangs forever)

Scenario 3: All tabs are discarded
  pages[0] enable → 3s timeout → create new about:blank → enable succeeds
  (before: hangs forever)
```

All existing tabs still appear in `tab list` (attach succeeded). Discarded tabs just aren't enabled. Users can `tab switch` to other tabs, which triggers enable at that point.

---

## Lessons Learned

### CDP Target Lifecycle

```
Target created → Attach (session) → Enable domains → Operable
                    ↑                     ↑
              No renderer needed    Renderer required
              Works on discarded    Hangs on discarded
```

### `/json/list` vs `Target.getTargets`

| | `/json/list` (HTTP) | `Target.getTargets` (CDP) |
|---|---|---|
| Sort order | Most recently active first | Unspecified (possibly creation order) |
| Info | Basic (id, title, url) | Full (includes attached state, etc.) |
| Use case | Quick tab listing | Programmatic operations |

A future improvement could use `/json/list` ordering to pick the most recently active tab for initial enable, making the connection smarter.

### Always Have a Timeout

When communicating with external systems, always have a timeout. Even though the CDP client has a 30-second timeout, it was too long for this specific scenario and blocked the entire daemon. Adding a shorter timeout + fallback on the critical path is a better approach.

---

## Troubleshooting

### Symptom: `agent-browser --cdp` hangs forever after connecting

- **Likely cause**: Chrome has discarded/frozen tabs (Memory Saver)
- **How to verify**: Go to `chrome://discards` and check tab states
- **Fix**:
  1. Upgrade to a version that includes this fix
  2. Or manually click discarded tabs in Chrome to reload them, then connect

### Symptom: Connection succeeds but active tab is `about:blank`

- **Cause**: The first tab was discarded, triggering the fallback
- **Fix**: Use `tab list` to see all tabs, then `tab switch <n>` to the desired tab

---

## References

- [Chrome DevTools Protocol - Target domain](https://chromedevtools.github.io/devtools-protocol/tot/Target/)
- [Chrome Memory Saver](https://support.google.com/chrome/answer/12929150)
- [Issue #1036: CDP connect hangs on browsers with discarded/frozen tabs](https://github.com/vercel-labs/agent-browser/issues/1036)
- [agent-browser GitHub](https://github.com/vercel-labs/agent-browser)
