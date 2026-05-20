# Dopamine Reset — Tech Stack & Getting Started Guide

> Full-stack development reference for the Dopamine Reset app: languages, frameworks, services, and step-by-step setup for every layer of the system.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Languages Used](#languages-used)
3. [Frontend (Mobile App)](#frontend-mobile-app)
4. [Frontend (Partner Companion Web App)](#frontend-partner-companion-web-app)
5. [Backend & API](#backend--api)
6. [Database Layer](#database-layer)
7. [Real-Time & Notifications](#real-time--notifications)
8. [DNS-Layer Blocking Service](#dns-layer-blocking-service)
9. [Browser Extension](#browser-extension)
10. [Infrastructure & DevOps](#infrastructure--devops)
11. [Security & Encryption](#security--encryption)
12. [AI / Behavioral Intelligence Layer](#ai--behavioral-intelligence-layer)
13. [Full Dependency Map](#full-dependency-map)
14. [Getting Started — Step by Step](#getting-started--step-by-step)
15. [Project Folder Structure](#project-folder-structure)
16. [Environment Variables Reference](#environment-variables-reference)
17. [Development Roadmap](#development-roadmap)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                         │
│                                                             │
│   iOS App (Swift/SwiftUI)   Android App (Kotlin/Compose)   │
│   Partner Web App (React + TypeScript)                      │
│   Browser Extension (TypeScript + WebExtensions API)        │
└──────────────────┬──────────────────────────────────────────┘
                   │  HTTPS / WebSocket
┌──────────────────▼──────────────────────────────────────────┐
│                         API LAYER                           │
│                                                             │
│   Node.js + TypeScript (Express / Fastify)                  │
│   REST endpoints + WebSocket server                         │
│   Auth: JWT + OAuth 2.0                                     │
└──────┬──────────────────────────┬───────────────────────────┘
       │                          │
┌──────▼──────┐          ┌────────▼────────┐
│  PostgreSQL  │          │     Redis        │
│  (primary   │          │  (sessions,      │
│   storage)  │          │   queues,        │
└─────────────┘          │   rate limits)   │
                         └─────────────────┘
┌─────────────────────────────────────────────────────────────┐
│                    SERVICES LAYER                           │
│                                                             │
│   Firebase Cloud Messaging  (push notifications)           │
│   SendGrid                  (email digests)                 │
│   Twilio                    (SMS fallback alerts)           │
│   NextDNS / ControlD API    (DNS-layer blocking)            │
│   Anthropic API             (behavioral nudge copy)         │
│   AWS S3                    (why-video storage)             │
└─────────────────────────────────────────────────────────────┘
```

---

## Languages Used

| Layer | Primary Language | Secondary |
|---|---|---|
| iOS app | Swift | — |
| Android app | Kotlin | — |
| Partner web app | TypeScript (React) | CSS |
| Browser extension | TypeScript | CSS |
| API server | TypeScript (Node.js) | SQL |
| Database queries | SQL (PostgreSQL dialect) | — |
| Infrastructure / IaC | HCL (Terraform) | YAML |
| CI/CD pipelines | YAML (GitHub Actions) | Bash |
| Scripts & tooling | Bash | Python |
| DNS blocking config | JSON (NextDNS API) | — |

### Why These Languages

**Swift + Kotlin** — Native languages for iOS and Android respectively. System-level access is mandatory for the friction engine (app blocking, screen time integration, usage detection). React Native or Flutter cannot reliably hook into `ScreenTime API` (iOS) or `UsageStatsManager` (Android) at the depth this app requires.

**TypeScript everywhere else** — Single language for backend, web app, and extension. Shared type definitions between layers. Eliminates runtime type errors in the accountability and vault code paths where bugs have real user consequences.

**SQL (PostgreSQL)** — The audit log, bypass history, and streak records are the heart of the accountability system. They must be immutable, queryable, and impossible to silently corrupt. An ORM-only approach is insufficient for the audit trail queries.

**HCL (Terraform)** — All infrastructure defined as code. The app's vault system and blocking services need reproducible, auditable infrastructure. No clickops.

---

## Frontend (Mobile App)

### iOS — Swift + SwiftUI

```
Minimum iOS version: iOS 16.0
Xcode version:       15+
Swift version:       5.9+
```

**Key frameworks:**

| Framework | Purpose |
|---|---|
| `SwiftUI` | All UI screens |
| `FamilyControls` | Screen Time API integration (requires Apple entitlement) |
| `ManagedSettings` | App blocking enforcement on iOS |
| `DeviceActivity` | Usage monitoring and session cap detection |
| `UserNotifications` | Local notifications for friction prompts |
| `AVFoundation` | Why-video recording in onboarding |
| `CryptoKit` | Local encryption of vault code |
| `Combine` | Reactive data binding |

**Package manager:** Swift Package Manager (SPM)

**Notable dependency:** `FamilyControls` requires a special Apple entitlement (`com.apple.developer.family-controls`). Apply via the Apple Developer portal under App Services. Without it, the app cannot enforce restrictions. This is the most critical setup step on iOS.

---

### Android — Kotlin + Jetpack Compose

```
Minimum Android version: Android 10 (API 29)
Kotlin version:          1.9+
Gradle version:          8.0+
```

**Key APIs and libraries:**

| Library / API | Purpose |
|---|---|
| `Jetpack Compose` | All UI screens |
| `UsageStatsManager` | App usage monitoring |
| `AppOpsManager` | Permission to read usage stats |
| `DevicePolicyManager` | Enforce app restrictions (MDM-lite approach) |
| `WorkManager` | Background tasks (digest generation, silent-period detection) |
| `Room` | Local SQLite database for offline streak data |
| `Retrofit + OkHttp` | API communication |
| `Firebase SDK` | Push notifications |
| `CameraX` | Why-video recording in onboarding |
| `Hilt` | Dependency injection |

**Package manager:** Gradle + Maven Central

**Notable dependency:** `UsageStatsManager` requires the `PACKAGE_USAGE_STATS` permission, which users must grant manually in Settings > Special App Access. The onboarding flow must guide users through this.

---

## Frontend (Partner Companion Web App)

### React + TypeScript + Vite

The partner companion app is web-first (accessible from any device without install) and built as a Progressive Web App (PWA) for optional home-screen installation.

```bash
# Create the project
npm create vite@latest dopamine-reset-partner -- --template react-ts
cd dopamine-reset-partner
npm install
```

**Core dependencies:**

```json
{
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "react-router-dom": "^6.22.0",
    "zustand": "^4.5.0",
    "react-query": "^5.28.0",
    "socket.io-client": "^4.7.0",
    "date-fns": "^3.6.0",
    "recharts": "^2.12.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "vite": "^5.2.0",
    "vite-plugin-pwa": "^0.20.0",
    "@vitejs/plugin-react": "^4.2.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.0"
  }
}
```

**State management:** Zustand (lightweight, no boilerplate — appropriate for the partner app's focused scope)

**Data fetching:** React Query (handles caching and background refetch for real-time alert updates)

**Real-time:** Socket.io client (deletion attempt alerts must arrive in under 3 seconds)

---

## Backend & API

### Node.js + TypeScript + Fastify

```bash
mkdir dopamine-reset-api && cd dopamine-reset-api
npm init -y
npm install fastify @fastify/cors @fastify/jwt @fastify/websocket
npm install pg drizzle-orm redis ioredis
npm install -D typescript @types/node ts-node nodemon
```

**Why Fastify over Express:** Fastify's schema-based validation (JSON Schema) enforces strict request/response contracts on every endpoint. For an accountability system where data integrity matters, automatic validation is non-negotiable.

**Key dependencies:**

| Package | Purpose |
|---|---|
| `fastify` | HTTP server |
| `@fastify/jwt` | JWT auth middleware |
| `@fastify/websocket` | WebSocket server for real-time alerts |
| `drizzle-orm` | Type-safe ORM with raw SQL escape hatch |
| `pg` | PostgreSQL driver |
| `ioredis` | Redis client (sessions, vault code storage, rate limiting) |
| `bullmq` | Job queue (digest generation, scheduled alerts) |
| `zod` | Runtime schema validation for all API inputs |
| `nodemailer` | Email sending interface |
| `@aws-sdk/client-s3` | Why-video upload to S3 |

**API structure:**

```
POST   /auth/register
POST   /auth/login
POST   /auth/refresh

GET    /user/profile
PATCH  /user/settings
GET    /user/streaks
GET    /user/bypasses

POST   /vault/setup
GET    /vault/code           # partner-only
POST   /vault/release        # partner-only
POST   /vault/hold           # partner-only

GET    /digest/latest
GET    /digest/history

POST   /events/bypass
POST   /events/relapse
POST   /events/redirect-complete

GET    /partner/alerts
PATCH  /partner/alerts/:id

WS     /realtime             # WebSocket for instant alerts
```

---

## Database Layer

### PostgreSQL 16

```bash
# Local development
docker run -d \
  --name dopamine-reset-db \
  -e POSTGRES_USER=dreset \
  -e POSTGRES_PASSWORD=localdev \
  -e POSTGRES_DB=dopamine_reset \
  -p 5432:5432 \
  postgres:16-alpine
```

**Core schema overview:**

```sql
-- Users
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         TEXT UNIQUE NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT now(),
  why_video_url TEXT,
  settings      JSONB DEFAULT '{}'
);

-- Vault codes (held by partner, rotated every 48h)
CREATE TABLE vault_codes (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id),
  partner_id    UUID REFERENCES users(id),
  code_hash     TEXT NOT NULL,           -- bcrypt hash, never stored plain
  expires_at    TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT now()
);

-- Immutable audit log (append-only, no UPDATE/DELETE allowed)
CREATE TABLE audit_events (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id),
  event_type    TEXT NOT NULL,           -- bypass_attempt | relapse | redirect | deletion_attempt
  context       JSONB,
  occurred_at   TIMESTAMPTZ DEFAULT now()
);

-- Streaks
CREATE TABLE streaks (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id),
  started_at    TIMESTAMPTZ NOT NULL,
  ended_at      TIMESTAMPTZ,            -- NULL = active streak
  length_days   INT GENERATED ALWAYS AS (
    EXTRACT(DAY FROM COALESCE(ended_at, now()) - started_at)::INT
  ) STORED
);

-- Weekly digests (snapshot at generation time)
CREATE TABLE weekly_digests (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id),
  week_start    DATE NOT NULL,
  data          JSONB NOT NULL,          -- full metrics snapshot
  sent_at       TIMESTAMPTZ
);
```

**Critical constraint:** The `audit_events` table must have row-level security enabled and an explicit `DENY` on UPDATE and DELETE for the application role. Bypass history must be structurally immutable — not just policy-immutable.

```sql
-- Enforce append-only on audit_events
REVOKE UPDATE, DELETE ON audit_events FROM app_role;
```

**ORM:** Drizzle ORM for type-safe queries with raw SQL available for complex audit trail queries.

---

## Real-Time & Notifications

### Push Notifications — Firebase Cloud Messaging (FCM)

Used for: deletion attempt alerts (urgent), settings change requests, streak milestones, silent period alerts.

```bash
npm install firebase-admin
```

```typescript
// Server-side — send urgent alert to partner
import { initializeApp, cert } from 'firebase-admin/app';
import { getMessaging } from 'firebase-admin/messaging';

const message = {
  token: partnerFcmToken,
  notification: {
    title: 'Vault alert — deletion attempt',
    body: `${userName} is trying to uninstall. Tap to respond.`,
  },
  android: { priority: 'high' },
  apns: { headers: { 'apns-priority': '10' } },
};

await getMessaging().send(message);
```

### Email Digests — SendGrid

Weekly Sunday digest sent at 8:00 AM in the user's local timezone.

```bash
npm install @sendgrid/mail
```

### SMS Fallback — Twilio

For partners who don't have the app installed. Urgent vault alerts (deletion attempts) fall back to SMS after 60 seconds of no FCM acknowledgment.

```bash
npm install twilio
```

### WebSocket — Socket.io

Real-time alert channel between API and partner web app. Target latency under 3 seconds for deletion attempt alerts.

---

## DNS-Layer Blocking Service

### NextDNS API Integration

The app configures a NextDNS profile for the user at setup. The profile blocklist is managed server-side and applied at DNS resolution level — effective even without the app running.

```typescript
// Add a domain to the user's block profile
const response = await fetch(
  `https://api.nextdns.io/profiles/${profileId}/denylist`,
  {
    method: 'POST',
    headers: {
      'X-Api-Key': process.env.NEXTDNS_API_KEY,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ id: 'tiktok.com', active: true }),
  }
);
```

**Setup flow:**
1. App creates a NextDNS profile via API on user onboarding
2. User configures their device DNS to the NextDNS resolver address
3. App manages blocklist remotely — user cannot edit it directly
4. Bypass requires DNS reconfiguration (a manual, time-consuming step)

**Alternative:** ControlD (similar API surface, supports per-device profiles without router changes via app-level VPN).

---

## Browser Extension

### TypeScript + WebExtensions API (Chrome + Firefox + Safari)

```bash
mkdir dopamine-reset-extension && cd dopamine-reset-extension
npm init -y
npm install -D typescript webpack webpack-cli copy-webpack-plugin @types/chrome
```

**Manifest V3 structure:**

```json
{
  "manifest_version": 3,
  "name": "Dopamine Reset",
  "version": "1.0.0",
  "permissions": ["storage", "tabs", "webNavigation", "alarms"],
  "host_permissions": ["<all_urls>"],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [{
    "matches": ["<all_urls>"],
    "js": ["content.js"]
  }],
  "action": {
    "default_popup": "popup.html"
  }
}
```

**Key responsibilities:**

| Script | Responsibility |
|---|---|
| `background.ts` | Block list enforcement, bypass attempt logging, alarm-based session caps |
| `content.ts` | Inject friction gate overlay, grayscale filter, intent prompt |
| `popup.ts` | Quick stats, current session status |

**Cross-browser build:**

```bash
# Build for Chrome (Manifest V3)
npm run build:chrome

# Build for Firefox (Manifest V2 adapter)
npm run build:firefox

# Safari extension via Xcode wrapping
npm run build:safari
```

---

## Infrastructure & DevOps

### Terraform (HCL)

All cloud infrastructure defined as code. Target cloud: AWS (primary), with GCP for Firebase services.

```hcl
# Example: RDS PostgreSQL instance
resource "aws_db_instance" "main" {
  identifier        = "dopamine-reset-prod"
  engine            = "postgres"
  engine_version    = "16.2"
  instance_class    = "db.t4g.medium"
  allocated_storage = 50
  storage_encrypted = true        # mandatory for health data
  deletion_protection = true      # cannot be deleted without manual override

  backup_retention_period = 30    # 30-day backups
  backup_window           = "03:00-04:00"
}
```

### CI/CD — GitHub Actions (YAML)

```yaml
name: Deploy API

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run type-check
      - run: npm run test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster dopamine-reset \
            --service api \
            --force-new-deployment
```

### Containerization — Docker

```dockerfile
# API Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

---

## Security & Encryption

### Vault Code Security

The 6-digit vault code is never stored in plaintext anywhere.

```typescript
import bcrypt from 'bcrypt';
import crypto from 'crypto';

// Generate and store a new vault code
function generateVaultCode(): { plain: string; hash: string } {
  const plain = crypto.randomInt(100000, 999999).toString();
  const hash = bcrypt.hashSync(plain, 12);
  return { plain, hash };
}

// Only the partner receives the plain code (via push notification)
// The database stores only the hash
// Verification:
function verifyCode(input: string, hash: string): boolean {
  return bcrypt.compareSync(input, hash);
}
```

### Why-Video Encryption

User-recorded "why" videos are sensitive. Encrypted at rest using AWS S3 server-side encryption (SSE-KMS) with a per-user KMS key. Accessible only via short-lived signed URLs (15-minute expiry).

### Data in Transit

All API traffic over TLS 1.3. WebSocket connections over WSS. Certificate pinning enforced in both iOS and Android apps.

### Audit Log Integrity

Audit events are hashed as a chain — each event includes the hash of the previous event, making silent retroactive modification detectable.

---

## AI / Behavioral Intelligence Layer

### Anthropic API (Claude)

Used for two features:

**1. Personalized redirect copy**
When a friction gate fires, the intent prompt and redirect suggestions are generated dynamically based on the user's own urge journal patterns.

```typescript
const response = await fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: {
    'x-api-key': process.env.ANTHROPIC_API_KEY,
    'anthropic-version': '2023-06-01',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 200,
    system: `You write short, compassionate friction-gate prompts for a dopamine
             reduction app. The user's most common craving trigger is: ${topTrigger}.
             Their stated goal is: ${userWhy}. Write a single sentence (max 20 words)
             that creates a brief pause and gently redirects.`,
    messages: [{ role: 'user', content: 'Generate a friction prompt.' }],
  }),
});
```

**2. Weekly digest narrative**
A 2–3 sentence plain-language summary of the user's week is generated from metrics and included in the digest email.

---

## Full Dependency Map

```
dopamine-reset/
├── apps/
│   ├── ios/                     Swift 5.9 + SwiftUI
│   ├── android/                 Kotlin 1.9 + Compose
│   ├── partner-web/             TypeScript + React 18 + Vite
│   └── extension/               TypeScript + WebExtensions API
├── services/
│   ├── api/                     TypeScript + Node.js + Fastify
│   ├── digest-worker/           TypeScript + BullMQ
│   └── vault-rotation/          TypeScript + BullMQ (48h code rotation)
├── infra/
│   ├── terraform/               HCL (Terraform 1.7+)
│   └── docker/                  Dockerfiles per service
├── scripts/
│   ├── seed.ts                  TypeScript
│   └── migrate.ts               TypeScript + Drizzle Kit
└── .github/
    └── workflows/               YAML (GitHub Actions)
```

---

## Getting Started — Step by Step

### Prerequisites

Install the following before starting:

```bash
# Node.js (v20 LTS)
# https://nodejs.org

# Xcode 15+ (macOS, for iOS development)
# https://developer.apple.com/xcode/

# Android Studio Hedgehog+ (for Android development)
# https://developer.android.com/studio

# Docker Desktop
# https://www.docker.com/products/docker-desktop/

# Terraform CLI
brew install terraform

# PostgreSQL client
brew install postgresql@16
```

---

### Step 1 — Clone and install

```bash
git clone https://github.com/your-org/dopamine-reset.git
cd dopamine-reset
npm install      # root workspace install
```

---

### Step 2 — Start local infrastructure

```bash
# Start PostgreSQL + Redis via Docker Compose
docker compose up -d

# Verify containers are running
docker compose ps
```

**`docker-compose.yml`:**

```yaml
version: '3.9'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dreset
      POSTGRES_PASSWORD: localdev
      POSTGRES_DB: dopamine_reset
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pg_data:
```

---

### Step 3 — Run database migrations

```bash
cd services/api
npm run db:migrate       # runs Drizzle migrations
npm run db:seed          # seeds test users and partner pairs
```

---

### Step 4 — Configure environment variables

```bash
cp .env.example .env
# Fill in the values (see Environment Variables Reference below)
```

---

### Step 5 — Start the API server

```bash
cd services/api
npm run dev              # starts Fastify with hot reload via nodemon
# API available at http://localhost:3000
```

---

### Step 6 — Start the partner web app

```bash
cd apps/partner-web
npm install
npm run dev              # Vite dev server at http://localhost:5173
```

---

### Step 7 — Start the browser extension (development mode)

```bash
cd apps/extension
npm install
npm run build:dev        # watch mode build

# Chrome:
# Open chrome://extensions → Enable developer mode → Load unpacked → select dist/chrome/

# Firefox:
# Open about:debugging → This Firefox → Load Temporary Add-on → select dist/firefox/manifest.json
```

---

### Step 8 — iOS setup

```bash
cd apps/ios
open DopamineReset.xcodeproj

# In Xcode:
# 1. Set your development team under Signing & Capabilities
# 2. Add the FamilyControls entitlement (requires Apple Developer account + approval)
# 3. Select a simulator or connected device
# 4. Press Cmd+R to build and run
```

**Apple FamilyControls entitlement request:**
Go to [developer.apple.com/contact/request/family-controls-distribution](https://developer.apple.com/contact/request/family-controls-distribution) and submit a request. This typically takes 2–5 business days.

---

### Step 9 — Android setup

```bash
cd apps/android
# Open in Android Studio

# Before running:
# 1. Create a Firebase project at console.firebase.google.com
# 2. Download google-services.json and place in apps/android/app/
# 3. Enable Usage Access permission in the emulator:
#    Settings → Apps → Special App Access → Usage Access → DopamineReset → Allow
# 4. Run via Android Studio or:

./gradlew assembleDebug
adb install app/build/outputs/apk/debug/app-debug.apk
```

---

## Environment Variables Reference

```bash
# Database
DATABASE_URL=postgresql://dreset:localdev@localhost:5432/dopamine_reset

# Redis
REDIS_URL=redis://localhost:6379

# Auth
JWT_SECRET=your_jwt_secret_min_32_chars
JWT_REFRESH_SECRET=your_refresh_secret_min_32_chars

# Firebase (push notifications)
FIREBASE_PROJECT_ID=your-firebase-project
FIREBASE_CLIENT_EMAIL=firebase-adminsdk@your-project.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n..."

# SendGrid (email digests)
SENDGRID_API_KEY=SG.xxx
SENDGRID_FROM_EMAIL=digest@dopaminereset.app

# Twilio (SMS fallback)
TWILIO_ACCOUNT_SID=ACxxx
TWILIO_AUTH_TOKEN=xxx
TWILIO_FROM_NUMBER=+1xxxxxxxxxx

# NextDNS (DNS blocking)
NEXTDNS_API_KEY=xxx

# AWS (video storage)
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_REGION=ap-southeast-1
AWS_S3_BUCKET=dopamine-reset-videos

# Anthropic (AI nudge copy)
ANTHROPIC_API_KEY=sk-ant-xxx

# App
NODE_ENV=development
PORT=3000
```

---

## Project Folder Structure

```
dopamine-reset/
├── apps/
│   ├── ios/
│   │   ├── DopamineReset/
│   │   │   ├── App/
│   │   │   ├── Features/
│   │   │   │   ├── Onboarding/
│   │   │   │   ├── FrictionGate/
│   │   │   │   ├── Vault/
│   │   │   │   ├── Tracker/
│   │   │   │   └── Recovery/
│   │   │   └── Services/
│   │   └── DopamineReset.xcodeproj
│   │
│   ├── android/
│   │   └── app/src/main/
│   │       ├── java/com/dopaminereset/
│   │       │   ├── onboarding/
│   │       │   ├── friction/
│   │       │   ├── vault/
│   │       │   ├── tracker/
│   │       │   └── recovery/
│   │       └── res/
│   │
│   ├── partner-web/
│   │   └── src/
│   │       ├── pages/
│   │       │   ├── Alerts.tsx
│   │       │   ├── Digest.tsx
│   │       │   └── Settings.tsx
│   │       ├── components/
│   │       ├── stores/
│   │       └── hooks/
│   │
│   └── extension/
│       └── src/
│           ├── background.ts
│           ├── content.ts
│           └── popup.ts
│
├── services/
│   ├── api/
│   │   └── src/
│   │       ├── routes/
│   │       │   ├── auth.ts
│   │       │   ├── vault.ts
│   │       │   ├── events.ts
│   │       │   └── digest.ts
│   │       ├── db/
│   │       │   ├── schema.ts
│   │       │   └── migrations/
│   │       ├── services/
│   │       │   ├── vault.service.ts
│   │       │   ├── notify.service.ts
│   │       │   └── ai.service.ts
│   │       └── server.ts
│   │
│   ├── digest-worker/
│   │   └── src/
│   │       ├── jobs/
│   │       │   ├── generate-digest.ts
│   │       │   └── send-digest.ts
│   │       └── worker.ts
│   │
│   └── vault-rotation/
│       └── src/
│           └── rotate.ts
│
├── infra/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── rds.tf
│   │   ├── redis.tf
│   │   ├── s3.tf
│   │   └── ecs.tf
│   └── docker/
│       ├── api.Dockerfile
│       └── worker.Dockerfile
│
├── shared/
│   └── types/                   # shared TypeScript types across web + API
│       ├── vault.types.ts
│       ├── digest.types.ts
│       └── events.types.ts
│
├── docker-compose.yml
├── package.json                 # npm workspace root
└── .github/
    └── workflows/
        ├── api.yml
        ├── partner-web.yml
        └── extension.yml
```

---

## Development Roadmap

### Phase 1 — Core infrastructure (weeks 1–4)
- PostgreSQL schema and migrations
- Fastify API with auth (JWT)
- Vault code generation, hashing, and rotation worker
- Basic iOS and Android shells with usage monitoring

### Phase 2 — Friction engine (weeks 5–8)
- iOS: FamilyControls + ScreenTime integration
- Android: UsageStatsManager + DevicePolicyManager
- Friction gate overlay (delay timer, grayscale, intent prompt)
- Session caps with hard lockout

### Phase 3 — Vault Partner system (weeks 9–12)
- Partner onboarding flow
- Real-time alerts via WebSocket + FCM
- Settings change request approval flow
- Vault lock screen on deletion attempt
- SMS fallback via Twilio

### Phase 4 — Tracking and digest (weeks 13–16)
- Urge journal and bypass audit log
- Streak calculation and display
- Weekly digest generation (BullMQ job)
- SendGrid email template
- Partner web app (React)

### Phase 5 — Browser extension (weeks 17–18)
- Chrome + Firefox builds
- Content script friction gate injection
- Background bypass logging

### Phase 6 — AI layer and polish (weeks 19–20)
- Anthropic API integration for nudge copy
- Digest narrative generation
- Why Vault video recording and S3 upload
- App Store and Play Store submission

---

*Dopamine Reset — Tech Stack & Getting Started*
*Development reference document, May 2026*
```
