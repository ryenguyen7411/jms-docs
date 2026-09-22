# 04 · Users, Login, Leader Accounts, Operator Features

> **Answers BE question 5**: "Besides the endpoints the app calls, what features does operations have?"

---

## 1. User model

| Attribute | Values | Meaning |
|---|---|---|
| `UserType` | `1` Master, `2` Leader | Master = collecting operator. Leader = view-only, balance pool |
| `SystemType` | `1` INTERNAL, `2` RESELLER, `3` SP | decides default feature flags |
| `Status` | `1` Active, `2` Disabled, `3` Deleted | Deleted is a soft delete; every query excludes it |
| `MerchantCode` | master merchant code | data scope |
| `MerchantId` | int | merchant id on the BO side |

There is **no "agent" user type**. "agent" only exists as a chat role derived at login: leaders get `leader`, everyone else gets `agent`.

Usernames are unique **across the whole deployment**: login does not filter by merchant or system.

---

## 2. Login

`POST /api/User/login` — unauthenticated endpoint.

Request: `{ Username, Password }`. (On the unreleased chat branch it also takes `DeviceId`, `Phone`, `Name`, `FcmToken`.)

Password hashing today is **unsalted MD5, lowercase hex**. The new backend must reproduce it for verification during migration and should rehash to a modern algorithm on first successful login.

Checks run in this fixed order and produce distinct errors:

1. user exists and is not deleted, else `INVALID_USERNAME`;
2. password matches, else `INVALID_PASSWORD`;
3. status is Active, else `INVALID_USER_STATUS_DISABLE`.

Response (production today — **no token is issued**):

```
{ Username, MerchantCode, SystemCode, SystemType, UserType, DisplayedUserType,
  IsUseLandingPage, IsSmsEnabled, IsNotiEnabled, IsPingEnabled, IsNotiPingEnabled }
```

The flags are always the **effective** flags (§3), never the raw stored columns.

### Device binding (unreleased chat branch)

The login also reconciles devices: same device with a new SIM logs out the old rows; same SIM on a new device kicks the previous device. The invariant is **one SIM, one active device** — not one user, one device. A leader has no SIM, so only its push token is updated.

---

## 3. Feature flags — the one rule that must be ported exactly

Five flags control what the app does:

| Flag | Effect in the app |
|---|---|
| `IsUseLandingPage` | show the landing page |
| `IsSmsEnabled` | collect SMS |
| `IsNotiEnabled` | collect bank-app notifications |
| `IsPingEnabled` | heartbeat to the SMS service (SMS path) |
| `IsNotiPingEnabled` | heartbeat to the BO (notification path) — a **different endpoint on a different service** |

Normalisation is applied on **every read and every write**, so stored and served values can never disagree:

1. **Leader override** — for a leader, incoming values are discarded and forced to `sms=false, noti=false, ping=true, notiPing=false`. A leader never collects, but still reports that the phone is alive.
2. **Collector fallback** — if both `IsSmsEnabled` and `IsNotiEnabled` are false, `IsSmsEnabled` is forced to true. "Both off" is treated as "never configured", not as "disabled". To actually stop a user, set the status to Disabled.
3. `IsNotiPingEnabled` is deliberately **not** tied to `IsNotiEnabled`.

Defaults at creation: leader as above; `SP` gets `sms=false, noti=true, ping=true, notiPing=true`; `INTERNAL` and `RESELLER` get `sms=true, noti=false, ping=true, notiPing=false`.

`POST /api/User/get-user-config` (unauthenticated, called by the app after restart) takes `{ Username, MerchantCode, SystemType }` and returns the same shape as login.

---

## 4. Administration API (called by the BO and the Devsite)

These endpoints are what the BO will keep calling after the migration, because **creating login users on the Devsite and creating leader-balance users stay on the BO side**. They are the most important contract in this document after deposit confirmation.

Authentication today is a header pair: `X-BO-DateTime` (format `yyyyMMddHH`, UTC) plus `X-BO-Signature = MD5(sharedSecret + thatDateTimeString)`, accepted within a two-hour window. The same signature is used for reads and writes, and on one page it is **rendered into the browser**. The new backend must replace this with a proper service-to-service scheme.

| Endpoint | Request | Notes |
|---|---|---|
| `POST /api/User/create` | `Username, Password, MerchantCode, MerchantId, SystemType, UserType, IsUseLandingPage, Is*Enabled?` | password arrives **in plain text**; rules: 6..16 chars, at least one uppercase and one special character. Duplicate usernames are rejected except against deleted users. Returns `{ Id, Username, Message }` — the BO stores that `Id` as a foreign key |
| `POST /api/User/change-status` | `UserId, Status` | no transition validation |
| `POST /api/User/change-password` | `UserId, NewPassword, SecretKey` | plain text; additionally requires the shared secret in the body |
| `POST /api/User/change-user-config` | user id plus nullable flags | null means "leave unchanged"; if no flag is supplied only the landing-page flag is written |
| `POST /api/User/change-user-config-bulk` | up to 1000 ids plus flags | single bulk statement |
| `POST /api/User/change-user-config-by-master-code` | master code plus flags | applies to a whole tenant |
| `POST /api/User/fetch` | `MerchantCode?, Username?, Id?, SystemType?, UserType?, Status?, Page, Limit` | limit clamped to 1..500, newest first, password column never selected, flags returned are effective |
| `GET /api/User/fetch-active-master-codes` | — | distinct master codes with active users |

**Required change in the new backend**: stop accepting plain-text passwords. Accept a hash, or issue an invite/reset link. The BO also stores a leader password in clear today and will remove that when the new API allows it.

---

## 5. Leader accounts

Ownership is split:

| Side | Holds |
|---|---|
| **BO** | leader business record: name, currency, bank codes, status, limit mode, total limit, balance type, master merchant, and a pointer to the user id in the SMS service |
| **SMS service** | credentials and feature flags (the user row) plus leader device heartbeats |

Synchronisation is one-way, BO to service:

| BO action | Call |
|---|---|
| create leader | `POST /api/user/create`, the returned id is stored on the leader record |
| change password | `POST /api/user/change-password` |
| disable or delete | `POST /api/user/change-status` |
| failed bulk import | rollback by setting the created user to Deleted |

There is also a cleanup routine for orphaned service users. The new backend must keep a **stable user id** so this foreign key survives migration.

---

## 6. Chat and tokens — planned, not in production

On branch `feature/livechat-full-sync-260810` (not merged) login also issues:

* an **HS256 JWT**, issuer `smsservice`, audience `chatservice`, default lifetime 24 h, claims `sub` (`user-{id}`), `phone`, `mid` (merchant code), `role` (`agent` or `leader`, always derived server-side), optional `did` (device id) and `name`;
* a **refresh token**: random, stored only as a SHA-256 hash, one year lifetime, single use with rotation, revoked on logout, password change, disable, or a newer login; a background job removes expired rows.

The chat service verifies the token by selecting a secret based on the issuer (it accepts a second issuer for BO staff tokens), requires the audience `chatservice`, and requires `sub`, `mid` and `phone` to be present. It also uses the token's issued-at time to decide which of two sessions for the same user is newer.

If chat ships on the new backend, these claims and this behaviour must be preserved, or the chat service must be changed at the same time.

---

## 7. Operator features (answer to question 5)

Everything below is functionality that exists **outside the app's own API surface**. All of it must exist somewhere after the migration — either as an API the BO calls, or as a screen in the new backend's own console. The BO team's preference is stated in the last column.

| # | Feature | Where it lives today | How it is used | Preferred future |
|---|---|---|---|---|
| 1 | **Account SMS** page | BO page, browser calls the service directly | look up parsed statements for one collecting account: filter by ref code, bank, phone, date range, validity | API, called by the BO server |
| 2 | **All Account SMS** page | BO page, direct browser call | same across a whole merchant, plus amount filter and Excel export | API + export |
| 3 | **Public SMS Manage / Message** pages | Devsite, direct browser call | inspect parsed statements and **raw** SMS bodies; run a parser test on a sample body | API |
| 4 | **Device** page | BO page, direct browser call | device list per merchant with connectivity, plus Excel export | API |
| 5 | **Device Manage** page | Devsite, direct browser call | device list with device id and UI version, plus **force reset all devices for a phone** | API, authenticated |
| 6 | **SMS User** page | Devsite, direct browser call | create users, enable/disable, change password, change feature flags (single, bulk, or by master code) | API, §4 |
| 7 | **SMS Template** page | Devsite, through a BO proxy | create and edit regex templates and sender masks, test a pattern against a sample | either an API or a screen in the new console |
| 8 | **SMS Parser** page | Devsite | ad-hoc "what would this SMS parse to" | screen in the new console is fine |
| 9 | **SMS Monitor** page | BO | ingest counts and daily series per merchant, used to spot a tenant that stopped collecting | metrics API |
| 10 | **Scam / suspicious SMS review** | BO calls the service | intended to list flagged statements and approve them back into use. **Currently returns nothing**, because the balance check does not persist its result (`01-sms-ingest.md` §6) | API, once the outcome is stored. Confirm with operations that the review step is still wanted |
| 11 | **Telegram bot: OTP lookup** | service endpoint | fetch the newest SMS body for a phone within the last N seconds, so support can read an OTP without touching the phone | API |
| 12 | **Telegram bot: device reset** | BO to service, signed | reset a device from chat during an incident | API, keep it signed |
| 13 | **Export jobs** | BO scheduler to a dedicated export host | scheduled Excel exports of confirmed SMS per merchant, uploaded to object storage | API or scheduled export in the new backend |
| 14 | **Stale device sweep** | scheduler to the service | clears the connected flag | can be derived, see `03-device-ping-sim.md` §6 |
| 15 | **Deposit-confirm configuration** | BO table, pushed to the service | per master merchant and bank: confirm enabled, minimum and maximum confirmable amount | stays on the BO; see `05-deposit-confirm.md` |
| 16 | **Notice management** | BO pages to the service | compose, edit, resend push notifications and read delivery statistics | API, see `02-notifications.md` |
| 17 | **JMS Manage (app version)** | BO/Devsite page | set the published version, minimum version, force update, force logout; upload the APK | stays on the BO; the new backend polls it |
| 18 | **Environment check** | service endpoint | dumps the effective host and version configuration to a Telegram group during incidents | keep an equivalent, authenticated |

Points 1 to 6 are today **called straight from the browser** to the service. That will be replaced by BO server-side proxying, so the new backend only needs to serve **service-to-service** calls, not browser traffic, and does not need CORS for BO pages.
