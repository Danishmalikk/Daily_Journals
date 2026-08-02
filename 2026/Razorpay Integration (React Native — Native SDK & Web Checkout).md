# Razorpay Integration — React Native (Native SDK vs Web Checkout + Webhooks)

> Context: React Native CLI project, adding Razorpay payments. Two integration shapes exist — (1) native SDK, checkout happens inside the app, (2) hosted/web checkout, user is sent to a browser. Both need the same backend safety net: signature verification + webhooks. Correcting one thing up front — **webhooks don't "give you" idempotency, they're the reason you *need* idempotency** (see §4). More on this below.

---

## 0. The big picture — order of operations (applies to both flows)

```
1. App asks YOUR backend to create a payment (order / payment link)
2. YOUR backend calls Razorpay API (using key_id + key_secret) → gets order_id
3. App opens Razorpay checkout (native SDK) OR redirects to a hosted URL (web flow)
   → user pays with card / UPI / netbanking / wallet
4. Razorpay tells you payment happened, via TWO independent channels:
      a) client-side callback / redirect  (fast, but NOT trustworthy alone)
      b) server-side webhook              (slower, but the actual source of truth)
5. YOUR backend marks the order paid ONLY after verifying (a) via signature,
   and treats (b) as the final confirmation — reconciling if (a) never arrived.
```

The one thing to internalize: **the order/payment must always be created server-side.** The `key_secret` needed to call Razorpay's Orders API can never live in the mobile app — only `key_id` (public) goes into the RN bundle.

---

## 1. Native SDK flow (`react-native-razorpay`) — the standard mobile approach

This is what you want for a normal buy-flow inside the app (best UX — UPI intent apps like GPay/PhonePe open and return via deep link, no browser tab involved).

**Setup:**
- `npm install react-native-razorpay`, then `pod install` for iOS.
- **Android:** for Android 11+ (API 30+), add a `<queries>` block in `AndroidManifest.xml` listing UPI app packages — otherwise the SDK can't detect installed UPI apps to show them in the sheet (package visibility restriction).
- **iOS:** add `LSApplicationQueriesSchemes` in `Info.plist` for UPI URL schemes (`upi`, `tez`, `phonepe`, `paytmmp`, etc.), and wire up the redirect-back handling in `AppDelegate` so the app resumes after the UPI app hands control back.

**Flow:**
1. App calls **your backend** `POST /create-order`.
2. Backend calls Razorpay's **Orders API** (`POST /v1/orders`) with `amount`, `currency`, `receipt` → gets back `order_id`. This requires `key_secret`, hence server-side only.
3. Backend returns `order_id` to the app.
4. App calls:
   ```js
   RazorpayCheckout.open({
     key: RAZORPAY_KEY_ID,   // public key_id, safe to embed
     order_id,                // from step 2 — binds this checkout to that order
     amount, currency, name, description,
     prefill: { email, contact },
     theme: { color: '#...' },
   })
     .then(data => { /* razorpay_payment_id, razorpay_order_id, razorpay_signature */ })
     .catch(err => { /* err.code, err.description — includes user-cancelled */ });
   ```
5. On success, the promise resolves with `razorpay_payment_id`, `razorpay_order_id`, `razorpay_signature`. **This is a client-side value — treat it as unverified input, not proof of payment.**
6. App sends these three values to **your backend**, which verifies:
   ```
   generated_signature = HMAC_SHA256(order_id + "|" + razorpay_payment_id, key_secret)
   valid = (generated_signature == razorpay_signature)
   ```
   Only after this check passes does the backend mark the order as paid. This is what actually proves the payment wasn't forged/tampered — anyone could call your "mark as paid" endpoint with a fake payment_id otherwise.
7. **Capture:** a payment can land in `authorized` state, not `captured`, depending on your account/order config. If auto-capture isn't enabled (`payment_capture: 1` on order creation, or account-level setting), you must call the **Capture API** within the authorization window (5 days) or Razorpay auto-refunds it. Worth checking which mode your account is in — this is a common silent bug ("payment shows authorized forever, money never actually settles").

**Common misconception to avoid:** "I got the success callback in the app, so the payment is done — update the order from the client." Don't do this even for the native SDK flow. The callback can be lost entirely (app crash, force-close, network drop) *after* the user has actually paid — the money moved, but your app/server never heard about it via the callback. That's exactly why webhooks (§3) exist as the backend's real source of truth, for native SDK too, not just the web flow.

---

## 2. Web-based / hosted checkout flow — browser redirect

This is the right call when you're using **Payment Links**, a hosted subscription page, or any flow where you deliberately want the user outside the app (e.g. a link sent over SMS/WhatsApp/email, or you don't want to maintain native SDK plumbing for a secondary flow).

**Flow:**
1. Backend creates a **Payment Link** (or hosted order checkout URL) via Razorpay's API, configuring a `callback_url` (redirect target) and `callback_method`.
2. App opens that URL — **use an in-app browser (Custom Tabs on Android / `SFSafariViewController` on iOS, e.g. via `react-native-inappbrowser-reborn`), not a bare `Linking.openURL`**, so you keep a usable UX and can catch the redirect back into the app via a deep link / universal link.
3. User completes payment on Razorpay's hosted page.
4. Razorpay redirects the browser to your `callback_url` with payment details attached.
5. **This redirect is not trustworthy as proof of payment either** — same reasoning as the SDK callback, plus it's easier to tamper with (a user could hand-craft the redirect URL/deep link with fake params) or the user could just close the tab before the redirect fires, leaving your app in the dark. Redirect = "hey, probably done, go check" — not "payment confirmed."
6. Backend treats the redirect the same way as the SDK success payload: extract the payment id, verify signature server-side, and/or just wait for confirmation via webhook before crediting anything.

So the correction to your original framing: routing to the browser is right for this flow, but **the redirect itself doesn't establish trust** — it's still webhook + signature verification doing the actual confirming, same as the native flow. The browser-vs-native choice is a UX/architecture decision, not a security one; the security model (don't trust the client, verify server-side) is identical either way.

---

## 3. Webhooks — the actual source of truth

A webhook is Razorpay's backend calling **your** backend directly (server-to-server), independent of whatever happened on the user's device. Configure the endpoint + a **webhook secret** in Razorpay Dashboard → Webhooks (this secret is different from your account's `key_secret`).

**Key events to handle:**
- `payment.authorized` / `payment.captured` — payment succeeded (captured = money actually settled).
- `payment.failed` — payment attempt failed.
- `order.paid` — order fully paid (useful if an order can be paid in parts).
- `refund.processed` — refund completed.

**Verifying a webhook (different formula from checkout verification — different secret too):**
```
expected_signature = HMAC_SHA256(raw_request_body, webhook_secret)
valid = (expected_signature == header['X-Razorpay-Signature'])
```
Important: hash the **raw request body bytes**, before any JSON parsing/re-serialization — re-serializing and hashing that can produce a different string (key order, whitespace) and make a legitimate webhook fail verification.

---

## 4. Idempotency — correcting the framing

Your instinct that idempotency matters here is right, but the causal direction is backwards. It's not "webhooks maintain idempotency" — it's:

> **Webhooks are delivered at-least-once, not exactly-once.** Razorpay retries delivery (with backoff, over a window of time) whenever your endpoint doesn't respond with a 2xx. So your handler *will* receive the same event more than once in normal operation — a slow response, a transient 500, a network blip on your end all trigger a legitimate retry. Because of that, **you** have to make the handler idempotent — webhooks are the reason idempotency is required, not the mechanism that provides it.

**How to actually implement it:**
- Use the payment/order id (or the webhook's `event.id` if you store one) as a dedupe key. Two common patterns:
  - A `payments` table with a **unique constraint** on `razorpay_payment_id` — a second insert for the same payment just fails/no-ops instead of double-crediting.
  - A guarded update: `UPDATE orders SET status='paid' WHERE id=? AND status != 'paid'` — running it twice is harmless, only the first call actually changes anything.
- Always return **2xx quickly** once you've durably recorded the event (even if downstream processing like sending a confirmation email happens async) — a slow handler is what triggers unnecessary retries in the first place.
- This is a separate concept from **Razorpay's own idempotency keys** (`X-Razorpay-Idempotency-Key` header, supported on some of their POST APIs like capture/refund) — that mechanism protects *you* when *you* retry a call *to* Razorpay (e.g. your capture request times out and you retry it — the idempotency key stops Razorpay from double-capturing). Two different directions, easy to conflate: webhook idempotency = protecting your DB from Razorpay's retries; API idempotency key = protecting Razorpay's ledger from your retries.

---

## 5. Signature verification cheatsheet (two different formulas, two different secrets)

| | Checkout / SDK verification | Webhook verification |
|---|---|---|
| When | Right after `RazorpayCheckout.open()` resolves, or after the browser redirect | On every incoming webhook POST |
| Input | `order_id + "|" + payment_id` | Raw request body |
| Secret | Account `key_secret` | Separate **webhook secret** (Dashboard → Webhooks) |
| Header/field | `razorpay_signature` returned in the client payload | `X-Razorpay-Signature` request header |
| Proves | This specific checkout session's payment wasn't forged by the client | This specific HTTP call genuinely came from Razorpay |

Both use HMAC-SHA256. Doing only one of these and skipping the other is the most common shortcut that quietly breaks the security model — client-side-only verification trusts the device, webhook-only misses the case where you want instant UI feedback.

---

## 6. Checklist

- [ ] Order (or payment link) is always created **server-side** — `key_secret` never ships in the RN bundle, only `key_id`.
- [ ] Client success callback / redirect is treated as "probably done, go verify" — never as proof by itself.
- [ ] Backend verifies the checkout signature (`order_id|payment_id` HMAC) before marking anything paid.
- [ ] Webhook endpoint configured, with its **own** secret, verifying `X-Razorpay-Signature` against the **raw** body.
- [ ] Webhook handler is idempotent (unique constraint or guarded update) — because retries are expected, not exceptional.
- [ ] Webhook handler returns 2xx fast; heavy side effects (emails, notifications) happen async, not inline before the response.
- [ ] Know whether your account/order auto-captures — if not, capture explicitly within the 5-day authorization window.
- [ ] Android: `<queries>` block for UPI app visibility (API 30+). iOS: `LSApplicationQueriesSchemes` + `AppDelegate` redirect handling.
- [ ] Web/browser flow uses an in-app browser (Custom Tabs / SFSafariViewController) with a deep link back, not a bare external `Linking.openURL` dead end.
- [ ] Test mode keys during development; Razorpay's test cards/UPI ids for sandbox runs; test webhook delivery via the Dashboard's "Test Webhook" button or a tunnel (ngrok) to local.
- [ ] Reconciliation path exists for the rare case where the webhook itself is delayed/lost too — e.g. a periodic job that checks Razorpay's Payment/Order fetch API for anything stuck "created"/"authorized" past some threshold.

---

## Interview one-liner

> "We support two Razorpay flows on React Native — native SDK checkout for the primary in-app buy flow, and hosted/browser checkout for payment links. In both cases the order is created server-side, the client-side success callback or redirect is treated as untrusted, and the backend verifies the checkout signature before marking anything paid. Webhooks are the real source of truth on top of that — and because Razorpay delivers them at-least-once, not exactly-once, the webhook handler has to be idempotent, keyed on the payment id, rather than webhooks providing idempotency themselves."
