# TeleCare Lite

## 1. Overview

**TeleCare Lite** is an offline-first, cross-platform (iOS & Android) mobile application designed to empower community health workers (CHWs) in low-connectivity areas. The application enables health workers to:

- Capture patient data (symptoms, vitals, photos, audio) entirely offline.
- Sync data securely to a FastAPI backend with Supabase integration once connectivity returns.
- Receive AI-generated symptom advice using pre-trained APIs (Infermedica, Google Vision, Google Speech-to-Text).
- Provide clinicians and NGOs with automated analytics, trend detection, and alerts.

The system is built with **Flutter** for the mobile app, **FastAPI** for backend services, and **Supabase** for authentication, storage, and database management.

---

## 2. Project Structure
```
telecare-lite/
├── mobile-app/       # Flutter app (iOS & Android)
├── backend/          # FastAPI + Celery backend
├── supabase/         # Database migrations, policies
├── .github/          # GitHub Actions workflows
└── docs/             # Optional documentation (e.g., architecture diagrams)
```

---

## 3. Core Functionalities and Architecture

### 3.1 Visit Capture and Offline Storage
- **File**: `mobile-app/lib/screens/visit_form.dart`
- **Purpose**: Capture patient data (name, age, sex, symptoms, vitals, photos, audio) offline.
- **Details**: 
  - Uses **sqflite** with **SQLCipher** for encrypted local storage.
  - Data saved to an "outbox" queue in SQLite.
  - Form input validation included.

### 3.2 Background Sync Service
- **File**: `mobile-app/lib/services/sync_service.dart`
- **Purpose**: Syncs queued data when network becomes available.
- **Details**:
  - `background_fetch` plugin for background tasks.
  - `dio` client for HTTP requests with retries.
  - Uploads media via presigned URLs to Supabase.
  - Posts visit data to `/api/visits`.

### 3.3 Backend API
- **File**: `backend/app/main.py`, `backend/app/routers/visits.py`
- **Purpose**: REST API for visit data, media uploads, and stats retrieval.
- **Details**:
  - FastAPI with Pydantic models.
  - JWT authentication via Supabase.
  - Supabase Storage for media files.

### 3.4 AI-Powered Analysis
- **File**: `backend/app/tasks/ai_tasks.py`
- **Purpose**: Enhance visit records with AI-generated advice and media analysis.
- **Details**:
  - Celery workers call Infermedica, Google Vision, and Speech-to-Text APIs.
  - Results stored in `visits.ai_jsonb`.

### 3.5 Analytics and Alerts
- **File**: `backend/app/tasks/stats.py`
- **Purpose**: Nightly analysis and alert emails.
- **Details**:
  - Aggregates data with Pandas.
  - Calculates z-scores for trend detection.
  - Sends alerts via email with Matplotlib-generated charts.

### 3.6 Authentication and Security
- **File**: `mobile-app/lib/services/auth_service.dart`
- **Purpose**: Secure user authentication and token management.
- **Security**:
  - Encrypted local data.
  - JWTs for API calls.
  - Supabase RLS policies.

---

## 4. Setup Instructions

### 4.1 Prerequisites
- Flutter 3.x
- Python 3.11+
- Supabase account
- Redis for Celery
- Docker (optional for backend)
- Xcode for iOS, Android Studio for Android

### 4.2 Backend Setup
```
cd telecare-lite/backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # Edit with keys
redis-server          # Start if needed
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
celery -A app.tasks.ai_tasks worker --loglevel=info
celery -A app.tasks.stats beat --loglevel=info
```

### 4.3 Mobile App Setup
```
cd telecare-lite/mobile-app
flutter pub get
flutter run -d ios        # iOS Simulator (Mac)
flutter run -d android    # Android Emulator
flutter build apk         # Android release
flutter build ios         # iOS release (requires Xcode)
```

### 4.4 Supabase Setup
- Create a project at [Supabase](https://app.supabase.com)
- Configure tables `patients`, `visits`, `stats_daily` (use `supabase/seed.sql`)
- Apply RLS policies (`supabase/policies.sql`)
- Create a `media` bucket for uploads
- Obtain your Supabase URL, anon key, and service key

### 4.5 Environment Variables (`backend/.env`)
```
SUPABASE_URL=your_supabase_project_url
SUPABASE_SERVICE_KEY=your_service_key
JWT_SECRET=your_jwt_secret
REDIS_URL=redis://localhost:6379
INFERMEDICA_APP_ID=your_infermedica_id
INFERMEDICA_APP_KEY=your_infermedica_key
GOOGLE_CLOUD_VISION_KEY=your_google_vision_api_key
SENDGRID_API_KEY=your_sendgrid_api_key
EMAIL_FROM=alerts@telecarelite.com
EMAIL_TO=admin@telecarelite.com
```

---

## 5. Testing and CI
```
# Backend tests
cd backend
pytest

# Flutter tests
cd ../mobile-app
flutter test

# Flutter linting
flutter analyze
```
CI/CD configured in `.github/workflows/`.

---

## 6. Deployment
- **Backend**: Deploy via Fly.io, Render, Railway, or Docker.
- **Mobile App**: Use Codemagic/Bitrise for automated builds.
- **Prepare for App Store Connect and Google Play submission.**

---

## 7. Security Considerations
- Use `.env` for sensitive keys (do not commit them).
- Properly configure iOS and Android permissions.
- Enforce Supabase RLS policies.
- AI-generated advice is guidance only, not a medical diagnosis.

---

## 8. Contributing
Contributions are welcome. Fork the repository, create a feature branch, and submit a pull request. Report issues via GitHub. A `CONTRIBUTING.md` will be added soon.

---

## 9. Legal Disclaimer
TeleCare Lite is not a licensed medical device. It provides AI-generated health insights for educational purposes. Always consult a licensed healthcare provider for medical decisions.

