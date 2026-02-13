# Troo × NovacPay — Android POS Payment Flow

## Overview

This document describes the **end-to-end payment flow** for Troo’s Android POS system using **NovacPay**, with **card payments as the default** and **Transfer / USSD / QR** as fallback options.

All payment outcomes are **verified server-side** before an order is fulfilled.

---

## Actors

- Cashier
- Customer
- Android POS Device (Troo App)
- Troo Backend (Order & Payment Services)
- NovacPay Platform

---

## High-Level Flow

1. Cashier creates order
2. Payment Intent is created
3. Card payment is attempted (default)
4. If card fails → fallback channels offered
5. NovacPay sends webhook
6. Troo verifies transaction
7. Order is finalized

---

## Sequence Diagram (Card-First with Fallback)

```mermaid
sequenceDiagram
    participant C as Cashier
    participant D as Android POS (Troo)
    participant O as Order Service
    participant P as Payment Service
    participant N as NovacPay

    C->>D: Create order
    D->>O: Create/Update Order
    O-->>D: Order total

    C->>D: Checkout
    D->>P: Create Payment Intent (CARD)
    P->>N: Initiate Checkout
    N-->>P: transactionReference

    D->>N: Launch Android SDK (Card)
    N-->>D: SDK Result (Approved / Failed)

    alt Card Approved
        N->>P: Webhook (successful)
        P->>N: Verify Transaction
        N-->>P: Verified = Success
        P->>O: Mark Order PAID
        O-->>D: Order PAID
        D-->>C: Show Success + Receipt
    else Card Failed
        D-->>C: Show Fallback Options
        C->>D: Select Transfer / USSD / QR
        D->>P: Request Fallback Instructions
        P->>N: Initiate Fallback Payment
        N->>P: Webhook
        P->>N: Verify Transaction
        N-->>P: Verified = Success
        P->>O: Mark Order PAID
        O-->>D: Order PAID
        D-->>C: Show Success + Receipt
    end
