# Genuine UPI checkout setup

The project intentionally includes **no QR image, UPI ID, or fabricated payment destination**. Checkout cannot create a request until the owner supplies the real details and the configured image loads in the deployed app.

## 1. Add the real QR

1. Export/download the genuine QR from the owner’s actual UPI application. Do not use a generated placeholder QR.
2. Put the image in `public/assets/payments/`, for example `public/assets/payments/owner-upi-qr.png`.
3. Keep it a public payment QR only. Never put PINs, OTPs, private tokens or credentials into assets.
4. Commit the asset securely, deploy it to the staging site, and confirm that `/assets/payments/owner-upi-qr.png` loads directly in a browser.
5. In `/admin` → **Payment settings**, enter that exact local path, the exact verified recipient name, and the actual UPI ID. Save; the change is server-validated and audited.

## 2. Confirm before collecting money

- Compare displayed recipient name and UPI ID with the payment app’s actual account.
- Use a low-risk internal test or another payment method in staging without granting production entitlements.
- Checkout shows the server-calculated total and billing period. Do not pay if amount or recipient differs.
- The customer creates a pending request, completes payment separately, and submits the transaction reference and optional date/time. The request stays pending and paid APIs remain locked.
- A reviewer opens the actual UPI/payments record and confirms recipient, amount, transaction reference and receipt. The admin approval form requires the exact verified amount, matching reference and a confirmation checkbox, plus a review note.
- Approval activates only the plan tied to that request, using snapshotted duration/features/modules/support priority, and records the reviewer, timing, subscription and customer notification.
- The target review time displayed is 1–2 hours subject to admin availability—not a guarantee.

Do not use a screenshot or transaction reference alone as conclusive evidence. This version does not independently query a payment provider and does not upload payment screenshots. Never ask customers for UPI PINs, bank passwords, OTPs, or account passwords.

## 3. Payment limitations

No automatic renewal, refunds, bank reconciliation, webhook gateway, or recurring payment engine is included. A new order after an existing subscription is scheduled for the end of the currently recorded term; no mid-term proration/credit is applied. The checkout displays the expected start date, and the backend calculates the final date again during approval.
