# Desi Toon Gaon AI YouTube Studio v1
## Complete Technical Architecture & Development Plan

**Status:** Frontend prototype package; backend and Google integrations are not connected.
**Channel:** https://youtube.com/@desitoongaonyt
**Scope:** Single channel · Mixed content · Approval-first · 6-hour monitoring target

## 1. Product scope

The Studio is a single-channel creator operations console for:
- Cooking
- Family
- Indian Village Life
- 2D Cinematic Animated Documentaries

Core modules: Overview, Research, Content Plan (7/15/30 days), Projects, SEO Lab, Analytics, Approvals, Settings. The supplied `index.html` implements responsive navigation, module layouts, local draft interactions, and explicit disconnected states. It intentionally contains no fabricated channel metrics or simulated live research.

## 2. Recommended production architecture

```text
Android / Desktop Browser
        |
        | HTTPS + secure session cookie
        v
Frontend (static app / CDN)
        |
        v
Backend API / BFF  -------------------- AI provider adapter
        |                               (script/prompt/SEO drafts)
        +---- OAuth service
        |       |
        |       +---- Google OAuth 2.0
        |                |
        |                +---- YouTube Data API v3
        |                +---- YouTube Analytics API
        |
        +---- PostgreSQL (projects, plans, approvals, audit)
        +---- Encrypted token vault / KMS
        +---- Object storage (video assets, thumbnails, exports)
        +---- Queue + scheduler (6-hour jobs)
        +---- Worker services (sync, analysis, reports)
```

Suggested stack:
- Frontend: React + TypeScript + Vite; Tailwind or CSS modules; PWA optional.
- Backend: Node.js + TypeScript (Fastify/NestJS) or Python FastAPI.
- Database: PostgreSQL; migrations with Prisma/Drizzle/Alembic.
- Queue: Redis + BullMQ/Celery; managed scheduler where available.
- Object storage: private S3-compatible bucket with signed upload/download URLs.
- Secrets: cloud Secret Manager/KMS; never browser localStorage or source control.
- Observability: structured logs, metrics, traces, alerting, error reporting.

## 3. Google OAuth and YouTube integration

### OAuth
Use Google OAuth 2.0 Authorization Code flow with PKCE where supported, exact redirect URI allow-list, random `state`, nonce/session binding, CSRF protection, and server-side code exchange. Do not embed a Google client secret in the browser. Do not return refresh tokens to the frontend.

Recommended least-privilege scopes (confirm current Google scope requirements before implementation):
- `https://www.googleapis.com/auth/youtube.readonly` for channel/video read operations.
- `https://www.googleapis.com/auth/yt-analytics.readonly` for analytics reports.
- Add write scopes only when implementing a user-approved write feature. Request incremental authorization rather than broad access up front.

Tokens:
- Store refresh tokens encrypted at rest using KMS envelope encryption.
- Access tokens remain server-side and short-lived.
- Rotate/re-encrypt on key rotation; revoke on disconnect.
- Redact tokens, authorization codes, and secrets from logs.
- Support reconnect, revocation, token-expiry recovery, and account/channel mismatch handling.
- Backend verifies the authenticated Google identity and selected channel ID; never trust a channel ID supplied by the browser without ownership verification.

### APIs
- YouTube Data API v3: channels.list, videos.list, playlistItems.list, search.list (quota-sensitive), videos.insert, videos.update, videos.delete, commentThreads.list, comments.insert where permitted.
- YouTube Analytics API: reports.query for authorized channel metrics such as views, estimatedMinutesWatched, averageViewDuration, averageViewPercentage, impressions/click-through metrics where available, and traffic-source dimensions where supported.
- Metric availability varies by report, date range, content type, and API support. UI must show unavailable/unsupported rather than infer or fabricate values.
- Track quota costs and cache safe read results. Do not poll expensive search endpoints unnecessarily.

## 4. Approval-first policy (mandatory)

All sensitive actions require a distinct, explicit approval. No background worker may bypass this gate.

Protected actions:
1. Video upload
2. Publish or schedule
3. Delete video
4. Visibility change (private/unlisted/public)
5. Metadata change (title, description, tags, category, thumbnail, etc.)
6. Public comment reply

Approval request record:
- `id`, `channel_id`, `action_type`, `resource_id`, `requested_by`
- exact `before_state` and `proposed_after_state`
- immutable payload hash, reason, created/expiry timestamps
- status: `pending | approved | rejected | expired | executed | failed`
- `approved_by`, `approved_at`, `execution_id`, `idempotency_key`
- audit correlation ID and API result summary

Workflow:
1. Draft action and show exact diff/payload to the owner.
2. Create a pending approval record; do not call write API.
3. Owner authenticates and approves that one request.
4. Backend checks owner authorization, expiry, action type, payload hash, current resource state, channel ownership, and idempotency.
5. Execute the exact approved payload once.
6. Persist result and append immutable audit event.
7. If payload changes, invalidate prior approval and request a new one.

Approval UI must clearly show target video/comment, visibility, publish time, before/after metadata, and destructive-action warnings. Delete should require an additional typed confirmation. Approval must not be reusable across action types or payload versions. Provide reject, revoke-before-execution, expiry, and full audit history.

## 5. Six-hour monitoring architecture

Target cadence: every 6 hours (four runs per day). This is an architecture target, not active monitoring in this prototype.

Scheduler:
- Register a timezone-aware recurring job (`0 */6 * * *` UTC or equivalent managed schedule).
- Enqueue one job per connected channel; use a distributed lock to prevent overlapping runs.
- Persist `scheduled_at`, `started_at`, `finished_at`, status, attempt count, and cursor/watermark.
- Apply exponential backoff with jitter for 429/5xx/network failures; respect `Retry-After`.
- Dead-letter repeated failures and alert the owner.
- Do not retry non-idempotent write actions automatically.

Suggested read-only run:
1. Validate OAuth connection and channel ownership.
2. Sync channel/video metadata incrementally.
3. Query supported analytics windows; retain report date range and API query metadata.
4. Compare with prior comparable period; calculate deltas only from real retrieved values.
5. Detect configured anomalies (e.g., material view/retention changes) with documented thresholds.
6. Generate a draft report and notification; do not take public/write action.
7. Store provenance, timestamps, and API errors.

Monitoring outputs should be marked `fresh`, `stale`, `partial`, or `unavailable`. A missing metric is not zero. Never invent a trend line from absent data.

## 6. Module specifications

### Overview — Channel Dashboard
- Connection state, last sync, data freshness, pending approvals, recent projects.
- Channel health audit: upload cadence, metadata completeness, policy/strike state only where authorized/available, audience and performance trends.
- Audit findings include evidence, severity, source timestamp, and recommended next step.

### Research
- Topic brief, trend hypotheses, keyword groups, audience intent, seasonality.
- Competitor public-channel comparison using permitted public data; distinguish observed facts from editorial interpretation.
- Search/trend provider adapters must be declared and attributed. No “trending now” claims without a live source and retrieval timestamp.
- Research output: sources, retrieved-at time, confidence, limitations.

### Content Plan
- Calendar views for 7, 15, and 30 days.
- Draft, planned, in-production, awaiting approval, published, cancelled states.
- Category, format, hook, target audience, planned date, linked project.
- Calendar entries do not imply scheduled YouTube publication.

### Projects — production workflow
Stages:
1. Research brief and concept.
2. Hook, story outline, complete script.
3. Scene-by-scene breakdown and continuity checks.
4. Image prompts with character/style locks.
5. Image-to-video animation prompts (including Flow AI-friendly concise prompts).
6. Hindi voice-over plan, pronunciation notes, pacing.
7. Sound design: ambience, Foley, music cues, mix notes.
8. Thumbnail concept and text safe-area notes.
9. SEO package: title options, description, tags, hashtags, keywords.
10. Review, export, upload approval, publish approval.

Each asset should have versions, author, timestamps, linked source brief, and review state. Support user-provided character reference IDs and continuity constraints as project data.

### SEO Lab
- Generate and edit title variants, descriptions, tags, hashtags, keyword clusters.
- Show character counts and policy checks.
- Save drafts only. Any YouTube metadata write creates a protected approval request.

### Analytics
- Views, watch time, CTR/impressions where available, average view duration, retention, traffic sources.
- Video-level and channel-level filters, date comparisons, exportable daily/weekly reports.
- Clearly disclose metric definitions and API availability. Do not conflate YouTube Studio UI metrics with similarly named API metrics.

### Approvals
- One action-specific approval per protected operation.
- Exact payload preview, diffs, target resource, expiry, approve/reject.
- Audit history and execution status. Require re-approval after any payload mutation.

### Settings
- Channel connection, OAuth status, scopes, scheduler health, monitoring cadence.
- Category preferences, timezone, report preferences, alert thresholds.
- Disconnect/revoke, data export/deletion, retention policy, user roles (future multi-user).

## 7. Comment analysis and replies
- Read comments only with approved OAuth scope and API access.
- Classify themes/sentiment/spam as AI suggestions, not facts.
- Draft replies privately.
- Every public reply must create its own approval request and show exact comment + reply text before execution.
- Respect platform policies, blocklists, rate limits, and deleted/disabled comments.

## 8. Backend API outline

All endpoints require authenticated session and CSRF protections where cookie-based:
- `GET /api/session`
- `GET /api/channel`
- `POST /api/integrations/google/connect`
- `GET /api/integrations/google/callback`
- `POST /api/integrations/google/disconnect`
- `GET /api/analytics/overview?from=&to=`
- `GET /api/analytics/videos/:videoId`
- `GET /api/research/jobs/:id`
- `POST /api/research/jobs`
- `GET|POST /api/content-plans`
- `GET|POST|PATCH /api/projects`
- `POST /api/seo/drafts`
- `GET|POST /api/approvals`
- `POST /api/approvals/:id/approve`
- `POST /api/approvals/:id/reject`
- `GET /api/audit-events`
- `GET /api/reports/daily`
- `GET /api/reports/weekly`

Never expose a generic browser endpoint that accepts arbitrary YouTube API methods or arbitrary URLs. Implement allow-listed server actions.

## 9. Core data model

- `users`: internal user identity and auth provider mapping.
- `channels`: Google channel ID, title, handle, owner user, connection status.
- `oauth_credentials`: encrypted refresh token, granted scopes, expiry metadata.
- `videos`: YouTube video ID, metadata snapshot, privacy status, published time.
- `analytics_snapshots`: channel/video, metric key/value, dimensions, date range, source query, fetched_at.
- `research_jobs`, `research_sources`, `competitor_snapshots`.
- `content_plans`, `content_plan_items`.
- `projects`, `project_assets`, `project_stage_events`.
- `seo_drafts`, `comment_drafts`.
- `approval_requests`, `audit_events`, `monitoring_runs`, `scheduled_reports`.

Use tenant/channel ownership constraints even in a single-channel v1. Store timestamps in UTC and render in the configured channel timezone.

## 10. Security, privacy, and reliability checklist
- HTTPS only; secure headers; CSP; HSTS; input validation; output escaping.
- HttpOnly + Secure + SameSite cookies; CSRF defense; session rotation.
- OAuth state/PKCE; redirect URI allow-list; least-privilege scopes.
- KMS-encrypted tokens; secrets manager; no secrets in HTML, JS, build variables exposed to client, logs, or Git.
- RBAC/owner-only approval; re-authentication for destructive actions.
- Rate limits, quota budgets, idempotency, replay protection.
- Append-only audit trail; backups; restore test; retention/deletion workflow.
- AI prompt injection defenses for comments/research text; treat external content as untrusted data.
- Human review for factual claims, policy-sensitive content, and all public actions.
- Accessibility: keyboard navigation, semantic labels, contrast, responsive touch targets.

## 11. Development phases

**Phase 0 — Prototype (included):** responsive HTML dashboard, local draft interactions, disconnected states, documentation.

**Phase 1 — Secure foundation:** backend, database, deployment, session auth, secrets manager, logging, CI/CD.

**Phase 2 — Read-only YouTube connection:** Google OAuth, channel verification, Data API sync, Analytics API reports, freshness states.

**Phase 3 — Planning & AI drafting:** projects, calendar, research adapters, script/scene/prompt/voice/audio/thumbnail/SEO drafts; versioning.

**Phase 4 — Approval engine:** protected-action queue, payload hashes, owner approvals, audit events, safe execution adapters; begin with metadata or comment replies in a test channel.

**Phase 5 — Upload/publish operations:** resumable upload, private-by-default upload, explicit separate publish/visibility approvals, recovery and idempotency.

**Phase 6 — Monitoring & reports:** six-hour scheduler, anomaly rules, daily/weekly report drafts, notifications.

**Phase 7 — Hardening:** security review, OAuth verification if needed, quota testing, accessibility, backup/restore, incident response.

## 12. Acceptance criteria
- Mobile viewport (Android Chrome) and desktop navigation work.
- No fabricated metrics or trend data; disconnected state is explicit.
- Google secrets and tokens never reach browser code.
- Every listed protected action has a distinct approval gate.
- Approval is bound to an immutable exact payload and expires.
- Monitoring runs at six-hour target only after deployment and connection.
- Failed/stale/partial data states are visible and not represented as zero.
- Audit trail records every approval decision and execution attempt.
- User can disconnect Google and revoke stored credentials.
