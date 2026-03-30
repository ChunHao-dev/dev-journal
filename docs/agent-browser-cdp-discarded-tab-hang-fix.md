# agent-browser CDP 連線 Bug 修復：Memory Saver 凍結 Tab 導致 Hang

> **版本資訊**
> - agent-browser: 0.23.0
> - 相關 Issue: [#1036](https://github.com/vercel-labs/agent-browser/issues/1036)
> - 相關 PR: [#1080](https://github.com/vercel-labs/agent-browser/pull/1080)
> - 更新日期: 2026-03-30

---

## TL;DR

- **問題**：用 `--cdp` 連到有 Memory Saver 凍結 tab 的 Chrome 時，agent-browser 永遠 hang 住
- **根因**：啟動時對 discarded tab 送 `Page.enable`，該 tab 沒有 renderer process，永遠不回應
- **修復**：對 `enable_domains()` 加 3 秒 timeout，timeout 後 fallback 開新 tab
- **影響範圍**：僅修改 `discover_and_attach_targets()` 一個函數
- **風險**：低，不改變正常連線行為，只在 timeout 時觸發 fallback

---

## 解決的問題

### 使用情境

用 `agent-browser --cdp 9222` 連到日常使用的 Chrome 瀏覽器，進行自動化操作。

**預期行為：**
- 連上 Chrome，可以操作頁面

**實際行為：**
- 指令永遠 hang 住，沒有任何輸出或錯誤訊息
- 必須 `Ctrl+C` 中斷

**觸發條件：**
- Chrome 開啟了 Memory Saver（記憶體節省模式）
- 有 tab 被自動凍結（discarded）
- 使用 `--cdp` 連線

### 影響範圍

幾乎所有用 `--cdp` 連到日常瀏覽器的使用者都會遇到。Tab 越多，被凍結的機率越高。

---

## 核心概念

### agent-browser 的架構

agent-browser 採用 CLI + Daemon 架構，透過 CDP（Chrome DevTools Protocol）控制瀏覽器：

```
┌──────────┐     IPC      ┌──────────┐    WebSocket    ┌─────────┐
│          │   (Unix      │          │     (CDP)       │         │
│   CLI    │───socket)───>│  Daemon  │───────────────->│ Chrome  │
│          │              │          │                 │         │
│ 短命的    │              │ 常駐背景  │                 │ 瀏覽器   │
│ process  │              │ process  │                 │         │
└──────────┘              └──────────┘                 └─────────┘

你打指令                   維持連線                     實際的瀏覽器
(get title)               管理 tab 狀態                (你看到的畫面)
```

### CDP 連線流程

當 daemon 連上 Chrome 時，會經過以下步驟：

```
1. WebSocket 連線
   Daemon ──ws://127.0.0.1:9222──> Chrome
                                     │
2. 發現所有 targets                    │
   Target.getTargets ────────────────>│
   <── 回傳 tab 列表 ─────────────────│
                                     │
3. Attach 每個 tab                    │
   Target.attachToTarget ───────────>│  (建立 session)
   <── 回傳 session_id ─────────────│
                                     │
4. Enable domains (啟用監聽)           │
   Page.enable ─────────────────────>│  ← 這步會 hang!
   Runtime.enable ──────────────────>│
   Network.enable ──────────────────>│
```

### 什麼是 Attach 和 Enable？

這是兩個不同的動作：

| | Attach | Enable |
|---|---|---|
| 做什麼 | 建立 CDP session 連線到 tab | 啟用特定 domain 的事件監聽 |
| 對 discarded tab | ✅ 成功（不需要 renderer） | ❌ hang（需要 renderer） |
| 拿到什麼 | session_id、title、url | 可以操作頁面、執行 JS、監聽網路 |

**Attach** 只是在 CDP 層建立一個 session，不需要 tab 的 renderer process。

**Enable** 是告訴 Chrome「我要開始監聽這個 tab 的事件」：
- `Page.enable` → 頁面載入、導航事件
- `Runtime.enable` → JS 執行環境（才能跑 `eval`）
- `Network.enable` → 網路請求事件

沒有 enable 的 tab，你只知道它存在（title、url），但不能對它做任何操作。

### 為什麼只 enable 一個 tab？

agent-browser 的設計是：**啟動時只 enable 第一個 tab（active tab），其他 tab 在 `tab switch` 時才 enable。**

這不是 CDP 的限制 — CDP 允許同時 enable 多個 tab。這是 agent-browser 的設計選擇，好處是：
- 啟動快（不用等所有 tab enable）
- 資源省（不用監聽所有 tab 的事件）

### 什麼是 Discarded Tab？

Chrome 的 Memory Saver 功能會自動凍結不活躍的 tab 來節省記憶體：

```
活躍的 tab                    Discarded tab
┌─────────────────┐          ┌─────────────────┐
│  Tab: GitHub    │          │  Tab: YouTube   │
│                 │          │                 │
│  [Renderer ✅]  │          │  [Renderer ❌]  │
│  - DOM 在記憶體  │          │  - 已被殺掉     │
│  - JS 在執行    │          │  - 記憶體已釋放  │
│  - 可以操作     │          │  - 只剩標題/URL  │
└─────────────────┘          └─────────────────┘
  Page.enable → 立即回應        Page.enable → 永遠不回應
```

使用者點擊 discarded tab 時，Chrome 會重新載入頁面（重啟 renderer）。

可以在 `chrome://discards` 查看每個 tab 的狀態，也可以手動 discard tab。

---

## Bug 分析

### 問題出在哪裡

在 `cli/src/native/browser.rs` 的 `discover_and_attach_targets()` 函數中：

```rust
// 1. Attach 所有 tab（沒問題）
for target in &page_targets {
    let attach_result = self.client
        .send_command_typed("Target.attachToTarget", ...)
        .await?;
    self.pages.push(...);
}

// 2. Enable 第一個 tab（問題在這裡！）
self.active_page_index = 0;
let session_id = self.pages[0].session_id.clone();
self.enable_domains(&session_id).await?;  // ← 如果 pages[0] 是 discarded → 永遠 hang
```

`pages[0]` 的順序由 `Target.getTargets` 決定，不是按活躍度排的。如果第一個 tab 剛好是 discarded 的，`Page.enable` 就永遠不會回應。

### 為什麼沒有 timeout？

CDP client 的 `send_command` 其實有 30 秒 timeout：

```rust
let response = match tokio::time::timeout(
    std::time::Duration::from_secs(30), rx
).await { ... };
```

但 `enable_domains` 連續送 3 個指令（`Page.enable`、`Runtime.enable`、`Network.enable`），第一個就卡住了，30 秒後才會 timeout 報錯。而且 daemon 在這段時間完全被 block，CLI 透過 IPC 送的指令也得不到回應。

### 重現步驟

```bash
# 1. 開 Chrome with remote debugging
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222

# 2. 開幾個 tab，到 chrome://discards 手動 discard 幾個

# 3. 連線 — 會 hang
agent-browser --cdp 9222 get title
# （永遠不回應，必須 Ctrl+C）
```

---

## 修復方式

### 策略

對 `enable_domains()` 加 3 秒 timeout。如果第一個 tab 沒回應（大概率是 discarded），fallback 開一個新的 `about:blank` tab。

### 程式碼變更

只改了 `cli/src/native/browser.rs` 的 `discover_and_attach_targets()` 函數：

```rust
// 修改前：直接 enable，沒有 timeout
self.active_page_index = 0;
let session_id = self.pages[0].session_id.clone();
self.enable_domains(&session_id).await?;

// 修改後：加 3 秒 timeout + fallback
self.active_page_index = 0;
let session_id = self.pages[0].session_id.clone();
match tokio::time::timeout(
    std::time::Duration::from_secs(3),
    self.enable_domains(&session_id),
).await {
    Ok(Ok(())) => {}  // 成功，正常繼續
    _ => {
        // Timeout — 開新 tab 作為 fallback
        let result = self.client.send_command_typed(
            "Target.createTarget",
            &CreateTargetParams { url: "about:blank".to_string() },
            None,
        ).await?;
        // ... attach + enable 新 tab
    }
}
```

### 修復後的行為

```
情境 1：所有 tab 都活躍
  pages[0] enable → 立即成功 → 正常使用
  （跟修改前完全一樣）

情境 2：pages[0] 是 discarded tab
  pages[0] enable → 3 秒 timeout → 開新 about:blank tab → enable 成功
  （修改前：永遠 hang）

情境 3：所有 tab 都 discarded
  pages[0] enable → 3 秒 timeout → 開新 about:blank tab → enable 成功
  （修改前：永遠 hang）
```

所有已存在的 tab 仍然會出現在 `tab list` 中（因為 attach 是成功的），只是 discarded 的 tab 沒有被 enable。使用者可以用 `tab switch` 切到其他 tab，那時候才會觸發 enable。

---

## 學到的東西

### CDP 的 Target 生命週期

```
Target 建立 → Attach（建立 session）→ Enable domains → 可操作
                  ↑                       ↑
              不需要 renderer          需要 renderer
              對 discarded tab 也行    對 discarded tab 會 hang
```

### `/json/list` vs `Target.getTargets`

| | `/json/list` (HTTP) | `Target.getTargets` (CDP) |
|---|---|---|
| 排序 | 最近活躍的在前面 | 不確定（可能按建立時間） |
| 資訊 | 基本（id, title, url） | 完整（含 attached 狀態等） |
| 用途 | 快速查看 tab 列表 | 程式化操作 |

未來可以考慮用 `/json/list` 的排序來決定先 enable 哪個 tab，讓連線更智慧。

### Timeout 的重要性

跟外部系統溝通時，永遠要有 timeout。即使 CDP client 有 30 秒 timeout，在特定場景下（discarded tab）這個 timeout 太長了，而且會 block 整個 daemon。在關鍵路徑上加更短的 timeout + fallback 是更好的做法。

---

## Troubleshooting

### 症狀：`agent-browser --cdp` 連線後永遠 hang

- **可能原因**：Chrome 有 discarded/frozen tab（Memory Saver）
- **確認方式**：到 `chrome://discards` 查看 tab 狀態
- **處理方式**：
  1. 升級到包含此修復的版本
  2. 或手動在 Chrome 中點擊 discarded tab 讓它重新載入，再連線

### 症狀：連線成功但 active tab 是 `about:blank`

- **原因**：第一個 tab 是 discarded 的，觸發了 fallback
- **處理方式**：用 `tab list` 查看所有 tab，`tab switch <n>` 切到想要的 tab

---

## 參考資源

- [Chrome DevTools Protocol - Target domain](https://chromedevtools.github.io/devtools-protocol/tot/Target/)
- [Chrome Memory Saver](https://support.google.com/chrome/answer/12929150)
- [Issue #1036: CDP connect hangs on browsers with discarded/frozen tabs](https://github.com/vercel-labs/agent-browser/issues/1036)
- [agent-browser GitHub](https://github.com/vercel-labs/agent-browser)
