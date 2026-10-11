Proactive Air Quality Insight Agent
A backend service that watches hourly air quality data from OpenAQ for five cities and decides, on its own, when a user is worth interrupting and what to tell them. It sends a short, natural message with context and one clear next action. Silence is the default.
> **Demo video:** _add your Loom link here_
> **Design note:** [docs/DECISIONS.md](docs/DECISIONS.md) · **Notification history:** [docs/NOTIFICATION_HISTORY.md](docs/NOTIFICATION_HISTORY.md) · **Evaluation:** [docs/EVAL_REPORT.md](docs/EVAL_REPORT.md) · **AI usage / transcript:** [docs/AI_USAGE.md](docs/AI_USAGE.md)
What it does
Ingestion: pulls hourly PM2.5 for 5 cities (Delhi, Bengaluru, London, Reykjavik, Los Angeles), converts it to US AQI, and stores full history with quality flags (invalid values, stuck sensors).
Detection: a small learned model per station (baseline, time-of-day pattern, noise level) plus a CUSUM change-point alarm. It detects a spike, a sustained trend, a sustained elevation and a return to normal. A noisy city needs a bigger move than a clean one, and each user has a sensitivity multiplier.
Dedup and rate limiting: a per-user memory. If the air stays bad for six hours the user gets one message, not six. It speaks again only on a worse AQI band, a major worsening, or recovery, with a cooldown that backs off (3h, 6h, 12h, 24h) and decays after quiet periods.
Messages: short notifications that reference past context (an earlier alert, "highest reading since Saturday") and end with one action tailored to the user (runner, cyclist, parent, commuter). An LLM writes the text if `ANTHROPIC_API_KEY` is set. Otherwise a template does, and it is also the fallback if the LLM call fails.
Observability: every evaluation is logged, including the ones where nothing was sent, with the model, every check's pass or fail, the data window, and the user's state before and after.
Evaluation harness: replays history through the same pipeline and reports caught versus missed events, false positives and alert volume.
Architecture
```
OpenAQ v3 --hourly--> ingest --> readings (history + quality flags)
                                      |
                  fit per-station model (baseline, time-of-day shape, noise)
                                      |
              z-scores + CUSUM change-point --> candidate: spike | trend | elevated | recovery
                                      |            (user sensitivity scales the thresholds)
                 dedup / cooldown state machine (per user) --> send | suppress | none
                                      |
                 message builder (LLM if key set, template fallback)
                                      |
      decisions (EVERY evaluation, with inputs + checks) + notifications --> API + dashboard
```
Quick start
Requires Python 3.10+.
```bash
pip install -r requirements.txt
python -m pytest -q
```
1. Offline demo (no network, no keys)
Replays 28 simulated days through the full pipeline.
```bash
python -m aqi_agent demo          # writes demo.db and the docs/ reports
```
Start the dashboard on that data.
macOS / Linux:
```bash
AQI_DB=demo.db AQI_SCHEDULER=off AQI_SOURCE=none python -m aqi_agent serve
```
Windows PowerShell:
```powershell
$env:AQI_DB="demo.db"; $env:AQI_SCHEDULER="off"; $env:AQI_SOURCE="none"; python -m aqi_agent serve
```
Open http://127.0.0.1:8000.
Narrated terminal walkthrough for screen recording: `python -m scripts.demo_walkthrough`
Captioned demo video: `python -m scripts.make_video` (needs Pillow and ffmpeg).
2. Live against OpenAQ
Get a free API key at https://explore.openaq.org/register, then:
```bash
export OPENAQ_API_KEY=...          # PowerShell: $env:OPENAQ_API_KEY="..."
export ANTHROPIC_API_KEY=...       # optional: LLM-written messages
python -m aqi_agent check-openaq   # resolves a PM2.5 sensor per city and prints recent rows
python -m aqi_agent serve          # hourly job at HH:05, backfills 7 days at startup
```
Other commands: `python -m aqi_agent tick` (one ingest and evaluate pass) and `python -m aqi_agent eval --source sim|db`.
Observability endpoints
Endpoint	What it shows
`/`	Dashboard: chart per city with alert markers, notifications, recent decisions, per-user memory
`/api/decisions`	One row per evaluation. Filters: `station_id`, `user_id`, `action`, `candidate_only`
`/api/decisions/{id}`	Full trace: fitted model, thresholds, every check with numbers, 24h window with z-scores, policy state before and after, the notification and its context
`/api/notifications`	Notification history
`/api/state`	Dedup memory per user
`/api/readings/{station}`, `/api/ingest_runs`, `/api/stations`, `POST /api/tick`	Data and ingestion health
To debug "why didn't I get a nudge?", open the decision for that hour. The `reason` field is one of `no_candidate`, `stale_data`, `insufficient_history`, `duplicate_ongoing_episode`, `flap_cooldown_recent_alert`, `worsening_but_in_cooldown` and so on, and `checks` shows which detector condition failed.
Results (simulated replay)
28 days × 5 cities, with injected spikes, trends, sensor glitches, stuck sensors and missing hours (ground truth known):
Metric	Result
Spikes caught	9 / 10
Trends caught	6 / 6
Recoveries caught	10 / 15
Precision	0.59
Alerts per city-day	0.49 (a naive "AQI > 100 every hour" alerter sends 5.88)
Median detection delay	1 hour
Injected sensor glitches and stuck sensors produced no alerts. Full table: docs/EVAL_REPORT.md.
Project layout
```
aqi_agent/
  detector.py   learned model + CUSUM + spike/trend/elevated/recovery checks
  policy.py     dedup / cooldown state machine
  engine.py     history -> model -> candidate -> policy -> message -> decision log
  messages.py   context, action selection, LLM and template messages
  ingest.py, openaq.py   ingestion and the OpenAQ v3 client
  simulator.py, evaluate.py   ground-truth simulator and replay harness
  app.py        FastAPI service, scheduler, dashboard
tests/          8 tests (AQI conversion, glitch rejection, dedup, stuck sensor, OpenAQ client with a mock)
scripts/        demo, narrated walkthrough, video generator
docs/           design note, evaluation, notification history, AI usage
```
Limitations
The OpenAQ client has only been tested against a mocked response. OpenAQ was not reachable from the environment this was built in, so the client follows the documented v3 response shapes but has not run against the live API. Run `python -m aqi_agent check-openaq` first. Everything after ingestion is source-independent.
The evaluation uses simulated data. Detector thresholds were tuned on the simulator, so treat the numbers as a regression check, not proof of real-world accuracy. After you collect real history, `python -m aqi_agent eval --source db` replays it with simple model-free labels.
Ordinary noise swings in the simulated series are scored as false positives even though a real user might want to hear about them.
SQLite and an in-process scheduler are fine for a handful of stations. Use Postgres and a proper job queue for scale.
Submission checklist
[ ] GitHub repository
[ ] Demo video link (Loom) at the top of this README
[ ] Decision / architecture note: docs/DECISIONS.md
[ ] Transcript of AI tools used: export the chat to `docs/AI_TRANSCRIPT.md`
[ ] Notification history: docs/NOTIFICATION_HISTORY.md (regenerate with real OpenAQ data if you can)
