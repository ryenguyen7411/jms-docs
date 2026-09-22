# 06 · SIM

**Status:** outline.
**Depends on:** `01-principles.md`, `04-device-ping.md`.
**Build after:** `05-user-login.md`.

## 1. Identity

One row per SIM phone number and device id. The secret is opaque. Logged-in is separate from "pinged recently". Ping reads this row. It does not create it.

## 2. Activation

First bind issues a secret, encrypted to the phone's public key. The same device activating again gets the same secret and does not change login state. A different device is rejected until reset. Reset, then activate, issues a new secret and the old one stops working immediately.

## 3. Account check

Before creating a row, ask the BO whether the SIM belongs to that merchant. "Not registered" and "BO unavailable" are different errors.

## 4. Signature compatibility

The signed string is phone plus device id exactly as sent. The row is found with the normalised phone. State the check against real rows before this path is trusted.

## 5. Sign-out and reset

Sign-out is signed with the device secret and clears device id, secret, and push token. Every reset path is authenticated and audited. Reset from chat stays, and it is signed. Unauthenticated reset does not exist.

## 6. Done when

A SIM can bind, re-bind on the same phone, and move to a new phone only after reset, and uploads signed with the previous secret fail after that reset.
