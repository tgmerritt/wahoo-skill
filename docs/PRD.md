# ClawHub Skill: Wahoo Fitness Cloud API Integration

**Version:** 0.1.0 (Draft)
**Author:** Puck / Tyler Merritt
**Status:** Pre-Development — Wahoo Developer Approval Pending

---

## Problem Statement

Wahoo ELEMNT head units (BOLT, ROAM, ACE) generate high-quality cycling data: power, cadence, heart rate, speed, GPS, elevation — all stored in FIT files. But there's no programmatic way to access that data. The only official API is the Wahoo Fitness Cloud API, which requires a developer application review and OAuth2 authentication.

Meanwhile, Garmin has a thriving third-party ecosystem. Wahoo has nothing. This leaves Wahoo users in a data black hole — their ride data lives in the Wahoo app, the Wahoo cloud, or Strava/Komoot, but never in their own hands via a tool they control.

## Goal

Build a ClawHub skill that gives OpenClaw agents full programmatic access to the Wahoo Fitness Cloud API, enabling:

1. **Data retrieval** — fetch ride metadata and download FIT files
2. **Data ingestion** — parse FIT files and populate a local training database
3. **Data export** — upload FIT files to third-party platforms (Strava, etc.)
4. **Health analytics** — training load, power analysis, recovery tracking

## API Overview

The Wahoo Fitness Cloud API (wahooligan) is an OAuth2 REST API with these endpoints:

### Authentication
- OAuth2 Authorization Code flow
- `client_id` + `client_secret` from Wahoo Developer Portal
- `access_token` for API calls
- `workouts_read` and `workouts_write` scopes
- App registration: Sandbox → Production (requires review)

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/auth/token` | POST | Exchange auth code for access token |
| `/api/v1/user/profile` | GET | User profile, zones, settings |
| `/api/v1/user/workouts` | GET | List workouts (metadata, file URLs) |
| `/api/v1/user/workouts/{id}` | GET | Workout detail |
| `/api/v1/user/workouts` | POST | Upload FIT file |
| `/api/v1/user/workouts/{id}/file` | GET | Download FIT file |
| `/api/v1/user/health` | GET | Health metrics (if available) |

### FIT File Download

Workout objects include a `file` field with a URL to download the raw FIT file. This is the gold — every power curve, cadence spike, heart rate zone, and GPS track.

## Skill Architecture

### Files

```
skills/wahoo/
├── SKILL.md              # OpenClaw skill definition
├── lib/
│   ├── wahoo_auth.py     # OAuth2 token management
│   ├── wahoo_api.py      # API client (requests-based)
│   └── fit_parser.py     # FIT file parser (fitparse library)
├── scripts/
│   ├── fetch_workouts.py # Fetch workout list + download FIT files
│   ├── parse_fit.py      # Parse FIT file → JSON/CSV
│   └── upload_strava.py  # Upload FIT to Strava (optional)
├── config.example.json   # Template for credentials
└── README.md             # Setup + usage docs
```

### Core Functions

1. **`wahoo_auth.py`**
   - Store `client_id`, `client_secret` securely (OpenClaw secrets)
   - Handle OAuth2 flow: generate auth URL, exchange code for token, refresh token
   - Token storage: encrypted file or OpenClaw secret store
   - Auto-refresh before expiry

2. **`wahoo_api.py`**
   - Base API client with auth header injection
   - `get_workouts(page, per_page)` — list workouts
   - `get_workout(id)` — workout detail
   - `download_workout_file(id)` — download FIT file
   - `upload_workout(file_path)` — upload FIT to Wahoo cloud
   - `get_profile()` — user profile data
   - Rate limiting, retry logic, error handling

3. **`fit_parser.py`**
   - Use `fitparse` library (Python FIT file parser)
   - Parse FIT → structured JSON
   - Extract: power, cadence, HR, speed, elevation, GPS, time, calories
   - Output formats: JSON, CSV, SQLite insert

4. **`fetch_workouts.py`**
   - Main fetch script (analogous to `fetch_strava.py`)
   - Fetch all workouts (paginated, safety limit)
   - Download FIT files for each workout
   - Parse and upsert into local SQLite DB
   - Sync log for tracking

5. **`upload_strava.py`** (optional)
   - Upload FIT files to Strava via Strava API
   - Duplicate detection (skip if already synced)
   - Configurable target (Strava, TrainingPeaks, etc.)

### Integration with Existing Pipeline

The Wahoo skill feeds into the same `~/.openclaw/workspace/training/strava.db` schema, or a separate `wahoo.db`. This means:

- **One training DB** or **Two parallel DBs** — depends on Tyler's preference
- Training analytics (TSS, IF, zone compliance) work across both sources
- Weekly report can pull from either or both DBs
- If Wahoo data is richer (HR + power simultaneously), it could become the primary source

## User Workflow

### First Setup
1. Register developer app at `https://developers.wahooligan.com`
2. Get `client_id` and `client_secret` (Sandbox for testing)
3. Install skill: `clawhub install wahoo` (or `clawhub install <our-slug>`)
4. Configure credentials in skill config
5. Run OAuth2 flow (skill opens browser or provides URL for Tyler to authorize)
6. Store access token
7. Run `fetch_workouts.py` — initial data pull

### Daily/Weekly
- Skill runs on demand or via heartbeat
- Fetches new workouts since last sync
- Downloads and parses FIT files
- Updates local database
- Generates training summary

### On Request
- Tyler: "Show me my power curve from last Saturday's ride"
- Tyler: "Upload my Wahoo ride to Strava"
- Tyler: "What was my avg power last week?"
- Tyler: "Download my FIT file from the Hammer ride"

## Security Considerations

- OAuth2 credentials stored in OpenClaw secrets (not in plaintext)
- Access tokens have expiry — auto-refresh mechanism
- FIT files are local only — not exposed to any API unless Tyler explicitly requests upload
- Skill follows OpenClaw sandbox model
- No Wahoo credentials shared with any agent other than Puck

## Success Metrics

1. **Developer approval** — Wahoo reviews and approves production app
2. **Data sync** — all Wahoo workouts synced to local DB
3. **FIT parsing** — accurate extraction of power, cadence, HR, GPS
4. **Skill quality** — passes ClawHub review, documented, tested
5. **Adoption** — other Wahoo users install and benefit

## Competitive Positioning

| Platform | Third-Party API | ClawHub Skill | Data Export |
|----------|----------------|---------------|-------------|
| Strava   | ✅ Full API     | ✅ Strava skill | ✅ FIT download |
| Garmin   | ✅ Full API     | ⏳ Pending      | ✅ FIT download |
| Wahoo    | ⚠️ OAuth2 only  | 🔨 **Building**  | ⚠️ Cloud only |
| Zwift    | ❌ No API       | —             | ❌ Closed |

Wahoo is the last major cycling platform without broad third-party tooling. This skill fills that gap.

## Timeline

| Phase | Action | ETA |
|-------|--------|-----|
| 1 | Register Wahoo developer app | Immediate |
| 2 | Sandbox approval (automatic) | 1-2 days |
| 3 | Build skill prototype (fetch + parse) | 1 week |
| 4 | Test with real Wahoo data | 1 week |
| 5 | Submit for production approval | Week 3 |
| 6 | Production review (Wahoo decides timeline) | TBD |
| 7 | Publish to ClawHub | After production approval |

## Open Questions

1. **DB strategy** — One DB for Strava + Wahoo, or separate?
2. **Upload direction** — Wahoo → Strava, or also Strava → Wahoo?
3. **Health endpoint** — Does the Wahoo API actually expose health data, or just workouts?
4. **Token storage** — OpenClaw secrets, encrypted file, or keyring?
5. **CLI vs tool** — Skill with scripts, or full OpenClaw tool?
6. **FIT file parsing** — `fitparse` library? Or write our own parser?
7. **Sync frequency** — On-demand, daily heartbeat, or continuous?
8. **Pricing** — ClawHub free, paid, or tip-based?

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Wahoo rejects production app | Medium | High | Stay in Sandbox, document clearly |
| API changes without notice | Low | Medium | Version pinning, monitoring |
| OAuth2 flow complexity | Medium | Low | Document clearly, provide CLI helpers |
| FIT file format changes | Low | Low | `fitparse` handles most cases |
| Rate limiting | Low | Medium | Implement backoff, cache responses |

---

*This PRD is a living document. It will evolve as we build.*
