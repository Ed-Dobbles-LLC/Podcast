# Intelligence Briefings -- Project Context for Claude

## What This Is
An AI-powered executive podcast platform hosted on Railway. It generates 5-10 minute audio briefings on topics relevant to a senior analytics/AI executive (Ed Dobbles, CAO at Overproof). Two AI hosts -- Alex ("The Empiricist") and Morgan ("The Strategist") -- debate and analyze topics in a structured dialogue format with distinct rhetorical identities. Episodes are published as an RSS feed consumable by Apple Podcasts or any podcast app.

**Production URL:** https://intelligence-briefings-production.up.railway.app
**Listener URL:** https://intelligence-briefings-production.up.railway.app/listen
**Local project path:** `C:\Users\eddob\Claude Projects\podcast-console\`
**Railway project ID:** 8d9b2720-162c-45a4-a209-1af24d88583e

---

## Stack
- **Backend:** Python / Flask, deployed on Railway via Dockerfile (python:3.12-slim)
- **TTS:** ElevenLabs (eleven_turbo_v2_5, mp3_44100_128)
- **Script generation:** Anthropic Claude API (raw HTTP via urllib -- no SDK dependency) -- **dual-model architecture:**
  - `SCRIPT_MODEL` = `claude-opus-4-20250514` -- used for script generation, series bibles (premium quality)
  - `EDITORIAL_MODEL` = `claude-sonnet-4-20250514` -- used for topic generation, chat-to-topic, suggestions, episode summaries (cost-effective)
- **Data persistence:** Railway volume mounted at `~/Intelligence-Briefings/`
- **Frontend:** Two single-page HTML/CSS/JS templates (production console + public listener)
- **Audio concatenation:** ffmpeg (installed in Docker image) -- concatenates per-segment MP3s into episode files with correct duration headers. Pure-Python fallback strips Xing/Info VBR headers if ffmpeg unavailable.
- **Process manager:** gunicorn gthread workers, 300s timeout (Procfile + Dockerfile both define this)

## Key Files
```
podcast-console/
+-- app.py                          # Full backend -- all logic lives here (~1937 lines)
+-- templates/
|   +-- index.html                  # Production console (admin UI)
|   +-- listen.html                 # Public listener page (shareable)
+-- Procfile                        # gunicorn --workers 2 --worker-class gthread --threads 4 --timeout 300
+-- Dockerfile                      # python:3.12-slim, pip install, gunicorn CMD
+-- railway.json                    # Docker build config, healthcheck on /
+-- requirements.txt                # flask, elevenlabs, feedgen, mutagen, gunicorn
+-- .gitignore                      # __pycache__, .env, /data/
+-- CLAUDE.md                       # This file
```

## Dependencies (requirements.txt)
- flask==3.1.0
- elevenlabs==1.50.0 (TTS audio generation)
- feedgen==1.0.0 (RSS feed builder)
- mutagen==1.47.0 (MP3 metadata/duration reading)
- gunicorn==23.0.0

Note: Anthropic API is called via raw urllib.request (no `anthropic` SDK). ElevenLabs uses its SDK.

## Data Directory (Railway Volume)
```
~/Intelligence-Briefings/
+-- episodes/                       # MP3 files + per-episode segment dirs
+-- episodes.json                   # Episode metadata list
+-- topics_cache.json               # Today's 6 topics (refreshes daily)
+-- suggestions_cache.json          # 10 discover suggestions (refreshes daily)
+-- jobs.json                       # Async job queue state
+-- series.json                     # Series metadata + bibles
+-- production_log.json             # Production count log (weekly cap tracking)
+-- engagement_log.json             # Listener engagement events (capped at 2000)
+-- nightly_trailer_log.json        # Nightly cron run log (idempotency)
+-- morning_prep_log.json           # Morning cron run log (idempotency)
+-- feed.xml                        # RSS feed (auto-rebuilt after each episode)
```

---

## Architecture: Async Job Queue

Railway has a 60-second HTTP proxy timeout. All generation tasks (2-4 minutes) run in background threads.

**Flow:**
1. Client POSTs to `/api/generate` or `/api/chat` or `/api/series` -- returns `{job_id, status: "queued"}` in <1 second
2. Background thread runs full pipeline, updates job status/progress in `jobs.json`
3. Client polls `GET /api/job/<job_id>` every 3 seconds
4. When `status === "done"`, result contains full episode object
5. UI renders audio player

**Job statuses:** `queued -> running -> done | error`

---

## Episode Generation Pipeline

1. `generate_grounded_script(topic, depth, production_brief, series_bible, prior_episode_summaries)` -- calls Claude Opus API (with web search fallback) -- returns JSON array of `{host, text}` segments. When `series_bible` is provided, injects full series context (assigned research, narrative threads, prior episode summaries, arc instructions) into the prompt.
2. `generate_episode_audio(client, script, ep_dir, voice_a_id, voice_b_id)` -- calls ElevenLabs per segment -- concatenates MP3s
3. Episode saved to `episodes.json`, RSS feed rebuilt automatically

**Trailer episodes:** Shorter format using `build_trailer_script()` -- generates 90-second preview hooks for topics. Produced by nightly cron.

**Segment targets (strictly enforced in prompt):**
- executive: minimum 10 segments
- standard: minimum 16 segments
- deep: minimum 24 segments
- Each segment: 3-5 substantial sentences

**Web search:** Enabled with graceful fallback. Scripts attempt `use_web_search=True` first; if it fails (permissions, quota), falls back to Claude's training knowledge silently. Requires `anthropic-beta: web-search-2025-03-05` header (handled automatically in `call_anthropic`).

**Script structure (7 sections):**
1. COLD OPEN -- Drop into tension immediately
2. GROUND IT -- Named examples with natural source attribution
3. THE MECHANISM -- Structural forces and incentive misalignments
4. THE REAL MISTAKE -- Counterintuitive errors smart leaders make (hosts disagree here)
5. THE LEVER -- Specific Monday-morning moves
6. THE MONDAY MORNING -- Required actionable closer ("the email you should send Monday is...")
7. THE REFRAME -- One idea that permanently changes how they see the topic

---

## Host Personalities

**Alex -- "The Empiricist"**
- Evidence-first. Leads with data, derives implications. Quantifies what others leave vague. Catches when narrative contradicts data.
- Signature: "Here's what the data actually shows...", "The pattern that keeps recurring is..."
- Weakness Morgan exploits: "spreadsheet blindness" -- over-indexes on measurable outcomes, misses political dynamics.

**Morgan -- "The Strategist"**
- Decisions-first. Follows incentives, names the unsaid thing, connects unrelated dynamics. Uses analogies and pointed stories.
- Signature: "Follow the incentives...", "Who benefits from this being true?", "I sat in a board room where..."
- Weakness Alex exploits: "strategy without a denominator" -- narrative elegance without specified mechanism.

**Dynamic:** At least 2 genuine disagreements per episode. Not performative -- real intellectual friction with partial concessions.

---

## Features

### Today's Briefings (index.html)
- 6 AI-generated topics refreshed daily (cached in `topics_cache.json`)
- Topics pulled from AnswerRocket AR dashboard + Claude editorial judgment + fallback topics
- Each card: Generate Briefing button + Create Series button

### Discover Tab (index.html)
- 10 AI-generated suggestions from engagement-informed prompt (cached in `suggestions_cache.json`)
- Behavioral signals (strong interest, moderate interest, dismissed) feed into recommendations
- Actions per topic: Commission Episode, + Series, Dismiss
- Free-text chat input: describe a topic or paste an article URL to commission

### Series Tab -- Two-Phase Architecture (index.html)
- Create a multi-episode progressive arc (3, 6, 9, or 12 episodes)
- Input: topic card, free-text description, or article URL
- **Phase 1 -- Research & Series Bible:** `generate_series_bible()` -- Claude Opus + web search does real investigative research, then builds a structured series bible containing:
  - `series_thesis` / `narrative_arc` / `target_listener_transformation`
  - `research_findings`: key_facts, key_players, timeline, controversies -- each assigned to specific episodes
  - Per-episode: `role_in_arc`, `central_argument`, `builds_on`, `seeds_for_future`, `bridge_to_next`, `host_dynamics`
  - `narrative_threads`: themes that weave across episodes with evolution tracking
  - `progression_logic`: explicit reasoning for episode order
- **Phase 2 -- Episode Generation:** Each episode receives the full bible + summaries of all prior episodes as context. After each episode script, `summarize_episode_script()` (Sonnet) generates a continuity summary for subsequent episodes.
- Live progress tracking per episode in UI
- Series statuses: `queued -> researching -> producing -> done | error`
- Bible and episode summaries stored in `series.json`

### Episodes Tab (index.html)
- List all produced episodes with audio players
- Delete episodes (with confirmation)

### Admin Tab (index.html)
- **Health check:** ElevenLabs character usage + Anthropic API status
- **Credit warnings:** Banner in nav when API credits are low
- **Web search test:** Verify Anthropic web search beta access
- **Engagement stats:** View listener behavioral signals
- **Queue management:** View active jobs, clear queued/errored jobs
- **RSS feed:** Force rebuild, copy URL
- **Nightly trailers cron:** Queue 6 AI-selected trailer topics overnight
- **Auto-queue from AnswerRocket:** Pulls latest AR intelligence, generates topic, queues episode
- **Public listener link:** Shareable link to `/listen`

### Public Listener Page (listen.html)
- Clean, shareable episode player -- no production controls
- Episode list with search and filters (All / Series / Standalone)
- Now Playing bar with audio controls
- RSS subscribe modal (Apple Podcasts deep link)
- Share to LinkedIn per-episode
- Engagement tracking (play starts, progress milestones, completions)

### Cron Endpoints
- **Nightly trailers** (`/api/cron/nightly-trailers`): Generates 6 engagement-informed trailer topics at 22:00 CT. Idempotent (won't re-run same day unless forced). Uses behavioral signals to weight topic selection.
- **Morning prep** (`/api/cron/morning-prep`): Pre-warms topics cache and suggestions cache at 05:30 CT so page loads instantly. Idempotent.
- **Auto-queue** (`/api/cron/autoqueue`): Pulls from AnswerRocket dashboard, generates topic, queues full episode.

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Production console (index.html) |
| GET | `/listen` | Public listener page (listen.html) |
| GET | `/feed.xml` | RSS podcast feed |
| GET | `/episodes/<filename>` | Serve MP3 |
| POST | `/api/generate` | Queue episode generation -- returns `job_id` |
| POST | `/api/chat` | Queue chat-based generation -- returns `job_id` |
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
| DELETE | `/api/episodes/<ep_id>` | Delete an episode + its files |
| GET | `/api/voices` | Get available ElevenLabs voices |
| GET | `/api/health` | ElevenLabs + Anthropic health check |
| GET | `/api/test/web-search` | Verify web search beta access |
| GET | `/api/engagement` | Engagement summary (strong/moderate/dismissed) |
| POST | `/api/engagement` | Log engagement event |
| GET | `/api/discover/suggestions` | Get 10 engagement-informed topic suggestions |
| GET,POST | `/api/cron/nightly-trailers` | Queue 6 overnight trailers (cron) |
| GET | `/api/cron/nightly-trailers/status` | Check nightly run status |
| GET,POST | `/api/cron/autoqueue` | Cron-triggered AR auto-queue |
| GET,POST | `/api/cron/morning-prep` | Pre-warm topics + suggestions caches |

---

## Environment Variables (Railway)
```
ANTHROPIC_API_KEY=...
ELEVEN_LABS_API_KEY=...         # or ELEVENLABS_API_KEY
BASE_URL=https://intelligence-briefings-production.up.railway.app
DATA_DIR=                       # optional, defaults to ~/Intelligence-Briefings
CRON_SECRET=                    # optional, protects cron endpoints
```

---

## Cost Model (Dual-Model Architecture)

**Opus (scripts + series bibles):** ~$0.30-0.35 per script call (~2K input, ~3.5K output tokens)
**Opus (series bible generation):** ~$0.50-0.70 per bible (1.5K input, 6-8K output + web search)
**Sonnet (topics, suggestions, chat, episode summaries):** ~$0.06 per call

**Standalone episodes:** ~$0.30-0.35 Anthropic per episode
**Series episodes (with bible):**
- 3-episode series: ~$2.00-2.50 total (~$0.50 bible + $1.00 scripts + $0.18 summaries)
- 6-episode series: ~$3.50-4.00 total
- 12-episode series: ~$6.50-7.50 total

Realistic usage (15-20 episodes/week): ~$5-10/week Anthropic API depending on series mix.
ElevenLabs remains the dominant cost driver.

To downgrade scripts back to Sonnet: change `SCRIPT_MODEL` constant in `app.py`.

---

## Encoding Convention

Both HTML templates (index.html, listen.html) are saved as **pure ASCII** files. All special characters use:
- **HTML entities** (`&#x26A1;`) in markup sections
- **JS unicode escapes** (`\u26A1`) inside `<script>` blocks

This eliminates all UTF-8 encoding ambiguity regardless of server, proxy, or browser charset handling. When editing templates, never introduce raw multi-byte Unicode characters -- always use the entity/escape form.

`app.py` uses UTF-8 em dashes in Python prompt strings sent to the Claude API. These are fine since Python handles UTF-8 natively and they never reach the browser.

---

## Known Issues / Next Priorities

1. **Web search depends on account permissions** -- scripts and series bibles attempt web search with graceful fallback. If Anthropic account lacks web search beta access, generation silently uses training knowledge only. Verify with `GET /api/test/web-search`.

2. **Series bible architecture needs production validation** -- two-phase series generation (bible -> episodes) is implemented but needs a full end-to-end run to validate bible quality, narrative coherence across episodes, and token budget under real conditions. Test with a 3-episode series first.

3. **Cron jobs require external scheduler** -- nightly trailers (22:00 CT) and morning prep (05:30 CT) endpoints exist but need an external trigger (cron-job.org, Railway cron, etc.). Currently manual-only from Admin tab.

4. **Weekly cap set to 50** -- raised from original 5 to accommodate series production volume. Tracked in `production_log.json`.

5. **Opus generation time** -- script generation takes ~3-4 minutes with Opus (vs ~2 minutes Sonnet). Series bible generation adds ~3-5 minutes upfront (one-time). Well within Railway's 300s gunicorn timeout and the async job queue handles it, but total series wall time is longer.

6. **Series bible token size** -- full bible is stored in `series.json` and returned via the API. For 12-episode series the bible can be 5-10KB. Not a problem today but monitor if series.json grows large with many series.

7. **Duplicate Procfile/Dockerfile CMD** -- both define the gunicorn command. Railway uses the Dockerfile (per railway.json). Procfile is unused unless switching to buildpack deploy. Keep them in sync or remove Procfile.

8. **No anthropic SDK dependency** -- all Anthropic API calls use raw urllib.request. This is intentional (fewer dependencies) but means no automatic retries, streaming, or SDK-level error handling. The `call_anthropic()` function at app.py:284 is the single abstraction.

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
Add this to Apple Podcasts via Listen Now -> Add a Podcast by URL.
