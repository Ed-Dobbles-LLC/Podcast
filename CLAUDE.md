# Intelligence Briefings — Project Context for Claude

## What This Is
An AI-powered executive podcast platform hosted on Railway. It generates 5-10 minute audio briefings on topics relevant to a senior analytics/AI executive (Ed Dobbles, CAO at Overproof). Two AI hosts — Alex ("The Empiricist") and Morgan ("The Strategist") — debate and analyze topics in a structured dialogue format with distinct rhetorical identities. Episodes are published as an RSS feed consumable by Apple Podcasts or any podcast app.

**Production URL:** https://intelligence-briefings-production.up.railway.app
**Local project path:** `C:\Users\eddob\Claude Projects\podcast-console\`
**Railway project ID:** 8d9b2720-162c-45a4-a209-1af24d88583e

---

## Stack
- **Backend:** Python / Flask, deployed on Railway with gunicorn
- **TTS:** ElevenLabs (eleven_turbo_v2_5, mp3_44100_128)
- **Script generation:** Anthropic Claude API — **dual-model architecture:**
  - `SCRIPT_MODEL` = `claude-opus-4-20250514` — used for script generation, series bibles (premium quality)
  - `EDITORIAL_MODEL` = `claude-sonnet-4-20250514` — used for topic generation, chat-to-topic, suggestions, episode summaries (cost-effective)
- **Data persistence:** Railway volume mounted at `~/Intelligence-Briefings/`
- **Frontend:** Single-page HTML/CSS/JS in `templates/index.html`
- **Process manager:** `Procfile` with gunicorn gthread workers, 300s timeout

## Key Files
```
podcast-console/
├── app.py                          # Full backend — all logic lives here
├── templates/index.html            # Single-page frontend
├── Procfile                        # gunicorn --workers 2 --worker-class gthread --threads 4 --timeout 300
├── requirements.txt
└── CLAUDE.md                       # This file
```

## Data Directory (Railway Volume)
```
~/Intelligence-Briefings/
├── episodes/                       # MP3 files + per-episode segment dirs
├── episodes.json                   # Episode metadata list
├── topics_cache.json               # Today's topics (refreshes daily)
├── jobs.json                       # Async job queue state
├── series.json                     # Series metadata
├── production_log.json             # Production count log
└── feed.xml                        # RSS feed (auto-rebuilt after each episode)
```

---

## Architecture: Async Job Queue

Railway has a 60-second HTTP proxy timeout. All generation tasks (2-4 minutes) run in background threads.

**Flow:**
1. Client POSTs to `/api/generate` or `/api/chat` → returns `{job_id, status: "queued"}` in <1 second
2. Background thread runs full pipeline, updates job status/progress in `jobs.json`
3. Client polls `GET /api/job/<job_id>` every 3 seconds
4. When `status === "done"`, result contains full episode object
5. UI renders audio player

**Job statuses:** `queued → running → done | error`

---

## Episode Generation Pipeline

1. `generate_grounded_script(topic, depth, production_brief, series_bible, prior_episode_summaries)` → calls Claude Opus API (with web search fallback) → returns JSON array of `{host, text}` segments. When `series_bible` is provided, injects full series context (assigned research, narrative threads, prior episode summaries, arc instructions) into the prompt.
2. `generate_episode_audio(client, script, ep_dir, voice_a_id, voice_b_id)` → calls ElevenLabs per segment → concatenates MP3s
3. Episode saved to `episodes.json`, RSS feed rebuilt automatically

**Segment targets (strictly enforced in prompt):**
- executive: minimum 10 segments
- standard: minimum 16 segments
- deep: minimum 24 segments
- Each segment: 3-5 substantial sentences

**Web search:** Enabled with graceful fallback. Scripts attempt `use_web_search=True` first; if it fails (permissions, quota), falls back to Claude's training knowledge silently. Requires `anthropic-beta: web-search-2025-03-05` header (handled automatically in `call_anthropic`).

**Script structure (7 sections):**
1. COLD OPEN — Drop into tension immediately
2. GROUND IT — Named examples with natural source attribution
3. THE MECHANISM — Structural forces and incentive misalignments
4. THE REAL MISTAKE — Counterintuitive errors smart leaders make (hosts disagree here)
5. THE LEVER — Specific Monday-morning moves
6. THE MONDAY MORNING — Required actionable closer ("the email you should send Monday is...")
7. THE REFRAME — One idea that permanently changes how they see the topic

---

## Host Personalities

**Alex — "The Empiricist"**
- Evidence-first. Leads with data, derives implications. Quantifies what others leave vague. Catches when narrative contradicts data.
- Signature: "Here's what the data actually shows...", "The pattern that keeps recurring is..."
- Weakness Morgan exploits: "spreadsheet blindness" — over-indexes on measurable outcomes, misses political dynamics.

**Morgan — "The Strategist"**
- Decisions-first. Follows incentives, names the unsaid thing, connects unrelated dynamics. Uses analogies and pointed stories.
- Signature: "Follow the incentives...", "Who benefits from this being true?", "I sat in a board room where..."
- Weakness Alex exploits: "strategy without a denominator" — narrative elegance without specified mechanism.

**Dynamic:** At least 2 genuine disagreements per episode. Not performative — real intellectual friction with partial concessions.

---

## Features

### Today's Briefings
- 6 AI-generated topics refreshed daily (cached in `topics_cache.json`)
- Topics pulled from AnswerRocket AR dashboard + Claude editorial judgment
- Each card: Generate Briefing button + Create Series button

### Discover Tab
- Same 6 topics with expanded detail (core tension, why it matters, common mistake, sub-questions)
- Actions per topic: 90s Preview, Commission Episode, + Series, Dismiss

### Series Tab — Two-Phase Architecture
- Create a multi-episode progressive arc (3, 6, 9, or 12 episodes)
- Input: topic card, free-text description, or article URL
- **Phase 1 — Research & Series Bible:** `generate_series_bible()` → Claude Opus + web search does real investigative research, then builds a structured series bible containing:
  - `series_thesis` / `narrative_arc` / `target_listener_transformation`
  - `research_findings`: key_facts, key_players, timeline, controversies — each assigned to specific episodes
  - Per-episode: `role_in_arc`, `central_argument`, `builds_on`, `seeds_for_future`, `bridge_to_next`, `host_dynamics`
  - `narrative_threads`: themes that weave across episodes with evolution tracking
  - `progression_logic`: explicit reasoning for episode order
- **Phase 2 — Episode Generation:** Each episode receives the full bible + summaries of all prior episodes as context. After each episode script, `summarize_episode_script()` (Sonnet) generates a continuity summary for subsequent episodes.
- Live progress tracking per episode in UI
- Series statuses: `queued → researching → producing → done | error`
- Bible and episode summaries stored in `series.json`

### Admin Tab
- **Queue management:** View active jobs, clear queued/errored jobs
- **RSS feed:** Force rebuild, copy URL
- **Auto-queue from AnswerRocket:** Pulls latest AR intelligence → generates topic → queues episode automatically

### Chat / Commission (Discover tab)
- Free-text input: topic description or article URL
- Claude generates topic object, then full episode
- Same async queue pattern

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Frontend |
| GET | `/feed.xml` | RSS podcast feed |
| GET | `/episodes/<filename>` | Serve MP3 |
| POST | `/api/generate` | Queue episode generation → returns `job_id` |
| POST | `/api/chat` | Queue chat-based generation → returns `job_id` |
| GET | `/api/job/<job_id>` | Poll job status |
| GET | `/api/queue` | List active jobs |
| POST | `/api/queue/clear` | Clear queued/errored jobs |
| POST | `/api/feed/rebuild` | Force RSS rebuild |
| POST | `/api/autoqueue` | Queue AR intelligence briefing |
| GET | `/api/series` | List all series |
| POST | `/api/series` | Create new series |
| GET | `/api/series/<series_id>` | Series status + per-episode job statuses |
| GET | `/api/topics` | Get today's topics |
| GET | `/api/episodes` | Get all episodes |
| GET | `/api/voices` | Get available ElevenLabs voices |

---

## Environment Variables (Railway)
```
ANTHROPIC_API_KEY=...
ELEVEN_LABS_API_KEY=...         # or ELEVENLABS_API_KEY
BASE_URL=https://intelligence-briefings-production.up.railway.app
DATA_DIR=                       # optional, defaults to ~/Intelligence-Briefings
```

---

## Cost Model (Dual-Model Architecture)

**Opus (scripts + series bibles):** ~$0.30–0.35 per script call (~2K input, ~3.5K output tokens)
**Opus (series bible generation):** ~$0.50–0.70 per bible (1.5K input, 6-8K output + web search)
**Sonnet (topics, suggestions, chat, episode summaries):** ~$0.06 per call

**Standalone episodes:** ~$0.30-0.35 Anthropic per episode (unchanged)
**Series episodes (with bible):**
- 3-episode series: ~$2.00-2.50 total (~$0.50 bible + $1.00 scripts + $0.18 summaries)
- 6-episode series: ~$3.50-4.00 total
- 12-episode series: ~$6.50-7.50 total

Realistic usage (15-20 episodes/week): ~$5-10/week Anthropic API depending on series mix.
ElevenLabs remains the dominant cost driver.

To downgrade scripts back to Sonnet: change `SCRIPT_MODEL` constant in `app.py`.

---

## Known Issues / Next Priorities

1. **Web search depends on account permissions** — scripts and series bibles attempt web search with graceful fallback. If Anthropic account lacks web search beta access, generation silently uses training knowledge only. Verify with `GET /api/test/web-search`.

2. **Series bible architecture needs production validation** — two-phase series generation (bible → episodes) is implemented but needs a full end-to-end run to validate bible quality, narrative coherence across episodes, and token budget under real conditions. Test with a 3-episode series first.

3. **Auto-queue not on a schedule** — currently manual trigger only (Admin tab button). Could add cron-style trigger via Railway scheduled jobs if desired.

4. **Weekly cap set to 50** — raised from original 5 to accommodate series production volume.

5. **Opus generation time** — script generation takes ~3-4 minutes with Opus (vs ~2 minutes Sonnet). Series bible generation adds ~3-5 minutes upfront (one-time). Well within Railway's 300s gunicorn timeout and the async job queue handles it, but total series wall time is longer.

6. **Series bible token size** — full bible is stored in `series.json` and returned via the API. For 12-episode series the bible can be 5-10KB. Not a problem today but monitor if series.json grows large with many series.

---

## Deploy Command
```powershell
cd "C:\Users\eddob\Claude Projects\podcast-console"
git add -A
git commit -m "your message"
railway up
```

## RSS Feed URL
```
https://intelligence-briefings-production.up.railway.app/feed.xml
```
Add this to Apple Podcasts → Listen Now → Add a Podcast by URL.
