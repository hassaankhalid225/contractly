# Contractly

> Your freelance contracts, always in control.

A premium contract management app for freelancers and small agencies.
Generate contracts with AI, upload existing ones for AI risk analysis,
sign digitally, track payments, and get smart reminders — all in one place.

This repository contains two production-ready apps:

- **`Backend/`** — FastAPI + MongoDB API (auth, contracts, payments, AI, files)
- **`Frontend/`** — Flutter mobile app (Android & iOS) using Riverpod + GoRouter

## Highlights

- **iPhone-style frosted glass UI** — every card uses backdrop blur for depth
- **Brand palette** — accent rose (`#E94560`) on deep indigo backgrounds, never blue
- **Inter typography** throughout (via `google_fonts`)
- **Centralized API layer** with silent JWT refresh & friendly error mapping
- **AI-powered** contract generation, analysis and template suggestion
- **E-signature canvas** with PNG upload to Cloudinary
- **Local push reminders** scheduled 14/7/1 days before contract expiry
- **Multi-currency** (USD, PKR, AED, GBP, EUR, INR, CAD, AUD)

## Authentication

Three sign-in paths are wired end-to-end:

1. **Google Sign-In** — client sends Google ID token, server verifies with `google-auth`
2. **Phone OTP** — client requests OTP, server stores it in MongoDB with a 5-minute TTL,
   verification returns a JWT pair
3. **Test phone bypass** — `+923001234567` and `+923009876543` always succeed with OTP `123456`

JWT pair = 15-min access token + 30-day refresh token. The Flutter
interceptor refreshes silently on 401s and only redirects to the auth
screen if refresh itself fails.

## Quick Start

### Backend

```powershell
cd Backend
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env       # then fill in values
uvicorn main:app --reload
```

API docs at <http://localhost:8000/docs>.

### Frontend

```powershell
cd Frontend
flutter pub get
flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8000/api/v1
```

> `10.0.2.2` is the Android emulator's loopback to your host machine.
> On a real device or iOS simulator, replace it with your machine's LAN IP.

## Project Layout

```
Contractly/
├── Backend/
│   ├── main.py
│   ├── requirements.txt
│   ├── .env.example
│   └── app/
│       ├── core/         (config, database, security, dependencies, responses)
│       ├── models/       (user, contract, payment, token)
│       ├── routers/      (auth, users, contracts, payments, ai, notifications)
│       ├── services/     (auth, contract, ai, storage, notification)
│       └── middleware/   (cors, error_handler)
└── Frontend/
    ├── pubspec.yaml
    └── lib/
        ├── main.dart
        ├── core/
        │   ├── constants/   (colors, text styles, spacing, app constants)
        │   ├── network/     (api_client, endpoints, response, interceptor, exceptions)
        │   ├── services/    (storage, auth, file, notification)
        │   ├── models/      (user, contract, payment, auth_token)
        │   ├── providers/   (auth, contract, payment, notifications)
        │   └── router/      (app_router, route_guards)
        ├── ui/
        │   ├── components/  (glass_card, primary/secondary buttons, status_badge,
        │   │                 contract_tile, section_header, loading_overlay,
        │   │                 error_snackbar, empty_state, app_bottom_nav,
        │   │                 app_background)
        │   └── screens/     (splash, onboarding, auth/{auth, phone_input, otp},
        │                     home, contracts/{list, detail, create, upload, ai_analysis},
        │                     sign, payments, profile, notifications)
        └── util/
            ├── helpers/     (date, currency, string, validator)
            ├── extensions/  (context, string, date)
            └── mixins/      (loading)
```

## Functional Checklist

Everything below is wired end-to-end:

- [x] Google Sign-In → JWT received → user logged in → home visible
- [x] Phone OTP flow → `+923001234567` + `123456` → logged in
- [x] AI contract generation (with deterministic local fallback)
- [x] Manual contract creation with date pickers and validation
- [x] PDF upload to Cloudinary, URL stored on the contract
- [x] AI contract analysis (risk level, risky clauses, recommendations)
- [x] E-signature canvas → PNG upload → contract marked active
- [x] Payment milestones (CRUD + summary)
- [x] Mark payment as paid via bottom-sheet confirmation
- [x] Contract progress bar based on start/end dates
- [x] Real-time contract search with debounced API calls
- [x] Status filter chips (all / active / expiring / expired / draft / completed)
- [x] Local notifications scheduled at 14/7/1 days before expiry
- [x] Pull-to-refresh on home, contracts, payments
- [x] Token refresh runs silently inside `ApiInterceptor`
- [x] Logout / delete-account flows clear local + server state
- [x] Profile editing (name) persisted to backend
- [x] Friendly error snackbars — no raw network errors leak to users
- [x] Splash → onboarding (first launch) → auth flow guarded by `route_guards.dart`

## Standard Response Envelope

Both success and error responses follow the same shape:

```jsonc
{ "success": true,  "data": { /* ... */ }, "message": "Optional success message." }
{ "success": false, "error": { "code": "CONTRACT_NOT_FOUND", "message": "...", "details": null } }
```

The Flutter `ApiClient` parses both shapes automatically and surfaces a
typed `NetworkException` with a user-friendly message via
`NetworkException.friendlyMessage`.

## Configuration Notes

| Concern        | Required env keys                                                  |
|----------------|--------------------------------------------------------------------|
| MongoDB        | `MONGODB_URL`, `MONGODB_DB_NAME`                                   |
| JWT            | `SECRET_KEY` (≥16 chars), `ALGORITHM`, `ACCESS_TOKEN_EXPIRE_MINUTES`, `REFRESH_TOKEN_EXPIRE_DAYS` |
| Google Sign-In | `GOOGLE_CLIENT_ID`                                                 |
| AI features    | `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL`                             |
| File storage   | `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` |
| SMS OTP (prod) | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER`    |

If `ANTHROPIC_API_KEY` is missing the API falls back to a
deterministic local generator/analyzer so the app still works offline
during development.

If `TWILIO_*` is unset the OTP is logged to the server console (still
verifiable through the regular `/auth/phone/verify` endpoint).
