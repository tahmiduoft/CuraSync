# CuraSync

A cross-platform (iOS & Android) **React Native** application that lets community-health workers (CHWs) collect patient data completely offline, receive AI-powered symptom advice, and view interactive analytics in-app. A FastAPI + Supabase backend handles syncing, AI enrichment, statistical analysis, and automated alert e-mails.

---

## 1. Key Features

| Category             | Highlights                                                                                                    |
|----------------------|----------------------------------------------------------------------------------------------------------------|
| **Offline workflow** | Encrypted SQLite queue stores visits, photos, and audio when no signal is available.                          |
| **Background sync**  | react-native-background-fetch uploads queued items via presigned Supabase URLs once connectivity returns.    |
| **AI advice**        | Pre-trained APIs (Infermedica, Google Vision, Google Speech-to-Text). *No model training required.*            |
| **In-app analytics** | Descriptive stats, line-trend charts, heat-maps, and Prophet forecasts rendered with Victory Native + SVG.     |
| **Automated alerts** | Nightly Pandas job detects z-score spikes and e-mails a PDF brief to supervisors.                              |
| **Security**         | SQLCipher on device, JWT auth, HTTPS/TLS 1.3, Supabase Row-Level Security (RLS).                               |

---

## 2. Tech Stack

| Layer / Purpose      | Technology / Library                                          |
|----------------------|---------------------------------------------------------------|
| Mobile framework     | React Native 0.73 (TypeScript)                                |
| Local database       | react-native-sqlite-storage + SQLCipher                     |
| Background tasks     | react-native-background-fetch, react-native-upload        |
| Charts & maps        | Victory Native, react-native-svg, Mapbox GL                |
| Networking           | Axios with retry & JWT interceptor                            |
| Authentication       | Supabase Auth (@supabase/supabase-js)                       |
| Backend API          | FastAPI (Python 3.13)                                         |
| Queue & jobs         | Celery 5 + Redis                                              |
| Database & storage   | Supabase Postgres, Supabase Storage                           |
| Analytics            | Pandas, Statsmodels, Prophet, Plotly (server-side PNG/SVG)    |
| AI APIs              | Infermedica, Google Vision, Google Speech-to-Text             |
| Email & PDF          | SendGrid, WeasyPrint                                          |
| CI/CD                | GitHub Actions (API) · Codemagic (mobile builds)              |
| Deployment (API)     | Fly.io                                                        |

---

## 3. 📁 Project Structure

```
vitalsync/
├── README.md
├── DEVLOG.md
├── BUGLOG.md
├── .gitignore
├── .env                        # (never commit secrets!)

├── mobile-app/                 # React Native iOS + Android project
│   ├── package.json
│   ├── tsconfig.json
│   ├── metro.config.js
│   ├── babel.config.js
│   ├── android/
│   │   └── app/
│   │       └── src/main/
│   │           ├── AndroidManifest.xml
│   │           └── java/... (MainActivity.java / MainApplication.java)
│   ├── ios/
│   │   ├── VitalSync.xcodeproj/
│   │   ├── VitalSync.xcworkspace/
│   │   └── VitalSync/Info.plist
│   ├── app/                    # source-code root
│   │   ├── App.tsx
│   │   ├── navigation/
│   │   │   └── RootNavigator.tsx
│   │   ├── screens/
│   │   │   ├── VisitForm.tsx
│   │   │   ├── AnalyticsScreen.tsx
│   │   │   └── SyncStatus.tsx
│   │   ├── services/
│   │   │   ├── authService.ts
│   │   │   ├── dbService.ts
│   │   │   ├── syncService.ts
│   │   │   └── aiService.ts
│   │   ├── database/
│   │   │   ├── schema.sql      # initial SQLCipher schema
│   │   │   └── migrations/
│   │   ├── models/
│   │   │   ├── Patient.ts
│   │   │   └── Visit.ts
│   │   ├── components/
│   │   │   └── CustomInput.tsx
│   │   ├── utils/
│   │   │   ├── constants.ts
│   │   │   └── validators.ts
│   │   └── assets/
│   │       ├── images/
│   │       └── localization/
│   └── __tests__/
│       ├── VisitForm.test.tsx
│       └── syncService.test.ts

├── backend/                    # FastAPI + Celery backend
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── .env
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── database.py
│   │   ├── auth.py
│   │   ├── routers/
│   │   │   ├── visits.py
│   │   │   ├── stats.py
│   │   │   └── charts.py
│   │   ├── tasks/
│   │   │   ├── ai_tasks.py
│   │   │   ├── stats.py
│   │   │   └── email_alert.py
│   │   └── utils/
│   │       └── helpers.py
│   ├── celery_worker.py
│   └── tests/
│       ├── test_visits.py
│       └── test_ai_tasks.py

├── .github/
│   └── workflows/
│       ├── backend_ci.yml
│       └── mobile_ci.yml       # Codemagic/Bitrise trigger script

├── supabase/                   # Configurations for Supabase
│   ├── migrations/
│   ├── policies.sql
│   └── seed.sql

└── docs/                       # Optional: design & specs
    ├── architecture_diagram.png
    └── ui_mockups/
        ├── visit_form_wireframe.png
        └── analytics_wireframe.png

```

---

## 4. Setup

### 4.1 Backend

```bash
cd CuraSync/backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env      # add your keys
redis-server &            # or Docker Redis
uvicorn app.main:app --reload --port 8000 &
celery -A app.tasks.ai_tasks worker --loglevel=info &
celery -A app.tasks.stats beat --loglevel=info &
```

### 4.2 Mobile App

```bash
cd CuraSync/mobile-app
yarn install
npx pod-install            # iOS only
yarn ios       # or:  yarn android
```

Add Supabase URL and anon key in `app/env.ts`.

### 4.3 Supabase Init

1. Create project at https://app.supabase.com
2. Run SQL in `supabase/seed.sql`; enable RLS with `supabase/policies.sql`
3. Create private media bucket for uploads
4. Copy project URL, anon key, service key into `.env` and `env.ts`

---

## 5. Usage Flow

1. Launch app offline → fill Visit Form → tap **Save**.
2. When back online, background sync auto-uploads (or tap **Sync Now**).
3. Open **Analytics** tab:
   - Descriptive cards (totals, averages)
   - Trend line for fever %
   - Heat-map for symptom clusters (if GPS data exists)
   - Forecast card (next 7-day projection)
   - Red banner if z-score alert triggered
4. Supervisors receive daily PDF e-mail with charts & KPIs.

---

## 6. Environment Variables

Create `backend/.env`:

```ini
SUPABASE_URL=...
SUPABASE_SERVICE_KEY=...
JWT_SECRET=...
REDIS_URL=redis://localhost:6379
INFERMEDICA_APP_ID=...
INFERMEDICA_APP_KEY=...
GOOGLE_CLOUD_VISION_KEY=...
SENDGRID_API_KEY=...
EMAIL_FROM=alerts@CuraSync.app
EMAIL_TO=admin@CuraSync.app
```

---

## 7. Security Notes

- SQLCipher encrypts on-device DB; biometric lock optional.
- JWT stored in Keychain / Keystore via `react-native-keychain`.
- TLS 1.3 with certificate pinning (Axios).
- Only minimal, de-identified fields sent to AI APIs.

---

## 🤝 Contributing

1. Fork → branch (`feat/my-feature`) → commit.
2. Run `yarn test` (mobile) and `pytest` (backend) before PR.
3. Submit pull request; CI must pass.

---

## 📄 License

MIT — see [LICENSE](LICENSE).

---

## ⚠️ Disclaimer

CuraSync provides AI-generated health insights. It is **not** a licensed medical device and does not replace professional medical advice. Always consult qualified clinicians.
