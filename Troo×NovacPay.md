# Android POS – Card-First Payment Flow

## Document Purpose

This document explains how **Troo** intends to integrate **NovacPay** into our **Android-based checkout hardware** to support **card-present payments as the default**, with fallback payment channels (**Transfer, USSD, QR**) when card payments fail or are unavailable.

Our goal is to align early with **NovacPay** on:

- Android SDK + card reader support  
- Backend API interactions  
- Webhook and verification flow  
- Security and PCI scope boundaries  

---

## 1. Target Experience (High-Level)

- Cashier initiates checkout
- Card payment is prompted by default
- If card payment fails, times out, or is declined:
  - Device seamlessly offers **Transfer / USSD / QR**
- Payment confirmation is **always server-verified**
- Order is completed **only after successful confirmation**

---

## 2. System Architecture Overview

### On-Device (Android Checkout Hardware)

- Android app (Troo POS)
- Integrated card reader (EMV / contactless / PIN)
- NovacPay Android SDK (preferred)
- Secure local storage (no PAN/CVV persistence)

### Backend Services

#### Troo Core Service
- Device registration & authentication
- Merchant configuration (enabled channels)

#### Troo Order Service
- Cart & order lifecycle

#### Troo Payment Service
- Payment intents
- NovacPay API integration
- Webhook handling & transaction verification

### External

#### NovacPay Platform
- Payment processing
- SDK + APIs
- Webhooks & verification endpoints

---

## 3. Default Card-First Flow (Android SDK Path – Preferred)

### Step-by-Step Process Flow

#### A. Order & Intent Creation

- Cashier creates order on device
- Device → Order Service: calculate total
- Cashier taps **Checkout**
- Device → Payment Service:
  - Create Payment Intent
  - `channel = CARD` (default)

#### B. Card Payment (Primary Path)

- Device automatically launches **NovacPay Android SDK**
- UI prompt:

  > “Insert / Tap / Swipe Card”

- Card reader performs:
  - EMV / contactless
  - PIN entry (if required)

- SDK processes transaction and returns:
  - Transaction reference
  - Preliminary status

- Payment Service:
  - Records provider reference
  - Waits for webhook
  - Verifies transaction via Novac **verify** endpoint

- Payment Intent → **SUCCEEDED**
- Order → **PAID**
- Device displays success + receipt

---

## 4. Card Failure → Fallback Flow (Critical)

### Failure Scenarios

- Card declined
- PIN failure
- Reader error
- Network timeout
- SDK error

### Fallback UX Logic (On Device)

If **CARD** fails, device immediately shows:

> **Payment Failed**  
> Choose another payment method:
> - Bank Transfer  
> - USSD  
> - QR Code  

No need to recreate the order or payment intent.

---

## 5. Fallback Channels Flow

### 5.1 Bank Transfer (Virtual Account)

- Cashier selects **Transfer**
- Device → Payment Service:
  - Request virtual account from NovacPay
- Device displays:
  - Bank name
  - Account number
  - Amount
- Customer transfers
- NovacPay → Webhook
- Payment Service verifies transaction
- Order → **PAID**

---

### 5.2 USSD

- Cashier selects **USSD**
- Payment Service requests USSD code
- Device displays:
  - USSD string (e.g. `*737*...#`)
  - Countdown timer
- Customer dials on phone
- Webhook + verify
- Order → **PAID**

---

### 5.3 QR / NQR

- Cashier selects **QR**
- Payment Service generates QR payload
- Device displays QR
- Customer scans & pays
- Webhook + verify
- Order → **PAID**

---

## 6. Webhooks & Verification (Server-Authoritative)

### Webhook Handling (Payment Service)

NovacPay sends events such as:

- `successful`
- `failed`
- `reversed`
- `abandoned`

Troo:

- Acknowledges webhook immediately (`HTTP 200`)
- Asynchronously calls **Verify Transaction API**
- Updates payment intent status

### Verification Rule

> **No order is marked PAID until verification succeeds**, even if webhook says “successful”.

---

## 7. Transaction States

### Payment Intent States

- `CREATED`
- `PROCESSING`
- `REQUIRES_ACTION` (USSD / Transfer / QR)
- `SUCCEEDED`
- `FAILED`
- `CANCELLED`

### Order States

- `DRAFT`
- `PENDING_PAYMENT`
- `PAID`
- `FAILED / CANCELLED`

---

## 8. Android-Specific Expectations from NovacPay

### SDK & Hardware

We would like clarity on:

- Supported Android card readers
- EMV / contactless certification requirements
- PIN handling (on reader vs on device)
- Does the SDK:
  - Fully complete the transaction?
  - Or return a token/reference for backend confirmation?

### Security

- Confirmation that raw PAN/CVV is never exposed
- Encryption handled entirely within the SDK
- Key injection & device binding process

---

## 9. Reliability & Edge-Case Handling

### Network Loss

- Device shows **“Pending confirmation”**
- Cashier can tap **Check Payment Status**
- Payment Service verifies via Novac API

### Double-Charge Prevention

- Idempotency keys per payment intent
- One order ↔ one successful payment rule

### Reversals

- If SDK reports failure but payment later succeeds:
  - Payment Service reconciles via verify endpoint
  - Order updated accordingly
- If supported, use Novac reversal / void APIs

---

## 10. What We Want to Confirm in the Meeting (Key Questions)

### Android SDK
- Does it support card-present POS transactions?
- Supported reader models?

### Transaction Ownership
- Is the SDK final authority, or should backend always verify?

### Fallback Channels
- Best practice to reuse the same transaction reference across channels?

### Webhooks
- Production IP ranges
- Signature verification support

### Settlement
- Merchant sub-accounts?
- Settlement timelines & reporting APIs?

---

## 11. Summary

- Troo will use **NovacPay Android SDK** for card-present payments as the default
- All payments are verified server-side before fulfillment
- **Transfer / USSD / QR** serve as seamless fallbacks
- Architecture minimizes PCI exposure and ensures POS-grade reliability
