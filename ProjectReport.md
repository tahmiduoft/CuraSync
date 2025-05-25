# VitalSync – Project Report  
*For iOS & Android*

---

## 1  Executive Summary  

Community-health workers (CHWs) in low-connectivity regions need to record patient data offline, receive preliminary guidance, and share information with clinicians once a signal appears. **VitalSync** delivers a cross-platform React Native app plus a FastAPI backend that together provide:

| Capability                | Delivered Value                                                                                          |
|---------------------------|-----------------------------------------------------------------------------------------------------------|
| **Offline capture**       | Visits—including symptoms, vitals, photos, audio—saved to an **encrypted SQLite queue** on the device.   |
| **Background sync**       | Queue uploaded via **presigned Supabase URLs** when connectivity returns.                                |
| **AI-assisted triage**    | Pre-trained APIs (Infermedica, Google Vision, Google Speech-to-Text) provide symptom advice.             |
| **Interactive analytics** | CHWs & supervisors view **charts, heat-maps, and forecasts inside the app**—not just email reports.      |
| **Automated alerts**      | Nightly back-end analysis triggers z-score spike warnings and summary PDFs emailed to administrators.     |
| **Security**              | SQLCipher, JWT auth, HTTPS/TLS 1.3, Supabase Row-Level Security.                                         |

A future web dashboard can be added with minimal changes, but the MVP already offers full mobile functionality.

---

## 2  Objectives & KPIs  

| # | Objective                                                         | KPI                                          |
|---|-------------------------------------------------------------------|----------------------------------------------|
| 1 | ≥ 90 % of visits saved offline without data loss                  | Avg. time-to-save < 25 s                     |
| 2 | AI advice displayed ≤ 3 s after tap                               | p95 latency ≤ 3 s                            |
| 3 | Queue synced ≤ 10 min after connectivity                          | Sync success ≥ 98 %                          |
| 4 | In-app analytics page renders ≤ 2 s                               | p95 chart load ≤ 2 s                         |
| 5 | Nightly z-score alerts and PDF summary delivered                  | Email sent before 06:00 local                |
| 6 | TestFlight & Play Store beta by Month 6                           | ≥ 50 pilot CHWs onboarded                    |

---

## 3  System Architecture  

```
╔═════════════════════════════════════════════════════════════════════╗
║ 1. Mobile Layer (React Native)                                     ║
║    • VisitForm.jsx – offline form                                  ║
║    • AnalyticsScreen.jsx – renders charts & maps                   ║
║    • Local DB – SQLite + SQLCipher                                 ║
║    • Media Capture – image-picker / audio-recorder                 ║
║    • Background Sync – react-native-background-fetch               ║
║    • AI Advice – Infermedica REST call                             ║
╚═════════════════════════════════════════════════════════════════════╝
           │ HTTPS (JWT)
           ▼
╔═════════════════════════════════════════════════════════════════════╗
║ 2. FastAPI Layer                                                   ║
║    • /visits (POST) – store visit                                  ║
║    • /stats  (GET) – JSON stats for charts                         ║
║    • /charts (GET) – Plotly PNG/SVG for low-bandwidth clients      ║
║    • /media/signed (GET) – presigned Supabase URL                  ║
║    • JWT auth, rate limit, Celery enqueue                          ║
╚═════════════════════════════════════════════════════════════════════╝
           │ SQL                              │ Celery
           │                                  ▼
╔═════════════════════════════╦═══════════════════════════════════════╗
║ 3a. Supabase Storage        ║ 3b. Celery + Redis                    ║
║     • encrypted photos      ║    • AI API calls                     ║
║     • encrypted audio       ║    • Nightly Pandas stats job         ║
║                             ║    • Alert email w/ PDF               ║
╚═════════════════════════════╩═══════════════════════════════════════╝
           │                                     │ charts / CSV
           │ SQL copy                            ▼
           ▼
╔═════════════════════════════════════════════════════════════════════╗
║ 4. Database & Analytics                                            ║
║    • Supabase Postgres (JSONB, PostGIS)                            ║
║    • Pandas, Statsmodels, Prophet for forecasts                    ║
╚═════════════════════════════════════════════════════════════════════╝
```

---

## 4  Core User Flows  

### 4.1 Capture & Queue  

1. CHW opens VitalSync offline.  
2. Completes visit form → data + media AES-encrypted into SQLite queue.  
3. Optionally taps **Get Advice** (online only): Infermedica returns probable conditions and self-care tips with a disclaimer.  

### 4.2 Background Sync  

`react-native-background-fetch` wakes every 15 minutes:  
- If online, uploads media via presigned URLs and posts visit JSON to `/visits`.  
- On success deletes the queue row; otherwise retries with exponential back-off.  

### 4.3 Server-Side Processing & AI  

Celery pipeline after each visit:  
1. Infermedica (if not already used on device)  
2. Google Vision → photo labels  
3. Google Speech → audio transcript  
4. Store results in `visits.ai_jsonb`.  

### 4.4 Nightly Analytics  

- Pandas job aggregates last 30 days per clinic/district.  
- Computes:  
  - Descriptive stats (totals, averages)  
  - Trend lines  
  - z-score alerts for spikes  
  - Prophet forecast (optional)  
- Saves results to `stats_daily`; renders Plotly PNGs/SVGs; emails PDF summary to supervisors.  

### 4.5 In-App Analytics Screen  

- **Stats endpoint** returns JSON arrays for charts.  
- **Charts endpoint** returns small PNG/SVG for extreme low-bandwidth fallback.  
- Mobile renders:  
  - **Descriptive dashboard** (totals, averages).  
  - **Trend line** (fever % over time).  
  - **Heat-map** (symptom clusters using MapboxGL) if GPS data present.  
  - **Forecast card** (Prophet prediction for next 7 days).  
  - Red alert banner if z-score > 2.  

---

## 5  Statistical & Visualization Components  

| Analysis Type       | Implementation Details (Backend)             | Mobile Display (React Native)                               |
|---------------------|----------------------------------------------|-------------------------------------------------------------|
| Descriptive Stats   | `GROUP BY clinic_id` SQL → JSON              | Table & summary cards (`react-native-paper`)                |
| Trend Detection     | Pandas rolling mean; Matplotlib line PNG     | Live chart via `VictoryNative` or `react-native-svg-charts` |
| Alert Thresholds    | z-score or EWMA; triggers Celery email       | Red banner + modal detail                                   |
| Heat-map            | PostGIS point aggregation to GeoJSON grid    | MapboxGL choropleth layer                                   |
| Predictive Trends   | Prophet forecast saved to `stats_daily`      | Forecast card (sparkline + narrative text)                  |

All charts are cached client-side with React Query and update after each successful sync.

---

## 6  Technology Stack  

| Layer                     | Tech / Library                                             | Justification                                  |
|---------------------------|------------------------------------------------------------|------------------------------------------------|
| Mobile                    | **React Native 0.73** (TypeScript)                         | JS ecosystem, native modules                   |
| Charts                    | `VictoryNative`, `react-native-svg`, MapboxGL             | Lightweight, offline-cacheable                 |
| Local DB                  | `react-native-sqlite-storage` + SQLCipher                  | Encrypted relational store                     |
| Background Tasks          | `react-native-background-fetch`, `react-native-upload`     | Reliable cross-OS scheduling                   |
| Networking                | Axios + interceptors                                       | Retry/back-off, JWT injection                  |
| Backend                   | FastAPI, SQLAlchemy                                        | Async, type safety                             |
| Analytics/Stats           | Pandas, Statsmodels, Prophet, Plotly                       | Mature statistical libraries                   |
| Storage & Auth            | Supabase Postgres, Storage, Auth                           | Free tier, RLS, presigned uploads              |
| Task Queue                | Celery + Redis                                             | Asynchronous AI calls & nightly jobs           |
| AI APIs                   | Infermedica, Google Vision, Google Speech-to-Text          | Pre-trained, pay-as-you-go                     |
| Email / PDF               | SendGrid, WeasyPrint PDF                                   | Automated reporting                            |
| Deployment                | Fly.io (backend) + Supabase (DB/Storage)                   | Low-ops student-friendly                       |
| CI/CD                     | GitHub Actions (backend) • Codemagic (React Native)        | Automated tests & builds                       |

---

## 7  Data Model  

```sql
-- patients
CREATE TABLE patients (
  id          uuid PRIMARY KEY,
  sex         varchar,
  dob         date,
  created_at  timestamptz DEFAULT now()
);

-- visits
CREATE TABLE visits (
  id          uuid PRIMARY KEY,
  patient_id  uuid REFERENCES patients,
  symptoms    jsonb,
  vitals      jsonb,
  photo_url   text,
  audio_url   text,
  gps_point   geography(Point, 4326),
  ai_jsonb    jsonb,
  created_at  timestamptz DEFAULT now()
);

-- daily statistics
CREATE TABLE stats_daily (
  clinic_id   uuid,
  date        date,
  total       int,
  fever       int,
  diarrhoea   int,
  z_fever     numeric,
  forecast_total numeric,
  PRIMARY KEY (clinic_id, date)
);
```

---

## 8  Security & Compliance  

| Threat Vector                 | Control Mechanisms                                          |
|-------------------------------|-------------------------------------------------------------|
| Device loss                   | SQLCipher encryption, biometric lock, optional remote wipe |
| Network interception          | TLS 1.3, certificate pinning, short-lived JWTs             |
| API key leakage               | `.env` files ignored in repo, secrets in CI env vars       |
| PHI exposure to AI vendors    | De-identification, minimal field sharing                   |
| Data privacy regulations      | Supabase RLS; future BAA/ISO audit for production          |

---

## 10  Risk Register  

| Risk                                  | Likelihood | Impact | Mitigation                                        |
|---------------------------------------|-----------|--------|---------------------------------------------------|
| iOS background task throttling        | Medium    | Medium | Use BGProcessingTask, user education              |
| AI quota overrun                      | Medium    | Medium | Quota monitor, cache, exponential back-off        |
| Store rejection for medical claims    | Low       | High   | Prominent disclaimer, categorise as “reference”   |
| Data sync conflicts                   | Low       | Medium | Last-write-wins, manual reconciliation queue      |
| Storage space on old devices          | Medium    | Low    | Compress media, limit audio duration              |

---

## 11  Alignment with UN SDGs  

| SDG | Contribution                                                                              |
|-----|-------------------------------------------------------------------------------------------|
| 3.8 | Promotes universal health coverage via offline data capture & triage.                     |
| 9.c | Demonstrates resilient ICT solutions for low-bandwidth environments.                      |
| 6   | Early diarrhoea spike detection supports timely WASH interventions.                       |

---

## 12  Conclusion  

VitalSync demonstrates how carefully-chosen, production-ready open-source tools—**React Native**, **FastAPI**, **Supabase**, and **pre-trained AI APIs**—can be combined to deliver a fully offline-capable, data-driven telehealth solution. The project emphasises:

1. **Field practicality**: Works with or without network, on affordable Android or iOS devices.  
2. **Evidence-based insights**: In-app analytics give CHWs and supervisors immediate situational awareness.  
3. **Scalability & extensibility**: The architecture supports future web dashboards, on-device ML, or additional health modules without a major rewrite.  
4. **Security & compliance**: Encryption, JWT, and Supabase RLS form a solid baseline for protecting sensitive health data.

By equipping frontline workers with reliable tools and actionable information, VitalSync advances patient outcomes and supports Sustainable Development Goals focused on good health, resilient infrastructure, and clean water/sanitation interventions.
---

