# 01 · Principles

**For:** anyone about to add a decision.
**This file answers:** which rules a later page is not allowed to break.
**The overview is:** `00-index.md`. Do not expect the ping walkthrough here.

```mermaid
flowchart LR
    P["1. Ping"] --> L["2. Login"] --> S["3. SIM"] --> I["4. SMS and notification ingest"]
```

## Plan rules

This folder is the only plan. Do not cite or quote another plan. If a decision is not in these files, it is not decided.

When a later file disagrees with this one, edit this one first.

## Product rules

The Go service is the server the phone talks to. The Back Office still chooses which account can take a deposit, and it still credits the player.

Internal and Reseller are separate deployments. They do not share a database, a host, or a config value.

Build in this order. Ping does not wait for the later items.

1. Device ping — `04-device-ping.md`
2. User login — `05-user-login.md`
3. SIM binding — not written
4. SMS and notification ingest — not written

Deposit confirmation, outbound push, B2B, operator screens, and retention have no file yet. Do not specify them inside the files that do exist.

## Behaviour rules

These are the constraints. The steps live in the file named on the right.

| Rule | Where the steps are |
|---|---|
| A new ping records the heartbeat and does not call onward. The old ping is the onward call. Device, leader, and notification pings share that split. The flag is the one in the index. | `04-device-ping.md` |
| Ping does not accept or reject an app version. | Version API, not written |
| Login is forwarded. This service issues tokens and does not store a second user. Ping does not require a token. | `05-user-login.md` |
| One SIM is bound to one device. The secret on the row is opaque. Ping does not create device rows. | `06-sim.md`, not written |
| The new ping is signed. The phone and device id in a signed body are the values as sent, not a normalised form. | `04-device-ping.md` for ping. `06-sim.md` for SIM, not written |
