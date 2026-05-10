# openAB + Tailscale + Local Transcription + gbrain: Zero-API-Cost Video Knowledge Pipeline

> Learning new tech through the open-source project openAB: https://github.com/openabdev/openab

> **Version Info**
> - openAB: 0.8.2
> - mlx_whisper: whisper-large-v3-turbo
> - Tailscale / K3s / Bun
> - Updated: 2026-05-10

---

I wanted my AI agent to store YouTube video content into a knowledge base without paying for transcription APIs. So I built a pipeline on openAB: it connects to my local Mac via Tailscale, transcribes locally with mlx_whisper, sends the transcript back, stores it in gbrain, and gbrain automatically creates separate pages for any people or companies mentioned in the content and links them together.

The key point: openAB is connected to Discord, so I just paste a YouTube link on my phone, and as long as my MacBook is online, the entire transcription → storage pipeline runs automatically. No need to be at the computer.

---

## TL;DR

- openAB pod SSHs into local Mac via Tailscale private network
- Mac downloads with yt-dlp, transcribes with mlx_whisper locally — zero API cost
- Transcript sent back to openAB, user chooses full transcript or summary
- When stored in gbrain, AI auto-detects people/companies and creates separate pages
- Pages are linked bidirectionally via `[[slug]]` — search a person's name later and all related content shows up

---

## Problems Solved

| Problem | Solution |
|---------|----------|
| Transcription APIs cost money (Whisper API, AssemblyAI, etc.) | Run mlx_whisper locally on Mac, free |
| openAB pod is in the cloud, Mac is local, no network path | Tailscale creates a private network |
| Saved transcripts are isolated text files | gbrain auto-link creates connections |
| Hard to find "what did someone say" later | gbrain auto-creates people pages, search by name |

---

## Architecture

```
┌──────────────┐        Tailscale Private Network       ┌──────────────────┐
│  openAB pod  │ ──────── SSH (port 22) ──────────────→ │  Mac (local)      │
│  (cloud K3s) │                                        │                   │
│              │  1. ssh macbook "yt-dlp <URL>"         │  yt-dlp download  │
│              │  2. ssh macbook "mlx_whisper ..."      │  mlx_whisper      │
│              │  3. ssh macbook "cat transcript.json"  │  return transcript│
│              │ ←──────── transcript JSON ──────────── │                   │
└──────┬───────┘                                        └───────────────────┘
       │
       │ gbrain-cli put
       ▼
┌──────────────┐
│ gbrain:3131  │
│ (same K3s)   │
│              │
│ store page   │
│ ↓ detect name│
│ → people/    │
│ ↓ detect co. │
│ → companies/ │
│ ↓ [[slug]]   │
│ → bidi links │
└──────────────┘
```

---

## Full Pipeline

### Step 1: User Pastes YouTube Link

In Discord, tell openAB "save to gbrain" and paste the YouTube URL.

### Step 2: openAB SSHs into Mac to Download

```bash
ssh macbook "yt-dlp -x --audio-format wav -o '/tmp/yt-dlp_download/%(title)s.%(ext)s' '<URL>'"
```

Tailscale lets the cloud pod SSH directly into the local Mac — no port forwarding or public IP needed.

### Step 3: Local Transcription

```bash
ssh macbook "<MLX_WHISPER_PYTHON> -c \"
import mlx_whisper, json, os
os.makedirs('/tmp/whisper_out', exist_ok=True)
result = mlx_whisper.transcribe(
    '<audio_path>',
    path_or_hf_repo='mlx-community/whisper-large-v3-turbo',
    condition_on_previous_text=False,
    word_timestamps=True
)
with open('/tmp/whisper_out/transcript.json', 'w') as f:
    json.dump(result, f, ensure_ascii=False, indent=2)
\""
```

- `mlx_whisper` runs on Apple Silicon, accelerated by the MLX framework
- `condition_on_previous_text=False` prevents hallucination
- Language auto-detected — handles both Chinese and English

### Step 4: Transcript Sent Back to openAB

```bash
ssh macbook "cat /tmp/whisper_out/transcript.json"
```

Pod receives the full transcript JSON (with timestamps and segments).

### Step 5: Ask User What to Store

AI asks: "Save full transcript or summary only?"

- Full transcript → raw text in a "Transcript" section, summary above
- Summary only → key points + notable quotes

### Step 6: Store in gbrain + Auto-link

```bash
# Store main page
gbrain-cli put concepts/video-title "Video Title" "<formatted content with [[people/speaker-name]]>"

# gbrain automatically:
# - Detects [[people/speaker-name]] → creates link
# - AI creates speaker's people page
gbrain-cli put people/speaker-name "Speaker Name" "Speaker info..."

# Add timeline
gbrain-cli timeline-add people/speaker-name 2026-05-10 "Discussed ... in <Video Title>"
```

### Result

Search for the speaker later:

```bash
gbrain-cli search "speaker name"
# → shows people/speaker-name
gbrain-cli backlinks people/speaker-name
# → shows all videos, notes, articles mentioning this person
```

---

## Tailscale's Role

openAB pod runs in the cloud (K3s), Mac is at home. With Tailscale on both:

- Auto-assigned private IPs (100.x.x.x)
- Pod can directly `ssh macbook` to reach the Mac
- No port forwarding, dynamic DNS, or VPN server needed
- Encrypted connection via WireGuard

```
Cloud pod (100.x.x.1) ←── Tailscale encrypted tunnel ──→ Mac (100.x.x.2)
```

---

## How gbrain Auto-link Works

When a page is written, the gbrain engine automatically:

1. **Chunking** — splits content into searchable blocks
2. **tsvector index** — builds full-text search index
3. **Auto-link** — scans for `[[slug]]` syntax, creates bidirectional links
4. **Backlink boosting** — pages referenced by more pages rank higher in search

So when you write `[[people/garry-tan]]` in a video note, gbrain will:
- Create a link from this note to `people/garry-tan`
- When you check backlinks on `people/garry-tan`, this note shows up
- In search results, heavily-referenced pages score higher

No manual organizing — write it in and it's automatically connected.

---

## Security Notes

- Tailscale SSH is encrypted, but the pod stores an SSH key
- gbrain HTTP API has no auth currently (only accessible within K8s)
- Mac must be online for transcription; if offline, a to-do is logged for retry
- Recommendation: restrict SSH key to specific commands (`command=` in authorized_keys)

---

## Summary

The entire pipeline costs zero in API fees:

| Component | Tool | Cost |
|-----------|------|------|
| Download video | yt-dlp | Free |
| Transcription | mlx_whisper (local) | Free |
| Network | Tailscale | Free (personal plan) |
| Knowledge base | gbrain PGLite | Free |
| AI processing | LLM in openAB pod | Already available |

gbrain's auto-link means knowledge isn't just stored — it's automatically connected. The more you store, the better search gets, because backlink boosting surfaces important pages.

---

## References

- gbrain: https://github.com/garrytan/gbrain
- openAB: https://github.com/openabdev/openab
- Tailscale: https://tailscale.com
- mlx_whisper: https://github.com/ml-explore/mlx-examples/tree/main/whisper
- yt-dlp: https://github.com/yt-dlp/yt-dlp
