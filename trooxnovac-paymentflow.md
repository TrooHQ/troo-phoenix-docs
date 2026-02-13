# Troo × NovacPay — Android POS Payment Flow (Card Reader + Encrypted Card Charge)

## Overview

Troo’s Android POS supports a **card-first checkout flow** using a **hardware card reader**. Since Novac’s SDK is not used for reader processing, Troo:

1) Creates a Novac checkout reference  
2) Reads card details from the device card reader (via hardware SDK)  
3) Encrypts card payload using Novac `/api/v1/encrypt-data`  
4) Charges card using Novac `/api/v1/card-payment`  
5) Confirms final status via Novac **webhook + verify**  
6) Falls back to **Transfer / USSD / QR** if card fails

---

## NovacPay APIs Used (per docs)

- Initialize transaction: `POST https://api.novacpayment.com/api/v1/initiate`
- Encrypt card payload: `POST https://api.novacpayment.com/api/v1/encrypt-data`
- Complete card payment: `POST https://api.novacpayment.com/api/v1/card-payment`
- Webhooks: merchant endpoint receives Novac POST payload
- Verify transaction: `GET https://api.novacpayment.com/api/v1/checkout/{transactionRef}/verify`

---

## Sequence Diagram (Card-first + Fallback)

```mermaid
sequenceDiagram
    participant Cashier as Cashier
    participant Device as Android POS (Troo App)
    participant Reader as Card Reader SDK (Hardware)
    participant Order as Order Service
    participant Pay as Payment Service
    participant Novac as NovacPay

    Cashier->>Device: Build cart / select items
    Device->>Order: Create/Update Order
    Order-->>Device: Order total

    Cashier->>Device: Checkout (CARD default)
    Device->>Pay: Create Payment Intent (channel=CARD)
    Pay->>Novac: POST /api/v1/initiate (transactionReference, amount, currency, customer)
    Novac-->>Pay: transactionReference + payment options

    Device->>Reader: Start card collection (tap/insert/swipe + PIN)
    Reader-->>Device: Card data (PAN/expiry/CVV/PIN)\n(as provided by reader SDK)

    Device->>Pay: Submit card payload (one-time, no storage)
    Pay->>Novac: POST /api/v1/encrypt-data (card fields + reference)
    Novac-->>Pay: Encrypted payload

    Pay->>Novac: POST /api/v1/card-payment (encrypted fields + transactionReference)
    Novac-->>Pay: Immediate response (processing/accepted)

    Novac->>Pay: Webhook event (successful/failed/reversed/abandoned)
    Pay->>Novac: GET /api/v1/checkout/{ref}/verify
    Novac-->>Pay: Verified status (pending/successful/failed)

    alt Verified Successful
        Pay->>Order: Mark Order PAID
        Order-->>Device: Order PAID
        Device-->>Cashier: Success + Receipt
    else Verified Failed
        Pay-->>Device: Payment failed
        Device-->>Cashier: Offer fallback: Transfer / USSD / QR
        Cashier->>Device: Select fallback method
        Device->>Pay: Start fallback payment
        Note over Pay,Novac: Complete via Novac channel APIs, then webhook + verify again
        Novac->>Pay: Webhook
        Pay->>Novac: Verify /checkout/{ref}/verify
        Novac-->>Pay: Verified Successful
        Pay->>Order: Mark Order PAID
        Order-->>Device: Order PAID
        Device-->>Cashier: Success + Receipt
    end
