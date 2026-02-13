# Troo × NovacPay — Android POS Architecture

## Overview

This document describes the high-level architecture for integrating **NovacPay** into **Troo’s Android-based checkout hardware**.  
The system is designed to support **card-present payments as the default**, with **Transfer, USSD, and QR** as fallback payment channels.

The architecture prioritizes:
- POS-grade reliability
- Minimal PCI exposure
- Clear separation of responsibilities
- Server-authoritative payment confirmation

---

## Core Design Principles

- **Card-first UX** on the device
- **Payment Intent** as the single source of truth
- **Backend verification required** before order fulfillment
- **No raw card data** handled or persisted by Troo systems
- **Event-driven reconciliation** via webhooks

---

## System Components

### 1. Android Checkout Hardware (Troo POS)

**Responsibilities**
- User interface for cashier
- Card reader interaction
- Payment method selection & fallback handling
- Display of instructions (USSD, Transfer, QR)
- Polling for payment status if needed

**Key Characteristics**
- Android application
- Integrated EMV/contactless card reader
- Uses **NovacPay Android SDK**
- Does not store PAN/CVV or sensitive card data

---

### 2. Troo Core Service

**Responsibilities**
- Device registration and authentication
- Merchant and location management
- Staff user management
- Payment channel configuration per merchant

**Key Data**
- Device credentials
- Enabled payment channels
- Environment configuration (test/live)

---

### 3. Troo Order Service

**Responsibilities**
- Cart creation and updates
- Pricing calculations (items, discounts, taxes)
- Order lifecycle management

**Order States**
- `DRAFT`
- `PENDING_PAYMENT`
- `PAID`
- `FAILED`
- `CANCELLED`

---

### 4. Troo Payment Service

**Responsibilities**
- Create and manage **Payment Intents**
- Integrate with NovacPay APIs
- Handle webhooks and transaction verification
- Enforce idempotency and anti-double-charge rules
- Expose payment status to the device

**Payment Intent States**
- `CREATED`
- `PROCESSING`
- `REQUIRES_ACTION`
- `SUCCEEDED`
- `FAILED`
- `CANCELLED`

---

### 5. NovacPay Platform

**Responsibilities**
- Payment processing (Card, Transfer, USSD, QR)
- Android SDK for payment initiation
- Transaction verification APIs
- Webhook notifications
- Settlement and reconciliation

---

## Security & PCI Considerations

- Card data entry and encryption handled entirely by **NovacPay SDK**
- Troo backend never receives raw PAN/CVV
- Webhooks are IP-restricted and verified
- All transactions verified server-side before fulfillment
- Device credentials are scoped and revocable

---

## Communication Summary

| From | To | Purpose |
|----|----|----|
| Device | Order Service | Create/update cart |
| Device | Payment Service | Create payment intent |
| Device | Novac SDK | Card-present transaction |
| Payment Service | Novac API | Initiate, verify, reconcile |
| NovacPay | Payment Service | Webhooks |
| Payment Service | Order Service | Mark order paid/failed |

---

## Architecture Outcome

This architecture enables Troo to:
- Support high-volume POS transactions
- Safely accept card-present payments
- Gracefully recover from network or device failures
- Extend easily to additional payment channels or providers
