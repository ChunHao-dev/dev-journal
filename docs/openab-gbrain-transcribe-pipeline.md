# openAB + Tailscale + 本地轉錄 + gbrain：零 API 費的影片知識存入流程

> 透過開源專案 openAB 學習新技術：https://github.com/openabdev/openab

> **版本資訊**
> - openAB: 0.8.2
> - mlx_whisper: whisper-large-v3-turbo
> - Tailscale / K3s / Bun
> - 更新日期: 2026-05-10

---

想讓 AI agent 把 YouTube 影片的內容存進知識庫，但不想花錢呼叫轉錄 API。所以我在 openAB 上做了一套流程：透過 Tailscale 連到本地 Mac，用 mlx_whisper 本地轉錄，逐字稿回傳後存入 gbrain，gbrain 再自動把內容中提到的人、公司建成獨立頁面並串連。

重點是 openAB 接了 Discord，所以我只要在手機上貼個 YouTube 連結，只要 MacBook 有連上線，整套轉錄 → 存入就會自動跑完。人不用在電腦前。

---

## TL;DR

- openAB pod 透過 Tailscale 建立的私有網路 SSH 到本地 Mac
- Mac 上用 yt-dlp 下載影片、mlx_whisper 本地轉錄，零 API 費用
- 逐字稿回傳 openAB，使用者選擇存完整逐字稿或摘要
- 存入 gbrain 時，AI 自動識別內容中的人物/公司，建立獨立頁面
- 頁面之間用 `[[slug]]` 自動建立雙向連結，之後搜人名就能找到所有相關內容

---

## 解決的問題

| 問題 | 解法 |
|------|------|
| 轉錄 API 要錢（Whisper API、AssemblyAI 等） | 本地 Mac 跑 mlx_whisper，免費 |
| openAB pod 在雲端，Mac 在本地，網路不通 | Tailscale 建立私有網路 |
| 逐字稿存下來後是孤立的文字檔 | gbrain auto-link 自動建立關聯 |
| 之後想找「某人說過什麼」很難 | gbrain 自動建人物頁面，搜人名就出來 |

---

## 架構

```
┌──────────────┐         Tailscale 私有網路         ┌──────────────────┐
│  openAB pod  │ ──────── SSH (port 22) ──────────→ │  Mac (本地)       │
│  (雲端 K3s)  │                                    │                  │
│              │  1. ssh macbook "yt-dlp <URL>"         │  yt-dlp 下載     │
│              │  2. ssh macbook "mlx_whisper ..."      │  mlx_whisper 轉錄│
│              │  3. ssh macbook "cat transcript.json"  │  回傳逐字稿      │
│              │ ←──────── 逐字稿 JSON ──────────── │                  │
└──────┬───────┘                                    └──────────────────┘
       │
       │ gbrain-cli put
       ▼
┌──────────────┐
│ gbrain:3131  │
│ (同一台 K3s) │
│              │
│ 存入主頁面    │
│ ↓ 偵測到人名  │
│ → 建 people/ │
│ ↓ 偵測到公司  │
│ → 建 companies/ │
│ ↓ [[slug]]   │
│ → 雙向連結    │
└──────────────┘
```

---

## 完整流程

### Step 1：使用者貼 YouTube 連結

在 Discord 跟 openAB 說「存到 gbrain」並貼上 YouTube URL。

### Step 2：openAB SSH 到 Mac 下載影片

```bash
ssh macbook "yt-dlp -x --audio-format wav -o '/tmp/yt-dlp_download/%(title)s.%(ext)s' '<URL>'"
```

Tailscale 讓雲端的 pod 可以直接 SSH 到本地 Mac，不需要 port forwarding 或公網 IP。

### Step 3：本地轉錄

```bash
ssh macbook "<MLX_WHISPER_PYTHON> -c \"
import mlx_whisper, json, os
os.makedirs('/tmp/whisper_out', exist_ok=True)
result = mlx_whisper.transcribe(
    '<音檔路徑>',
    path_or_hf_repo='mlx-community/whisper-large-v3-turbo',
    condition_on_previous_text=False,
    word_timestamps=True
)
with open('/tmp/whisper_out/transcript.json', 'w') as f:
    json.dump(result, f, ensure_ascii=False, indent=2)
\""
```

- `mlx_whisper` 跑在 Apple Silicon 上，用 MLX 框架加速
- `condition_on_previous_text=False` 防止幻覺
- 語言自動偵測，中英文都能處理

### Step 4：逐字稿回傳 openAB

```bash
ssh macbook "cat /tmp/whisper_out/transcript.json"
```

Pod 拿到完整的逐字稿 JSON（含時間戳、分段）。

### Step 5：問使用者要存什麼

AI 問：「要存完整逐字稿還是只存摘要？」

- 完整逐字稿 → 原文放在「逐字稿」區塊，摘要放上方
- 只存摘要 → 重點整理 + 關鍵引述

### Step 6：存入 gbrain + auto-link

```bash
# 存入主頁面
gbrain-cli put concepts/video-title "影片標題" "<整理好的內容，包含 [[people/speaker-name]]>"

# gbrain 自動：
# - 偵測 [[people/speaker-name]] → 建立連結
# - AI 額外建立講者的人物頁面
gbrain-cli put people/speaker-name "Speaker Name" "講者資訊..."

# 加時間軸
gbrain-cli timeline-add people/speaker-name 2026-05-10 "在 <影片標題> 中談到 ..."
```

### 結果

之後搜尋講者名字：

```bash
gbrain-cli search "speaker name"
# → 出現 people/speaker-name
gbrain-cli backlinks people/speaker-name
# → 出現所有提到這個人的影片、筆記、文章
```

---

## Tailscale 的角色

openAB pod 跑在雲端（騰訊雲 K3s），Mac 在家裡。兩邊都裝 Tailscale 後：

- 自動分配私有 IP（100.x.x.x）
- Pod 可以直接 `ssh macbook` 連到 Mac
- 不需要設定 port forwarding、動態 DNS、或 VPN server
- 連線加密，走 WireGuard

```
雲端 pod (100.x.x.1) ←── Tailscale 加密隧道 ──→ Mac (100.x.x.2)
```

---

## gbrain auto-link 怎麼運作

寫入頁面時，gbrain 引擎自動做：

1. **Chunking** — 把內容切成搜尋用的區塊
2. **tsvector 索引** — 建立全文搜尋索引
3. **Auto-link** — 掃描 `[[slug]]` 語法，建立雙向連結
4. **Backlink boosting** — 被越多頁面引用的，搜尋排名越高

所以當你在影片筆記裡寫 `[[people/garry-tan]]`，gbrain 會：
- 從這篇筆記建立一條連結指向 `people/garry-tan`
- 之後查 `people/garry-tan` 的 backlinks，這篇筆記就會出現
- 搜尋時，被多篇引用的頁面分數更高

不需要手動整理，寫進去就自動關聯。

---

## 安全注意事項

- Tailscale SSH 連線是加密的，但 pod 裡存有 SSH key
- gbrain HTTP API 目前沒有認證（只有 K8s 內部可存取）
- Mac 需要開機才能轉錄，關機時會記待辦之後重試
- 建議：SSH key 限制只能執行特定指令（`command=` in authorized_keys）

---

## 總結

整條路不花一毛 API 費用：

| 環節 | 工具 | 費用 |
|------|------|------|
| 下載影片 | yt-dlp | 免費 |
| 轉錄 | mlx_whisper（本地） | 免費 |
| 網路連接 | Tailscale | 免費（個人方案） |
| 知識庫 | gbrain PGLite | 免費 |
| AI 處理 | openAB pod 裡的 LLM | 已有 |

gbrain 的 auto-link 讓知識不只是被存起來，而是自動串連。存越多，搜尋越準，因為 backlink boosting 會讓重要頁面浮上來。

---

## 參考資源

- gbrain：https://github.com/garrytan/gbrain
- openAB：https://github.com/openabdev/openab
- Tailscale：https://tailscale.com
- mlx_whisper：https://github.com/ml-explore/mlx-examples/tree/main/whisper
- yt-dlp：https://github.com/yt-dlp/yt-dlp
