# Wahoo Developer Account — Setup Plan

## Phase 1: Developer Registration (Do This Now)

### 1.1 Create Wahoo Developer Account
- Go to `https://developers.wahooligan.com`
- Sign up with your Wahoo/Fitline account (or create one)
- This is the portal where you register applications

### 1.2 Register Application (Sandbox)
- In the Developer Portal, create a new application
- Fill in:
  - **App Name:** OpenClaw Wahoo Skill (or "Puck Training Tools")
  - **Description:** OpenClaw integration for Wahoo Fitness Cloud API. Provides programmatic access to workout data, FIT file download, and cycling analytics.
  - **Redirect URI:** `http://localhost:8080/callback` (local OAuth2 callback)
  - **Scopes:** `workouts_read` (minimum), `workouts_write` (for upload capability)
  - **App Type:** Sandbox (this is automatic — no review needed)

### 1.3 Get Credentials
- After creating the app, you'll get:
  - `client_id` — unique identifier for your app
  - `client_secret` — secret key for your app
- **Store these in OpenClaw secrets:**
  ```bash
  openclaw secrets set wahoo.client_id "..."
  openclaw secrets set wahoo.client_secret "..."
  ```

### 1.4 Complete OAuth2 Authorization Flow
- Open the authorization URL in a browser:
  ```
  https://id.wahoo.com/auth/oauth2/v1/authorize
    ?response_type=code
    &client_id=<client_id>
    &redirect_uri=http://localhost:8080/callback
    &scope=workouts_read
  ```
- Log in with your Wahoo account
- Approve the permissions
- You'll be redirected to `http://localhost:8080/callback?code=<AUTH_CODE>`
- Exchange the auth code for an access token:
  ```
  POST https://id.wahoo.com/auth/oauth2/v1/token
    Content-Type: application/x-www-form-urlencoded
  
    grant_type=authorization_code
    &code=<AUTH_CODE>
    &client_id=<client_id>
    &client_secret=<client_secret>
    &redirect_uri=http://localhost:8080/callback
  ```
- Response includes:
  - `access_token` — use for all API calls (short-lived)
  - `refresh_token` — use to get new access tokens (long-lived)
  - `expires_in` — seconds until token expires

### 1.5 Store Tokens
```bash
openclaw secrets set wahoo.access_token "..."
openclaw secrets set wahoo.refresh_token "..."
```

## Phase 2: Build the Skill (After Approval)

### 2.1 Scaffold the Skill
```bash
mkdir -p skills/wahoo/{lib,scripts}
```

### 2.2 Implement Core Components
- `wahoo_auth.py` — OAuth2 token management + auto-refresh
- `wahoo_api.py` — API client with all endpoints
- `fit_parser.py` — FIT file parsing using `fitparse` library

### 2.3 Implement Scripts
- `fetch_workouts.py` — fetch + download + parse + DB insert
- `parse_fit.py` — standalone FIT parser for ad-hoc analysis

### 2.4 Write SKILL.md
- Define the skill's capabilities, parameters, and usage patterns
- Follow ClawHub skill format

### 2.5 Test Thoroughly
- Test with real Wahoo data (at least 5-10 rides)
- Verify FIT parsing accuracy against Strava data
- Test OAuth2 refresh flow
- Test error handling (rate limits, expired tokens, network errors)

## Phase 3: Production Approval

### 3.1 Submit for Production
- In Developer Portal, switch app from Sandbox to Production
- Provide:
  - **App Description** — expanded, professional
  - **Use Case** — explain the training analytics tool
  - **Data Usage** — describe how data is stored and used
  - **Privacy Policy** — basic statement about data handling
- Wahoo will review (timeline unknown, could be days to weeks)

### 3.2 If Approved
- Continue as-is, now with production access
- Publish to ClawHub

### 3.3 If Rejected
- Stay in Sandbox (it works, just labeled "test")
- Iterate on the submission
- Document the limitation clearly

## Phase 4: ClawHub Publishing

### 4.1 Prepare for Publish
- Finalize README.md
- Ensure SKILL.md is complete
- Add config.example.json
- Write test scripts

### 4.2 Publish
```bash
cd skills/wahoo
clawhub publish . --slug wahoo --name "Wahoo Fitness Cloud API" --version 1.0.0 --changelog "Initial Wahoo Cloud API integration: OAuth2 auth, workout fetch, FIT file download/parse, local DB sync"
```

---

## What Tyler Needs To Do

1. **Go to developers.wahooligan.com** — create account if needed
2. **Register a Sandbox app** — fill in the form (takes 5 minutes)
3. **Give me the client_id and client_secret** — I'll handle the OAuth2 flow
4. **Test ride** — go on a ride, let it sync to Wahoo cloud, then I'll pull it down

## What I'll Do

1. **Handle the OAuth2 flow** — generate auth URL, exchange tokens, store credentials
2. **Build the skill** — all Python code, parsing, DB integration
3. **Write documentation** — SKILL.md, README, setup guide
4. **Test everything** — with real data from your Wahoo
5. **Submit for production** — handle the review process
6. **Publish to ClawHub** — once approved

## Estimated Timeline

| Step | Time |
|------|------|
| Developer registration | 10 min (Tyler) |
| OAuth2 setup + testing | 30 min (me) |
| Skill prototype | 2-3 days |
| Real data testing | 1 day |
| Production submission | 1 hour |
| Production approval | Wahoo decides (could be fast, could be slow) |
| ClawHub publish | 1 day after approval |

---

*Note: The Sandbox app works identically to Production — it's just a flag on Wahoo's side. We can build and test everything in Sandbox. Production approval is only needed if we want to publish the skill publicly or if Wahoo changes sandbox limitations.*
