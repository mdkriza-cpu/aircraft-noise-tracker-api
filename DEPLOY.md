# Aircraft Noise Tracker — Backend Deployment Guide
## For Martin + Xcode Claude

---

## WHAT'S IN THIS PACKAGE

```
noise-tracker-backend/
├── main.py              ← FastAPI backend (the API)
├── requirements.txt     ← Python dependencies
├── render.yaml          ← Render deployment config
└── static/
    └── index.html       ← Layer 1 health dashboard (frontend)
```

---

## STEP 1 — Get it on GitHub (required for Render)

1. Create a new GitHub repo: `aircraft-noise-tracker-api`
2. Push these files to it:
   ```
   git init
   git add .
   git commit -m "Initial backend"
   git remote add origin https://github.com/YOUR_USERNAME/aircraft-noise-tracker-api.git
   git push -u origin main
   ```

---

## STEP 2 — Deploy to Render (free tier)

1. Go to https://render.com → sign up free
2. New → Web Service → connect your GitHub repo
3. Settings:
   - **Runtime:** Python 3
   - **Build command:** `pip install -r requirements.txt`
   - **Start command:** `uvicorn main:app --host 0.0.0.0 --port $PORT`
4. Add a **Disk** (under Advanced):
   - Name: `noise-db`
   - Mount path: `/var/data`
   - Size: 1 GB (free)
5. Add environment variable:
   - Key: `DB_PATH`  Value: `/var/data/noise_tracker.db`
6. Click **Create Web Service**

Your API will be live at: `https://your-service-name.onrender.com`

**Note:** Render free tier spins down after 15 min idle. First request after idle takes ~30s.
For always-on, upgrade to Render Starter ($7/mo) or add a UptimeRobot ping.

---

## STEP 3 — Update the frontend

In `static/index.html`, find this line near the bottom:
```javascript
const API_BASE = 'https://your-app-name.onrender.com';
```
Replace with your actual Render URL.

---

## STEP 4 — Test the API

Once deployed, test it works:

```bash
# Health check
curl https://your-app.onrender.com/health

# Dashboard summary (will be empty until data is uploaded)
curl https://your-app.onrender.com/api/v1/dashboard-summary
```

---

## API SPECIFICATION FOR XCODE CLAUDE
## (The iOS app should implement this POST)

### Endpoint
```
POST https://your-app.onrender.com/api/v1/upload-session
```

### Request format
```
Content-Type: multipart/form-data
Body field name: "file"
File content: the session CSV (tab-separated, same format the app already exports)
```

### Swift example (URLSession)
```swift
func uploadSession(csvURL: URL) async throws {
    let url = URL(string: "https://your-app.onrender.com/api/v1/upload-session")!
    var request = URLRequest(url: url)
    request.httpMethod = "POST"

    let boundary = UUID().uuidString
    request.setValue("multipart/form-data; boundary=\(boundary)", forHTTPHeaderField: "Content-Type")

    let csvData = try Data(contentsOf: csvURL)
    var body = Data()
    body.append("--\(boundary)\r\n".data(using: .utf8)!)
    body.append("Content-Disposition: form-data; name=\"file\"; filename=\"session.csv\"\r\n".data(using: .utf8)!)
    body.append("Content-Type: text/csv\r\n\r\n".data(using: .utf8)!)
    body.append(csvData)
    body.append("\r\n--\(boundary)--\r\n".data(using: .utf8)!)
    request.httpBody = body

    let (data, response) = try await URLSession.shared.data(for: request)
    // Handle response — check for 200 OK
    // On failure: store csvURL locally and retry later
}
```

### Expected response (200 OK)
```json
{
  "status": "ok",
  "session_id": "2026-05-09_17-12-59_39.11281",
  "observations_inserted": 97,
  "n65": 18,
  "n70": 14,
  "n80": 0,
  "recovery_deficit": 18,
  "unique_aircraft": 22
}
```

### Error responses
- `400` — not a CSV, or CSV is empty
- `500` — server error (retry later)

### Retry strategy (recommended)
Store failed upload URLs in UserDefaults. On next app launch or when connectivity
returns, retry any queued uploads before proceeding. The backend uses INSERT OR REPLACE
so duplicate uploads of the same session are safe.

---

## DATABASE SCHEMA SUMMARY

### `observations` table
One row per measurement row in the CSV. All CSV columns are stored.
Key columns: `session_id`, `timestamp`, `dba_level`, `loudness_sone`,
`annoyance`, `callsign`, `type_code`, `operator`, `flight_phase`

### `sessions` table
One row per uploaded session with pre-computed summary stats.
Key columns: `n65`, `n70`, `n80`, `recovery_deficit`, `event_density`,
`peak_dba`, `peak_loudness_sone`

---

## WHO BENCHMARK COLUMNS (FUTURE)

The `sessions` table already has placeholder columns:
- `who_daily_average_dba` — when the app starts computing this
- `who_exceedance_pct` — % of events above WHO 45 dBA night threshold

When the app sends these, the backend will store and display them automatically.
No schema migration needed — columns are already there.

---

## LAYER 2 ANALYTICS (FUTURE)

When ready to add the smart analytics layer, extend with:
- `GET /api/v1/analytics/aircraft-psychoacoustics` — sharpness/annoyance by type
- `GET /api/v1/analytics/trends` — time series of N70/recovery deficit
- `GET /api/v1/analytics/flight-path-analysis` — bearing/elevation angle clustering

These will be computed from the `observations` table which is already storing
all the raw data needed.
```
