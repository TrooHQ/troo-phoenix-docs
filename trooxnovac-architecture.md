# Troo × NovacPay — Android POS Architecture (Card Reader + Encrypted Card Charge)

## Overview

This document describes the architecture for integrating **NovacPay** into **Troo’s Android-based checkout hardware**, where:

- The **payment device is Android-based**
- **Card is default**
- If card fails (decline/timeout/reader error), the device offers **Transfer / USSD / QR**
- NovacPay SDK is **not used for reader processing**
- Troo uses the **hardware card reader SDK** + NovacPay **Encryption** + **Card Payment** APIs

Novac provides an **encryption endpoint** (`/api/v1/encrypt-data`) to encrypt sensitive card data before it is used in direct card charge flows.  
Troo will use this to avoid implementing custom encryption and to reduce exposure. (See Novac Encryption docs)

---

## Core Principles

- **Card-first UX** on the device
- **Payment Intent** as the single source of truth
- **Server-authoritative confirmation**: order is marked paid only after webhook + verify
- **No persistence of sensitive card data** (PAN/CVV/PIN) in Troo systems
- **Idempotency** to prevent double charges on retries/timeouts

---

## Components

### 1) Android Checkout Hardware (Troo POS)

**Responsibilities**
- Cashier UI (cart, checkout, payment selection)
- Integrates with **Hardware Card Reader SDK**
- Collects card details from reader (PAN, expiry, CVV, PIN *as made available by reader SDK*)
- Sends card payload securely to Troo Payment Service (or directly to Novac, depending on agreed pattern)
- Shows fallback payment instructions (Transfer/USSD/QR)
- Supports “Pending confirmation” and “Check payment status”

**Security**
- Never stores raw PAN/CVV/PIN
- Never logs sensitive card values
- Uses TLS + certificate pinning (recommended) for backend calls

---

### 2) Troo Core Service

**Responsibilities**
- Device registration & authentication
- Merchant configuration (enabled channels)
- Stores Novac configuration (keys, redirect URL settings, environment)

---

### 3) Troo Order Service

**Responsibilities**
- Cart/order lifecycle
- Total computation (discounts/taxes)
- Order states:
  - `DRAFT`
  - `PENDING_PAYMENT`
  - `PAID`
  - `FAILED`
  - `CANCELLED`

---

### 4) Troo Payment Service

**Responsibilities**
- Create and manage **Payment Intents**
- Integrate with NovacPay:
  - Initialize checkout reference (`/api/v1/initiate`)
  - Encrypt card payload (`/api/v1/encrypt-data`)
  - Submit card charge (`/api/v1/card-payment`)
  - Verify transaction (`/api/v1/checkout/{ref}/verify`)
  - Handle webhooks (IP whitelist + async processing)
- Enforce:
  - idempotency keys
  - one-successful-payment-per-order rule
- Expose payment status to device

**Payment Intent states**
- `CREATED`
- `PROCESSING`
- `REQUIRES_ACTION` (fallback methods)
- `SUCCEEDED`
- `FAILED`
- `CANCELLED`

---

### 5) NovacPay Platform

**Responsibilities**
- Payment processing
- Encryption service (`/api/v1/encrypt-data`)
- Checkout initialization (`/api/v1/initiate`)
- Card completion (`/api/v1/card-payment`)
- Webhooks (notifyType/status)
- Verify transaction (`/api/v1/checkout/{ref}/verify`)

---

## Data Flow Summary

### Card (Default)
Device (reader SDK) → Troo Payment Service → Novac encrypt → Novac card-payment → Novac webhook → Troo verify → Troo marks order PAID

### Fallback
Device → Troo Payment Service → Novac (transfer/ussd/qr completion) → webhook → verify → mark PAID

---

## Webhook Trust Model

Novac advises IP whitelisting and provides a public IP for webhook validation, plus notify types such as `successful`, `failed`, `reversed`, `abandoned`.
Troo Payment Service will:
- validate source IP
- respond HTTP 200 quickly
- process asynchronously
- verify the transaction before marking payment successful

---

## Outcome

This architecture enables:
- Android POS + physical card reader integration
- Card-first experience without Novac reader SDK dependency
- Encrypted card charge flows via Novac encryption service
- Strong reliability and auditability via webhook + verification
