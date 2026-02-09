# Parental Safety Monitoring MVP Plan

## 1) High-Level System Architecture

### Architecture goals
- Real-time, low-latency communication across child app, parent mobile app, and parent web dashboard.
- Consent-first and legally compliant by design.
- Fast to ship (MVP), while preserving future extensibility.

### Logical components
1. **Child Android App (data producer)**
   - Captures:
     - Screen frames (screen mirroring stream)
     - Camera stream (on parent request)
     - Microphone stream (on parent request)
   - Maintains secure persistent channel to backend for signaling.
   - Handles remote session start/stop commands only from authorized parent sessions.

2. **Parent Mobile App + Parent Web Dashboard (data consumers)**
   - Authenticate parent account.
   - Display child list and one-tap controls for:
     - Start/stop screen mirror
     - Start/stop camera access
     - Start/stop voice access
   - Render live media streams with status indicators.

3. **Backend Control Plane (Django + Channels)**
   - Authentication + authorization + consent records.
   - Device pairing and session orchestration.
   - Signaling server for real-time session setup.
   - Audit logging (who accessed what stream and when).

4. **Media Layer (WebRTC SFU/TURN)**
   - WebRTC transport for low-latency streaming.
   - SFU for scalable fan-out to multiple parent viewers.
   - TURN relay fallback for restrictive networks.

5. **Datastore + Queue**
   - PostgreSQL for accounts, pairings, consent, audit trails.
   - Redis for presence/session state + pub/sub signaling support.

### Suggested architecture diagram (text)
- Parent App/Web ↔ **HTTPS/WSS** ↔ Django API/Channels (control plane)
- Child App ↔ **WSS signaling** ↔ Django/Channels
- Child stream ↔ **WebRTC (DTLS-SRTP)** ↔ SFU/TURN ↔ Parent App/Web
- Django ↔ PostgreSQL (persistent state)
- Django/Channels ↔ Redis (real-time state + message fanout)

### Security and compliance-by-design
- End-to-end encrypted media transport using WebRTC (DTLS-SRTP).
- TLS everywhere for API and signaling.
- Explicit guardian consent captured during onboarding.
- Child-visible disclosure mode (required by policy/legal config).
- Strict RBAC: only paired parent can initiate access.
- Immutable access logs for all session starts/stops.

---

## 2) MVP Feature Breakdown (ONLY the three core features)

## A. Screen Mirroring
**Parent experience**
- Tap “Screen” → live child screen starts in <3–5 seconds.
- Tap again to stop.

**Child app responsibilities**
- Android MediaProjection-based capture.
- Adaptive bitrate and frame rate (network-aware).
- Background foreground-service notification where legally required.

**Backend responsibilities**
- Validate parent-child pairing.
- Issue short-lived stream token.
- Establish session signaling and route WebRTC candidates.

**MVP acceptance criteria**
- Median startup latency <5 seconds on 4G/Wi-Fi.
- Continuous stream with graceful quality degradation.

## B. Camera Access
**Parent experience**
- Tap “Camera” → live rear camera feed (switch front/rear optional post-MVP unless trivial).

**Child app responsibilities**
- CameraX capture pipeline.
- Resource arbitration (camera vs screen usage policies).
- Session-aware lifecycle handling.

**Backend responsibilities**
- Same auth/signaling/token pattern as screen mirroring.

**MVP acceptance criteria**
- Reliable connect/disconnect handling.
- No camera stream without active authorized session.

## C. Voice Access
**Parent experience**
- Tap “Voice” → live ambient audio monitoring starts immediately.

**Child app responsibilities**
- Microphone capture with noise suppression where available.
- Audio-only stream optimization for low bandwidth.

**Backend responsibilities**
- Auth, signaling, audit logs identical to other streams.

**MVP acceptance criteria**
- Clear intelligible audio under moderate network variability.

---

## 3) User Flows

## Child App Flow (onboarding + runtime)
1. Install app on child device.
2. Guardian signs in and pairs device using QR/invite code.
3. Consent screens presented; permissions requested (screen, camera, mic, notifications, battery optimization exemptions as needed).
4. Child app enters protected background mode:
   - Persistent service health checks.
   - Tamper friction: PIN-protected settings, uninstall friction via Device Admin/profile owner path where lawful.
5. On parent request:
   - Receive signed command via WSS.
   - Validate token/session.
   - Start chosen stream (screen/camera/voice).
   - Emit live status and heartbeat.
6. On stop/timeout/error:
   - Tear down media pipeline.
   - Write audit event.

## Parent App / Web Flow
1. Parent signs in (email/password + optional 2FA for MVP if fast).
2. Select paired child profile.
3. Home dashboard shows exactly three primary actions:
   - Screen
   - Camera
   - Voice
4. Tap one action:
   - App calls backend to create session.
   - Receives signaling/token details.
   - Connects to stream.
5. View stream with minimal controls:
   - Start/Stop
   - Connection quality indicator
   - “Session active” timestamp
6. Session ends; log entry available in simple history view (access transparency).

---

## 4) Suggested Tech Stack for Rapid MVP

## Child-side Mobile App (Android-first)
- **Kotlin + Jetpack (Compose optional)**
- **Foreground Services + WorkManager** for resilient background behavior.
- **WebRTC Native SDK** for real-time media.
- **MediaProjection** (screen), **CameraX** (camera), **AudioRecord/WebRTC audio** (voice).

## Parent-side Mobile App
- **React Native** (fast cross-platform) or **Flutter**.
- WebRTC client libraries for stream playback.
- Shared auth/session logic with web via API SDK.

## Parent-side Web Dashboard
- **Next.js (React)** minimal interface.
- WebRTC playback in-browser.
- Clean one-screen console with three action cards.

## Backend / Infrastructure
- **Django + Django REST Framework** for API.
- **Django Channels + Redis** for WebSocket signaling and realtime state.
- **PostgreSQL** for persistent data.
- **Coturn** for TURN relay.
- **Managed SFU** (e.g., LiveKit/Janus-managed option) for MVP speed; self-host post-PMF.
- **Docker Compose (dev) / Kubernetes or ECS (prod)**.

## Auth & Security
- JWT access/refresh tokens.
- Device-bound refresh tokens and token rotation.
- BCrypt/Argon2 password hashing.
- At-rest encryption for sensitive metadata.

---

## 5) Data Model (MVP-minimal)
- `users` (parent accounts)
- `children` (child profile metadata)
- `devices` (child device identity + health)
- `pairings` (parent↔child relationship + consent status)
- `sessions` (screen/camera/voice session lifecycle)
- `audit_events` (immutable stream access logs)

---

## 6) Scalability Plan (without feature creep)
- Stateless API instances behind load balancer.
- Redis-backed channel layer for horizontal websocket scaling.
- SFU tier autoscaling based on concurrent sessions.
- Region-aware TURN/SFU placement to reduce latency.
- Event-driven hooks for future alerts/features, but no extra surveillance features in MVP.

---

## 7) MVP Build Plan (8–10 weeks)
1. **Week 1–2:** Auth, pairing, consent, baseline UI shells.
2. **Week 3–4:** Screen mirroring end-to-end.
3. **Week 5–6:** Camera + voice streaming.
4. **Week 7:** Hardening (permissions, reconnects, tamper friction).
5. **Week 8:** Security review, legal copy, observability, beta prep.
6. **Week 9–10 (buffer):** Performance tuning + launch fixes.

---

## 8) Non-negotiable MVP Principles
- Consent-first, transparent usage.
- Exactly three real-time capabilities: screen, camera, voice.
- One-tap parent experience.
- Secure by default.
- Architected for scale, but optimized for shipping quickly.
