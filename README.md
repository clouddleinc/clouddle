# Clouddle — Phase 1: Omada Foundation

## What's in this repo

```
clouddle/
├── services/
│   ├── network-adapter-service/   # Go — talks to Omada API
│   │   ├── cmd/server/main.go     # HTTP server + handlers
│   │   ├── internal/
│   │   │   ├── adapter/           # NetworkController interface + config
│   │   │   └── omada/             # OmadaAdapter implementation
│   │   └── Dockerfile
│   └── resident-service/
│       └── cmd/server/
│           └── device_poller.go   # Firestore polling worker
├── infra/
│   ├── migrations/
│   │   └── 001_initial_schema.sql # AlloyDB schema
│   └── terraform/
│       └── main.tf                # All GCP resources
├── flutter/clouddle_app/          # iOS + Android resident app
│   └── lib/
│       ├── main.dart              # App entry, router, theme
│       ├── services/auth_service.dart
│       └── screens/
│           ├── auth/login_screen.dart
│           └── dashboard/devices_screen.dart
├── portal/src/                    # React property manager portal
│   ├── App.jsx
│   └── App.css
├── docker-compose.yml             # Full local dev stack
└── .env.example                   # Copy to .env, fill in credentials
```

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Go | 1.22+ | Backend services |
| Flutter | 3.19+ | iOS + Android app |
| Node.js | 20+ | React portal |
| Docker Desktop | Latest | Local dev stack |
| Terraform | 1.7+ | GCP infrastructure |
| `gcloud` CLI | Latest | GCP deployment |

---

## Day 1: Local development setup

### 1. Clone and configure

```bash
git clone https://github.com/your-org/clouddle.git
cd clouddle
cp .env.example .env
# Edit .env with your Omada controller credentials
```

### 2. Find your Omada credentials

You need four values from your Omada controller before anything works:

**OMADA_CONTROLLER_ID**
Log into your Omada web UI → Settings → Controller → copy the "Controller ID" shown on the page. It looks like `omadac_xxxxxxxxxxxxxxxx`.

**OMADA_SITE_ID**
Settings → Sites → click your site → the ID is in the browser URL: `.../sites/{SITE_ID}/...`
For single-site setups without MSP mode, this is usually `"default"`.

**OMADA_CLIENT_ID + OMADA_CLIENT_SECRET**
Settings → Open API → Create application → copy Client ID and Client Secret.
Note: Open API must be enabled on your controller. In Omada Cloud this is under your account settings.

**OMADA_PPSK_PROFILE_ID**
Wireless → PPSK → click your PPSK profile → copy the ID from the browser URL.
If you haven't created a PPSK profile yet: Wireless → SSID → edit your resident SSID → enable PPSK → create a new profile.

### 3. Start the local stack

```bash
docker compose up
```

This starts:
- `network-adapter-service` on port 8081
- `resident-service` on port 8080
- Firestore emulator on port 8200
- Pub/Sub emulator on port 8085
- Postgres (AlloyDB substitute) on port 5432 — migrations run automatically
- React portal on port 3000

### 4. Verify the Omada connection

```bash
curl http://localhost:8081/healthz
# Should return: ok
# If it returns 503, check your .env credentials and controller URL
```

### 5. Test resident provisioning end-to-end

```bash
# Provision a test resident
curl -X POST http://localhost:8081/v1/residents/provision \
  -H "Content-Type: application/json" \
  -d '{
    "residentId": "test-resident-001",
    "unitNumber": "402",
    "vlanId": 402,
    "residentName": "Test Resident",
    "email": "test@example.com"
  }'

# Expected response:
# {
#   "PPSKEntryID": "omada-internal-id",
#   "PPSKValue": "xk9m-7vbq-3npw-r2zt",
#   "VlanID": 402,
#   "ProfileID": "your-ppsk-profile-id"
# }
```

Check your Omada controller — you should see VLAN 402 created under Networks and a new PPSK entry under Wireless → PPSK.

### 6. Open the React portal

Visit http://localhost:3000 and sign in with Google (requires Firebase Auth to be configured — see Firebase setup below).

---

## Firebase setup (required for auth)

1. Go to https://console.firebase.google.com
2. Create a new project: "clouddle-dev"
3. Add a web app → copy the config values into `.env`
4. Authentication → Sign-in method → enable "Email/Link (passwordless)" and "Google"
5. Add your local domain to authorized domains: `localhost`
6. Firestore → Create database → Start in test mode (for development only)

For the Flutter app:
```bash
cd flutter/clouddle_app
flutter pub get
# Download google-services.json from Firebase console → Android app
# Download GoogleService-Info.plist → iOS app
# Place them in the correct platform directories
flutter run
```

---

## GCP deployment (staging)

### One-time setup

```bash
# Authenticate
gcloud auth login
gcloud auth application-default login

# Create the GCP project
gcloud projects create clouddle-staging
gcloud config set project clouddle-staging
gcloud billing projects link clouddle-staging --billing-account=YOUR_BILLING_ID

# Create Terraform state bucket
gsutil mb gs://clouddle-tf-state
gsutil versioning set on gs://clouddle-tf-state
```

### Deploy infrastructure

```bash
cd infra/terraform
terraform init
terraform plan -var="project_id=clouddle-staging"
terraform apply -var="project_id=clouddle-staging"
```

### Store Omada secret

```bash
# Add the Omada client secret to Secret Manager
echo -n "your-omada-client-secret" | \
  gcloud secrets versions add omada-client-secret --data-file=-
```

### Build and deploy services

```bash
# Configure Docker for Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev

# Build and push network-adapter-service
cd services/network-adapter-service
docker build -t us-central1-docker.pkg.dev/clouddle-staging/clouddle/network-adapter-service:latest .
docker push us-central1-docker.pkg.dev/clouddle-staging/clouddle/network-adapter-service:latest

# Deploy to Cloud Run (Terraform manages the service definition;
# this updates the image for the existing service)
gcloud run services update network-adapter-service \
  --image us-central1-docker.pkg.dev/clouddle-staging/clouddle/network-adapter-service:latest \
  --region us-central1

# Repeat for resident-service
```

### Run AlloyDB migrations

```bash
# Connect via Cloud SQL Auth Proxy or from a Cloud Run job
psql $ALLOYDB_CONNECTION_STRING -f infra/migrations/001_initial_schema.sql
```

---

## Environment variables reference

| Variable | Service | Description |
|---|---|---|
| `OMADA_BASE_URL` | network-adapter | Controller URL e.g. `https://192.168.1.100:8043` |
| `OMADA_CONTROLLER_ID` | network-adapter | Omada controller ID (omadac_xxx) |
| `OMADA_SITE_ID` | network-adapter | MSP site ID or "default" |
| `OMADA_CLIENT_ID` | network-adapter | OAuth2 client ID |
| `OMADA_SECRET_REF` | network-adapter | Secret Manager path (prod only) |
| `OMADA_CLIENT_SECRET` | network-adapter | Secret value (local dev only) |
| `OMADA_PPSK_PROFILE_ID` | network-adapter | Omada PPSK profile ID |
| `OMADA_MODE` | network-adapter | "local" or "cloud" |
| `GCP_PROJECT_ID` | all services | GCP project ID |
| `NETWORK_ADAPTER_URL` | resident-service | Internal URL of network-adapter-service |
| `ALLOYDB_URL` | resident-service | Postgres connection string |
| `FIRESTORE_EMULATOR_HOST` | all (local only) | Firestore emulator address |

---

## What's NOT in Phase 1

These are explicitly deferred to Phase 2 and 3:

- **Entrata webhook integration** — manual onboarding only in Phase 1
- **Support tickets** — UI placeholder only
- **Guest key UI** — backend endpoints exist, Flutter UI in Phase 2
- **Roommate sharing UI** — backend endpoints exist, Flutter UI in Phase 2
- **Property communications** — Pub/Sub topic exists, comms-service in Phase 2
- **BigQuery analytics** — wired up in Phase 3
- **Ruckus adapter** — Phase 4

---

## Key architectural decisions

**Why Go for backend services?**
Stateless, compiles to a single binary, tiny Docker images (distroless = ~10MB), excellent concurrency for the polling worker, and Cloud Run's cold starts are measured in milliseconds not seconds.

**Why Flutter for mobile?**
One codebase for iOS + Android. Firestore's Flutter SDK has first-class real-time streaming support — `StreamBuilder` directly on a Firestore document means device list updates happen without any polling from the app side.

**Why the NetworkController interface now?**
The OmadaAdapter implements it today. When Ruckus is added in Phase 4, a `RuckusAdapter` struct is added, the factory function gets one new `case`, and zero other files change. The interface is the contract that makes this possible.

**Why AlloyDB + Firestore instead of just one database?**
Firestore: real-time sync to mobile, simple document reads, free at low scale.
AlloyDB: SQL date queries for lease scheduling (`lease_end BETWEEN ... AND ...`), relational integrity for the lease→resident relationship, audit history.
They serve different access patterns. Trying to do lease scheduling in Firestore is painful; trying to stream real-time device updates through Postgres is unnecessary complexity.

**Why respond-immediately + Cloud Tasks for webhooks?**
Entrata retries webhooks if you don't respond within ~5 seconds. Omada API calls can take 1-3 seconds each. Creating a VLAN + PPSK + writing Firestore + sending email = easily 8-15 seconds. Cloud Tasks decouples "receiving the event" from "doing the work" and retries automatically if Omada is temporarily unavailable.

---

## Testing

```bash
# Unit tests for the OmadaAdapter (requires a test double for the HTTP client)
cd services/network-adapter-service
go test ./...

# Integration test against a real Omada controller
OMADA_BASE_URL=https://... go test ./... -tags=integration

# Flutter tests
cd flutter/clouddle_app
flutter test

# Portal tests
cd portal
npm test
```

---

## Phase 2 checklist (next sprint)

- [ ] Flutter: Guest key creation screen + QR display
- [ ] Flutter: Roommate sharing flow
- [ ] Flutter: Support ticket submission
- [ ] resident-service: Full CRUD with Firebase JWT validation
- [ ] comms-service: Broadcast endpoint + FCM + SendGrid
- [ ] entrata-sync-service: Webhook receiver + idempotency + Cloud Tasks
- [ ] AlloyDB: Migration 002 — add entrata_events, guest_keys, device_shares tables
- [ ] Terraform: Add entrata-sync-service Cloud Run + webhook URL
- [ ] App Store Connect + Google Play Console setup
- [ ] Fastlane configuration for CI/CD submissions
