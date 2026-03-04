# Feature Documentation

# Feature: Payment via HyperPay

---

## 1. Feature Overview

This feature enables users to complete order payments using the **HyperPay payment gateway** directly within the mobile application.

The payment process is initiated from the **Checkout screen** after the user confirms their order.

The application communicates with the backend to create a payment checkout session, then launches the **HyperPay SDK** where the user enters card details and completes the transaction.

Supported payment methods:

- Visa
- Mastercard
- Mada

The final payment status is always verified through the backend to ensure the transaction was processed successfully.

---

## 2. Flow (Workflow)

1. User opens the **Checkout screen**.
2. User selects **HyperPay** as the payment method.
3. Application validates order state and required user data.
4. Application fetches **billing address** from backend.
5. Application formats order amount according to payment gateway requirements.
6. Application sends request to backend to create payment checkout.
7. Backend returns `checkout_id`.
8. Application opens **HyperPay SDK** with the checkout session.
9. User enters card details.
10. SDK processes payment.
11. SDK returns payment result.
12. Application calls backend API to verify payment status.
13. Order status is updated accordingly.

---

## 3. Validation Rules

The following validations must be satisfied before initiating payment.

### User validation

- User must be authenticated.
- User must have a valid **billing address**.

Required billing address fields:

- full_name
- country
- city
- address_line1
- phone_number
- postal_code (required depending on country)

If billing address is incomplete, payment cannot be initiated.

---

### Order validation

Payment is allowed only if order status is:

`PENDING_PAYMENT`

Payment must NOT start if order status is:

- CANCELLED
- PAID
- FAILED

---

### Amount formatting rules

Payment amount must follow strict formatting rules required by the payment gateway.

Rules:

- Amount must be greater than **0**.
- Amount must not contain more than **2 decimal places**.
- Amount must not be sent as a **floating point value**.

Incorrect formatting may cause the payment request to fail.

Example valid values:

| Display Amount | Minor Units |
|---------------|-------------|
| 100.00 | 10000 |
| 10.50 | 1050 |
| 5.99 | 599 |

Invalid values:

- 10.555 → invalid
- 10.1 → must be formatted as 10.10
- float values like `10.499999`

All amounts must be converted to **minor units** before sending to backend.

Example:
minor_amount = round(amount * 100)

Developers must always use the shared utility:

`AmountFormatter.formatToMinorUnits()`

---

## 4. Important Cases / Edge Cases

### User cancels payment

If the user closes the HyperPay payment screen:

- Payment process is cancelled.
- Order remains in `PENDING_PAYMENT`.
- User can retry payment.

---

### Checkout expiration

Payment checkout sessions expire after a specific time.

If checkout expires before payment completion:

- Payment attempt fails.
- Application must request a **new checkout session** from backend.

---

### Network interruption during verification

If network connection fails during payment verification:

- Application must retry verification.
- Order status must NOT be updated until verification succeeds.

---

### Payment success but verification pending

In some cases the SDK may return success but backend verification returns `PENDING`.

In this case:

- App should display **Processing payment**
- Retry verification until final status is received.

---

## 5. Notes for Developers

- Do NOT rely on SDK response alone.
- Always verify payment status via backend API.
- Do NOT format amounts using `float` or `double`.
- Always use the shared utility for amount formatting.

Incorrect amount formatting is one of the most common reasons for payment failure.

---

## 6. APIs Used

### Create Checkout

Endpoint:

`POST /payments/checkout`

Purpose:

Creates a new payment checkout session.

Important request fields:

- order_id
- amount_minor
- currency
- billing_address

Important response fields:

- checkout_id
- expires_at

---

### Verify Payment

Endpoint:

`GET /payments/verify`

Purpose:

Verify the final payment result.

Important response fields:

- payment_status
- transaction_id
- result_code

Possible values for `payment_status`:

- SUCCESS
- FAILED
- PENDING

---

## 8. Preconditions

Before starting payment:

- User must be authenticated
- Billing address must exist
- Order must be valid
- Order amount must be formatted correctly

---

## 9. Platform / SDK Constraints

### iOS Limitation

The HyperPay iOS SDK does not allow editing the billing address within the payment interface.

Because of this limitation:

- Billing address must be fetched from backend before launching the SDK.
- Users must update incorrect address information from the **Profile / Address screen**.

If billing address is missing, payment must not start.

---

## 10. Failure Scenarios

Possible payment failures include:

- Card declined by issuing bank
- Invalid amount formatting
- Expired checkout session
- Network failure during payment verification

---

## 11. Error Handling

Application handles errors using the following approach:

- Display user-friendly error message
- Allow user to retry payment
- Log payment errors for debugging

---

## 12. Dependencies

This feature depends on:

- Orders module
- Cart module
- User profile service
- Payment backend service
- HyperPay SDK
