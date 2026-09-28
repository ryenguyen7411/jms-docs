# 12 · Work breakdown

| | |
|---|---|
| **For** | Whoever picks up the next task |
| **Answers** | What to build, in what order, in 15–60 minute pieces |
| **Order** | `01-principles.md`. External items are in `11-rollout.md` |

```mermaid
flowchart LR
    F["Foundation"] --> B["Bridge"] --> L["Login"] --> S["SIM"] --> P["Ping"] --> I["Ingest"] --> V["Version, config"] --> R["Rollout"]
```

Each task includes its tests: happy path, edge cases, and failures. **Blocked** means waiting on an external **Confirm** (`11`).

## Server

| # | Task | Min | After |
|---|---|---|---|
| F1 | Config loader: per-environment settings, secret-manager client, 30 s reload of switches | 45 | — |
| F2 | PostgreSQL migrations framework and the base tables from `02` | 45 | F1 |
| F3 | Redis client, key helpers, distributed lock | 30 | F1 |
| F4 | Error envelope `{ message, code }`, request logging with redaction | 30 | — |
| B1 | Legacy HTTP client: base URL, 5 s timeout, bare-string code parsing | 45 | F1 |
| B2 | Back Office HTTP client | 30 | F1 |
| B3 | Outbox table and insert helper used inside handler transactions | 30 | F2 |
| B4 | Outbox worker: claim with SKIP LOCKED, backoff, park | 60 | B3 |
| B5 | Outbox order per `partitionKey` | 45 | B4 |
| B6 | Outbox metrics and alerts | 30 | B4 |
| B7 | Webhook route: HMAC, skew, `legacy_event` idempotency | 60 | F2 |
| B8 | Webhook handlers for `user.*` (revokedBefore) | 30 | B7, L3 |
| B9 | Webhook handler for `device.reset`, plus `audit_log` | 45 | B7, S1 |
| B10 | User reconcile job (hourly hash, paging) | 45 | B1, L3 |
| B11 | Device reconcile job and adoption numbers · **Blocked** on list shape | 60 | B1, S1 |
| L1 | Token signer and verifier, claims, key per environment | 45 | F1 |
| L2 | Auth middleware: Bearer, `typ`, `did` check, `WRONG_USER_TYPE` | 45 | L1 |
| L3 | Revocation in Redis: `jti` set and `revokedBefore` | 30 | F3, L2 |
| L4 | `POST /v1/auth/login` with legacy mapping · **Blocked** on login error body | 60 | B1, L1 |
| L5 | Login rate limit | 30 | L4 |
| L6 | `POST /v1/auth/refresh` with rotation and family revoke | 60 | L1, F2 |
| L7 | `POST /v1/auth/logout` | 20 | L3, L6 |
| L8 | `GET /v1/auth/config` | 30 | B1, L2 |
| S1 | `sim_device`, `leader_device`, `noti_device` tables, and the legacy-secret encryption helper | 45 | F2 |
| S2 | Phone normalisation, tested against production samples | 45 | — |
| S3 | RSA: parse the app key and encrypt; ephemeral pair and decrypt | 45 | — |
| S4 | BO verify-account call · **Blocked** on phone form | 30 | B2 |
| S5 | Legacy activate call and code mapping | 45 | B1, S3 |
| S6 | `POST /v1/sim/activate`, including the synchronous outbox flush | 60 | S1–S5, B5 |
| S7 | `POST /v1/sim/sign-out` · **Blocked** on legacy sign-out body | 45 | S1, B3 |
| S8 | `POST /v1/sim/reset` | 30 | S1, B3 |
| P1 | Heartbeat write to Redis, dirty set, pending set | 45 | F3 |
| P2 | Heartbeat flusher | 60 | P1, S1 |
| P3 | `POST /v1/devices/ping` with its checks | 45 | L2, P1 |
| P4 | `POST /v1/leaders/ping` | 30 | L2, P1 |
| P5 | `POST /v1/noti-devices/ping` | 30 | L2, P1 |
| P6 | Ping forwarder: latest-only, 32 in flight, pending alert | 60 | P1, B1 |
| P7 | Legacy device and leader payloads, client-IP header · **Blocked** on header name | 30 | P6 |
| P8 | BO notification-ping payload and signature · **Blocked** on route | 30 | P6, B2 |
| P9 | Route switch with overlap, `X-Config-Version` header | 30 | P6, V3 |
| I1 | `sms_inbox`, `sms_statement`, `noti_inbox` tables | 30 | F2 |
| I2 | Sender-mask and template sync · **Blocked** on route | 45 | B1 |
| I3 | `POST /v1/ingest/sms`: validation and signature | 60 | L2, S1, I1, I2 |
| I4 | `legacy.sms` payload and code mapping | 30 | B4, I3 |
| I5 | `POST /v1/ingest/notifications` and `bo.notification` forward | 60 | L2, I1, B4 |
| I6 | Shadow worker: claim, stuck-row recovery | 45 | I1 |
| I7 | Parser bKash, 4 cases | 60 | I6 |
| I8 | Parser Nagad, 2 cases | 45 | I6 |
| I9 | Parser Rocket, 5 cases | 60 | I6 |
| I10 | Parser Upay, 4 cases | 45 | I6 |
| I11 | Template parser: synced templates, 250 ms regex timeout | 60 | I2, I6 |
| I12 | Statement key, UTC time, balance state | 45 | I7–I11 |
| V1 | Version poller, safety rule, metrics | 45 | B2 |
| V2 | `GET /v1/app/version` with presigned URL | 45 | V1 |
| V3 | `GET /v1/app/config` with ETag | 30 | F1, L2 |
| X1 | B2B passthrough · **Blocked** on scope and legacy auth | 60 | B1, L2 |
| R1 | Parity report job | 60 | I12 |
| R2 | Dashboards and alerts from `11` | 60 | all |

## App

| # | Task | Min | After |
|---|---|---|---|
| A1 | Read and keep the old app's device id | 30 | — |
| A2 | First-launch reset: keep device id and queue, drop the rest | 45 | A1 |
| A3 | HTTP layer: headers, error codes to actions (`03`) | 60 | — |
| A4 | Token store, proactive single-flight refresh | 60 | A3 |
| A5 | Login, flags, and config screens and calls | 60 | A4 |
| A6 | SIM activation with RSA key pair, per SIM | 60 | A4 |
| A7 | Ping loops ×3, following the flags | 60 | A6 |
| A8 | Phone-side legacy ping and fallback | 45 | A7 |
| A9 | SMS upload with the new signature | 45 | A6 |
| A10 | Notification upload carrying `boSignature` | 45 | A4 |
| A11 | Drain the old app's queue | 45 | A9, A10 |
| A12 | Version check, blocking update screen, stop collection | 45 | A3 |
| A13 | Config cache, `X-Config-Version` refetch, base-URL fallback | 45 | A3 |
| A14 | Sign-out and reset screens | 30 | A6 |
