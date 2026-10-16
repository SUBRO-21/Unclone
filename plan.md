# VoiceAuth: "OAuth with your voice", built on Detectif.ai

## 1. Context

Detectif.ai (https://detectif.ai) sells voice models. Two matter here:
- **Eidos**: speaker verification (live voice vs. an enrolled voiceprint).
- **Aletheia**: deepfake, clone and replay detection.

It can be hosted API, private VPC, on-prem or on-device. KYC onboarding, payment authorization and account recovery are listed use cases.

**Goal:** a platform other apps integrate with, the way they integrate "Sign in with Google". The user proves who they are by speaking. The relying app (RP) gets a standard OIDC token. Voice becomes an authentication factor, and also an identity-assurance signal for KYC and step-up.

**Big unknown:** Detectif.ai's API docs are not public. The site has no docs page and `/docs` returns 404. Access appears to be demo-gated. So:
- Step 0 is to get API access and docs from them.
- Everything is built behind a `VoiceEngine` adapter interface, so we can develop against a mock and swap in the real client.

## 2. Product concept

Three layers:

1. **Voice Identity Provider (IdP)**: OIDC/OAuth2 Authorization Server. Users enroll a voiceprint once. At login they speak a challenge phrase.
2. **Developer platform**: tenants register apps and get client_id/secret, redirect URIs, policies, a dashboard, and webhooks.
3. **KYC / step-up layer**: an assurance level (`acr`) in the token. Optionally document + liveness checks, to bind a voiceprint to a verified real-world identity.

### Primary flows
- **Login (OIDC code flow + PKCE):** RP redirects to `/authorize`. The hosted voice page records the user speaking a random challenge phrase. The backend runs deepfake check, then speaker verify, then phrase check. On pass it issues an auth code, then the RP exchanges it at `/token` for an id_token and access_token.
- **Enrollment:** a one-time flow of 3-5 samples across varied phrases, with quality checks. It requires a fallback factor (email OTP or passkey) to bind the account.
- **Step-up / transaction auth:** RP requests `acr_values` and a `transaction_binding` (e.g. "pay ₹5,000 to X"). The user speaks a phrase containing a per-transaction code.
- **Recovery / re-enrollment:** via the fallback factor plus a cooldown. Optionally a KYC document check.

## 3. Security design

Voice alone is a weak factor, so the design is defensive:

- **Anti-spoof first:** run Aletheia (deepfake/replay) before Eidos. A fail on either is a fail.
- **Challenge-response:** a random phrase or digits, with a short TTL (~60s), single use, bound to the session. This defeats replayed recordings. Verify the spoken content via ASR (Whisper or similar) in addition to the speaker match.
- **Score policy:** thresholds are per tenant, with a risk-based mapping:
  - high score → `acr=voice`
  - borderline → require a second factor (OTP/passkey)
  - low → deny
- **Rate limits and lockout:** per user, IP, device and tenant. Exponential backoff on failed attempts.
- **Biometric data handling:**
  - Store only the voiceprint/embedding or Detectif's enrollment ID, never raw audio beyond a short retention window.
  - Encrypt at rest with per-tenant keys (KMS envelope encryption).
  - Audio and embeddings are purged on account deletion.
  - Consent capture and audit log. Voiceprints are biometric data (GDPR Art. 9, BIPA, DPDP India), so consent is explicit and revocable.
- **Standard hardening:**
  - PKCE required, exact redirect URI match, `state` and `nonce`.
  - Short-lived codes (≤60s) and tokens.
  - Refresh token rotation.
  - JWT signing with rotating keys and a JWKS endpoint.
  - CSRF protection and CSP on the hosted page.
  - Secrets in a secret manager.
- **Never the sole factor for high-value actions** unless the tenant opts in. Recommend voice + a device-bound passkey.

## 4. Architecture

```
 RP app ──redirect──▶ Auth Service (OIDC) ◀──▶ Postgres (users, clients, voice_profiles, audit)
                          │                  ◀──▶ Redis (sessions, challenges, rate limits)
                          ▼
              Voice Verification Service
                ├─ ChallengeService  (phrase gen, TTL, single-use)
                ├─ AudioPipeline     (decode, resample, VAD, quality/SNR checks)
                ├─ ASR check         (spoken content matches challenge)
                └─ VoiceEngine (interface)
                     ├─ DetectifEngine  (Aletheia + Eidos, real)
                     └─ MockEngine      (dev/tests)
                          │
 Object store (short-TTL audio) · KMS · Queue (webhooks, async KYC)
 Hosted Voice Login Page (web, MediaRecorder/WebAudio) · Mobile SDKs later
 Developer Dashboard · Webhook dispatcher · Admin/Audit UI
```

### Suggested stack (changeable; repo is empty)
- Backend: TypeScript (Node + Fastify/NestJS) or Python (FastAPI). Recommend **TypeScript monorepo** for sharing types with the web app and SDK.
- OIDC: build on a certified library (e.g. `node-oidc-provider`) rather than hand-rolling the protocol. Customize the interaction step to be voice.
- DB: Postgres. Cache/queue: Redis (BullMQ).
- Frontend: Next.js for the hosted voice page and the dashboard.
- Infra: Docker Compose for dev, then deploy to a cloud with KMS.

### Repo layout (proposed)
```
/apps/auth-server     OIDC provider + voice interaction endpoints
/apps/web             hosted voice login page + developer dashboard
/packages/voice-engine  VoiceEngine interface, DetectifEngine, MockEngine
/packages/audio       decode/VAD/quality utils
/packages/sdk-js      RP client SDK (button + helpers)
/packages/shared      types, zod schemas
/infra                docker-compose, migrations
/docs                 API + integration guide
```

## 5. Data model (core tables)

- `tenants`, `developers`
- `clients` (client_id, secret hash, redirect_uris, allowed_acr, thresholds, branding)
- `users` (subject id, tenant scope, fallback factor refs, status)
- `voice_profiles` (user_id, engine, engine_enrollment_id, enrolled_at, sample_count, consent_id, key_ref)
- `consents` (user, tenant, purpose, version, timestamp, revoked_at)
- `challenges` (id, user/session, phrase, expires_at, used_at)
- `auth_attempts` (user, client, scores, decision, spoof_flag, ip, device, ts). No raw audio.
- `audit_log` (append-only), `webhook_endpoints`, `webhook_deliveries`
- `kyc_checks` (type, status, evidence refs, retention)

## 6. API surface

OIDC standard: `/.well-known/openid-configuration`, `/authorize`, `/token`, `/userinfo`, `/jwks`, `/revoke`, `/introspect`.

Voice interaction (internal, used by the hosted page):
- `POST /voice/enroll/start`, `/voice/enroll/sample`, `/voice/enroll/complete`
- `POST /voice/challenge` returns phrase + id
- `POST /voice/verify` takes audio + challenge id, returns decision (not raw scores to the client)
- `DELETE /voice/profile` (right to erasure)

Developer: client CRUD, policy config, webhook config, usage and attempt analytics.

Tokens: id_token claims include `sub`, `acr` (`voice`, `voice+mfa`, `voice+kyc`), `amr` (`["voice","otp"]`), `auth_time`, optional `voice_match_level` (coarse band, not raw score).

## 7. Detectif.ai integration (to be confirmed)

Assumptions to validate once we have docs:
- Enroll: submit N audio clips, get back an enrollment/voiceprint ID.
- Verify: submit a clip plus an enrollment ID, get a similarity score or decision.
- Aletheia: submit a clip, get a spoof probability.
- Audio constraints: format, sample rate (likely 16 kHz mono PCM/WAV), min/max duration.
- Auth: API key or OAuth, plus rate limits and pricing.
- Deployment: hosted API for the MVP. VPC/on-prem later for regulated tenants.
- Does it return per-clip scores usable for thresholds? Is the voiceprint portable (exportable) or stored only on their side? That decides lock-in and data-residency answers.

Action: request a demo/sandbox key via their form. Ask for the API reference, SLA, data-retention terms and DPA. Ask whether text-dependent verification is supported.

## 8. Phases

**Phase 0: Discovery (≈1 wk)**
- Get Detectif.ai access and docs. Record real request/response shapes.
- Finalize the `VoiceEngine` interface. Pick the stack.
- Threat model workshop: replay, cloning, injection, enrollment hijack.

**Phase 1: Voice core (≈2 wk)**
- Monorepo scaffold, Docker Compose (Postgres, Redis).
- `VoiceEngine` + `MockEngine` + `DetectifEngine`.
- Audio pipeline: decode, resample, VAD, quality checks. ChallengeService + ASR check.
- CLI/test harness: enroll and verify from wav files.

**Phase 2: IdP (≈2 wk)**
- OIDC provider with voice interaction step. Fallback factor (email OTP).
- Hosted voice enrollment and login pages.
- Client registration (seeded, no dashboard yet). End-to-end demo with a sample RP app.

**Phase 3: Developer platform (≈2 wk)**
- Dashboard: apps, keys, redirect URIs, thresholds, branding, analytics.
- Webhooks, JS SDK ("Sign in with Voice" button), docs site.

**Phase 4: Hardening and compliance (≈2 wk)**
- Rate limits, lockout, risk-based step-up. Per-tenant encryption and retention jobs.
- Consent management, erasure, audit log UI.
- Pen-test checklist, load test, observability (metrics, tracing, alerts).

**Phase 5: KYC layer (later)**
- Document + selfie liveness vendor integration; bind to voice profile; `acr=voice+kyc`.
- Transaction signing flow. Mobile SDKs (on-device capture). Private VPC deployment option.

## 9. Risks and open questions

- Detectif.ai API unknown or gated → mitigated by the adapter and mock.
- Voice cloning is rising → mandatory challenge-response, Aletheia, and a second factor for high-risk actions.
- Accuracy varies with noise, accent and device → quality gates, per-tenant thresholds, graceful fallback.
- Legal exposure from biometrics → consent, regional data residency, DPA with Detectif, lawyer review before launch.
- Vendor lock-in → adapter plus option to add other engines (e.g. open-source ECAPA-TDNN) as backups.
- Pricing per call may shape the business model (per-verification pricing for tenants).

## 10. Verification (once implementation begins)

- Unit tests: engine adapters (mock + recorded Detectif fixtures), challenge TTL/single-use, threshold logic.
- Integration: docker-compose up, run sample RP app through the full code+PKCE flow with test wavs: genuine pass, different speaker fail, replayed recording fail, TTS clone fail.
- Security: OIDC conformance suite, rate-limit and lockout tests, check no raw audio persists past TTL.
- Manual: real mic enrollment and login in browser, plus failure-mode UX (noise, short audio).

## 11. Decisions needed

1. Stack preference (TypeScript recommended vs Python).
2. Do you already have Detectif.ai API access or a sandbox key?
3. Target market and regulation (India/DPDP, EU/GDPR, US) and whether KYC with documents is in v1 or later.
4. B2B platform with multi-tenant dashboard from day one, or a single-tenant MVP first?
