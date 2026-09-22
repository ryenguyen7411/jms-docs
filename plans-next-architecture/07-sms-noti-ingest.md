# 07 · SMS and notification ingest

**Status:** outline.
**Depends on:** `01-principles.md`, `06-sim.md`.
**Build after:** `06-sim.md`.

SMS and bank-app notifications are both ingested here. They do not share a parser or a dedup key. They do share the pipeline shape: accept, persist, acknowledge, then process on a worker that survives restarts.

## 1. SMS request

One SMS per call. Bare result code. Which codes stay. A rejected sender and a duplicate are success for the app until the app changes with us.

## 2. SMS checks, in order

Payload, phone normalisation, sender allow-list, device logged in, signature, then store. The signature input is the one in `06-sim.md`.

## 3. SMS parse

Hard-coded bank cases, then templates. Direction of money applied on every hard-coded case. Invalid transfer time is rejected, not stored as an empty date. Template amounts have no sign.

## 4. SMS after parse

Balance check stored as valid, suspect, or approved. Unparsed messages delivered durably onward. An alert is not that delivery.

## 5. Notification request

Title, body, package name, device, phone, signature over the raw body before any normalisation. Duplicate is success.

## 6. Notification processing

Route by package name. Dedup on a canonical event id. Park, then process one at a time. Retry with backoff. Keep the row and the last error when retries are exhausted.

## 7. Time

Device timestamp, server receive time, and the time parsed from the message are separate facts. Do not overload one field with two meanings.

## 8. Done when

A restart cannot lose a message that was already acknowledged, and the same SMS or notification posted twice does not create two statements.
