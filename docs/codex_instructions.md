# Codex Instructions — Receipt Management App

**System prompt for AI coding assistants to build and maintain the receipt management application**

## Guardrails (must follow)

* Money = **integer cents** (no floats). Dates = **ISO `YYYY-MM-DD`** over the wire; UTC for timestamps.
* Validate all inputs with **Zod**; server rejects unknown fields; derive types via `z.infer`.
* Accessibility: labeled fields, keyboard nav, `role="alert"`, alt text.
* Uploads: PNG/JPEG/WEBP/PDF ≤ 10 MB; verify MIME + magic bytes server-side; strip EXIF; store in **MinIO** via presigned URLs.
* Security: rate limit by plan, OWASP hardening, CSRF/SameSite=Lax, no PII in logs, JWT rotation (mobile), **RLS** in Postgres.
* Performance: indices, pagination, Redis caching, idempotent ingestion, Bull workers.
* TypeScript **strict**, no `any`, no unhandled promises.

## Monorepo Structure (workspaces)

```
/api (Express/TS + Prisma|TypeORM)
/web (Next.js app router + Tailwind)
/workers/ocr (Tesseract + Bull)
/mobile (React Native)
/packages/types (Zod+TS shared)
/packages/utils (money/date/a11y)
/infrastructure (docker-compose, nginx, scripts)
```

## Deliver these capabilities (with tests)

### 1. API

* `POST /receipts/upload` → presign in MinIO, enqueue `ocr:process`.
* `POST /receipts` (manual create), `GET /receipts` (filters+pagination), `PATCH /receipts/:id` (apply **delta** corrections).
* `GET /vendors/suggest?q=` (pg_trgm fuzzy).
* GDPR: `GET /gdpr/export`, `DELETE /gdpr/delete` (30-day job).

### 2. Database

* Models: organizations, users, receipts, vendors, line_items, corrections.
* Indices for org+date, status, content_hash; optional **monthly partitions**.
* Enable **RLS** policy for tenant isolation.

### 3. Workers (OCR)

* Preprocess (denoise, binarize, deskew) → Tesseract → field/line-item extract → field-level confidences; escalate to paid OCR only if `confidence < threshold`.

### 4. Web (Next.js)

* Dashboard (stats, trends).
* Upload (drag-drop, progress).
* Manual Wizard (Basic → Details → Items → Review).
* OCR Review (image pane + editable fields + confidence badges).
* Bulk Entry Grid (`role="grid"`, roving tabindex, paste parsing).

### 5. Mobile (React Native)

* Vision Camera capture → edge detect → resize → enqueue for **offline** background sync (retry/backoff).
* Quick Entry (camera/manual toggle, numeric pad, date shortcuts).

## Testing & Quality Gates

* Unit (utils, Zod), API (Supertest), Web (RTL + axe), Worker (golden OCR inputs), RN (Detox).
* Seed 100k receipts; list query p95 ≤ 300 ms with indices & pagination.
* CI: typecheck, lint, tests, build, Docker.

## Commands the assistant may perform

* Create/modify files (include imports & tests), write migrations, update `docker-compose.yml`, add runbooks under `/docs`.
* **Prohibitions:** float money, `Date` in API payloads, `any` types, unvalidated uploads.

## Priority tasks to implement in order

1. `/packages/types` and `/packages/utils` (MoneyCents, Zod schemas, date helpers).
2. DB schema + migrations + seed.
3. Upload endpoint + MinIO + OCR queue & worker.
4. Manual wizard + OCR review UI.
5. Vendor suggest (pg_trgm) + anomaly checks.
6. GDPR export/delete.
7. CI + perf checks.

## Model/Use Guidance

* Use current **chat/reasoning models** for coding and tool calling, and follow production & safety guidance.
* Review **security & privacy** posture and **usage policies** when handling user data and logs.
* Watch the **deprecations** page for model lifecycle notices before hard-pinning models in CI.

## Pitfalls to Avoid (Quick Checklist)

* Money as floats → **always integer cents** end-to-end.
* Date/time skew → persist **ISO date** for receipts; convert in UI only.
* "MIME only" upload checks → also validate **magic bytes** and strip EXIF.
* Logging payloads → **no PII**; log stable error codes and request IDs.
* OCR retries → cap attempts; gate escalation to paid OCR by confidence & budget.
* Missing indices → add org+date, status, content_hash; analyze plans before shipping.

## Additional Architecture Details

### Data Models

#### Organizations
* Multi-tenant support with proper isolation
* Subscription tiers and usage limits
* Admin roles and permissions

#### Users
* Authentication via JWT (web) and refresh tokens (mobile)
* Profile management and preferences
* Activity tracking for audit trails

#### Receipts
* Core entity with vendor, date, total, currency
* Status workflow: draft → processing → complete → archived
* Content hash for deduplication
* Attachment references to MinIO

#### Line Items
* Belong to receipts with quantity, unit price, total
* Category classification
* Tax and discount tracking

#### Corrections
* Audit trail of all edits with before/after values
* User attribution and timestamps
* Confidence scores from OCR

### API Design Principles

* RESTful with consistent naming conventions
* Pagination using cursor-based approach for large datasets
* Filtering via query parameters with Zod validation
* Sorting by common fields (date, vendor, total)
* Bulk operations with transaction support
* Idempotency keys for critical operations

### Security Requirements

* Authentication: JWT with short expiry, refresh token rotation
* Authorization: Role-based access control (RBAC)
* Data isolation: Row-level security (RLS) in Postgres
* API rate limiting by tier: Basic (100/hr), Pro (1000/hr), Enterprise (custom)
* Input validation: Zod schemas for all endpoints
* Output sanitization: No PII in logs or error messages
* File upload security: MIME type + magic byte validation, virus scanning
* HTTPS everywhere, HSTS headers
* CORS configuration for web app only
* SQL injection prevention via parameterized queries
* XSS prevention via Content Security Policy

### Performance Targets

* API response times: p50 < 100ms, p95 < 300ms, p99 < 1s
* Upload processing: < 5s for image under 5MB
* OCR processing: < 30s for standard receipt
* Dashboard load: < 2s initial, < 500ms subsequent
* Mobile app: cold start < 3s, camera ready < 1s
* Database queries: proper indexing for all common access patterns
* Caching strategy: Redis for session, vendor suggestions, dashboard stats

### Monitoring & Observability

* Structured logging with correlation IDs
* Metrics: response times, error rates, OCR accuracy
* Alerts: failed uploads, OCR queue depth, error spikes
* Health checks for all services
* Distributed tracing for OCR pipeline
* User analytics for feature usage

### Deployment & Infrastructure

* Docker containers for all services
* docker-compose for local development
* Environment-based configuration (dev, staging, prod)
* Database migrations with rollback support
* Zero-downtime deployments
* Backup strategy for Postgres and MinIO
* Disaster recovery plan with RTO < 4hrs

### Development Workflow

* Git flow with feature branches
* PR templates with checklist
* Code review required for all changes
* Automated testing in CI/CD
* Semantic versioning for packages
* Documentation updates with code changes
* Runbooks for common operations

This comprehensive instruction set ensures consistent implementation across all developers and AI assistants working on the receipt management application.