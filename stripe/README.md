# Osysharp.Payments.Stripe — card payments, in a few lines of Osy#

Take money through [Stripe](https://stripe.com) — hosted Checkout or your own card form, customers, refunds, and
**verified webhooks** — from classes written entirely in Osy# over the public `Http.*` surface.

## Use case

Any app that charges for something: a shop checking out a basket, an invoice with a "pay now" button, a booking that
takes a deposit and captures it on arrival, a subscription sign-up. The app says what to charge and for what; Stripe
does the card handling, the 3-D Secure, the Apple Pay and the PCI surface.

It has no user interface of its own — the payment page is Stripe's, and you redirect to it — so there is nothing to
screenshot. What you see is Stripe's checkout.

## What the package door buys a payments integration

`npm i stripe` gives you a library with unrestricted network access and a transitive dependency graph nobody reads.
This kit declares the one host it may reach, and an app that uses it grants that host **or does not compile**:

```osy
package Osysharp.Payments.Stripe { egress "api.stripe.com"; }
```

There is no expression in Osy# that lets the package reach anywhere else. For the one integration in your app that
handles money, *"it can provably only talk to Stripe"* is a property worth having mechanically rather than by review.

It asks for **no secret**. The API key is a setting on the client, so your app names its own — this package never
learns which.

## Install and use

```osy
// app.osy
app Shop {
  model "**/*.osy";
  use Osysharp.Payments.Stripe@0 { egress "api.stripe.com"; }
}
```

### Hosted Checkout — the short path

```osy
using Osysharp.Payments.Stripe;

app.Secrets = [ new Secret("StripeApiKey"), new Secret("StripeWebhookSecret") ];

// ONE factory. It returns the client, key and all — legal, because nothing can read `ApiKey` back out of it: not a
// page, not a request, not a log line (the compiler tracks where the secret went).
StripeClient Stripe() { return new StripeClient { ApiKey = Secret.StripeApiKey }; }

string CheckoutUrl(Order order) {
  var session = Stripe().CreateCheckoutSession(new CheckoutSessionRequest {
    LineItems  = [new CheckoutLineItem { ProductName = $"Order {order.Reference}",
                                         UnitAmount = order.TotalMinor, Currency = "sek" }],
    SuccessUrl = $"{App.BaseUrl}/orders/{order.Reference}/{{CHECKOUT_SESSION_ID}}",
    CancelUrl  = $"{App.BaseUrl}/orders/{order.Reference}",
    ClientReferenceId = order.Reference,   // ← comes back on the session AND the webhook: which order was paid
  }, order.Reference);           // ← the idempotency key: a retry creates ONE session, not two
  return session.Url ?? "";
}
```

### The webhook — and this is the part that is not optional

```osy
[AllowAnonymous] void StripeHook(string body, string signature) {
  var e = new StripeWebhook { SigningSecret = Secret.StripeWebhookSecret }.Verify(body, signature);
  // A test event must not fulfil a live order — compare the event's mode with your KEY's, rather than dropping every
  // test event, or nothing in test mode can ever complete.
  if (e.Livemode != Secret.StripeApiKey.StartsWith("sk_live_")) { return; }
  if (e.Type == "checkout.session.completed" && e.Object?.PaymentStatus == "paid") {
    Fulfil(e.Object?.ClientReferenceId ?? "");
  }
}
```

The endpoint is public and unauthenticated — Stripe cannot log in — so **the signature is the authentication**.
Without `Verify`, that route reads "anyone on the internet may mark any order paid", and an unverified handler passes
every functional test you will write for it.

`Verify` throws `ValidationException` (answer 400) when the request is not provably Stripe's: a tampered body, the
wrong secret, a missing or malformed header, or a signature old enough to be a replay. It accepts a header carrying
several `v1=` signatures, which is what an endpoint receives while a signing secret is being rotated.

⚠ **Pass the RAW request body.** The signature covers the bytes Stripe sent; a body that has been parsed and
re-serialized has different whitespace and key order, and fails to verify for reasons that look nothing like the
cause.

## A card reader at a shop's counter

`StripeTerminal` takes a card on a Stripe Terminal smart reader (Stripe Reader S700, BBPOS WisePOS E), driven from the
server as Stripe describes (https://docs.stripe.com/terminal/payments/collect-card-payment?terminal-sdk-platform=server-driven):
a `card_present` payment intent handed to the reader (`process_payment_intent`), paid once the intent has succeeded for
this sale's reference and amount. It implements `ICounterPayment`; list it in the shop's `CounterPayments` beside the
`StripePayments` that refunds it. A reader takes its account's local currency (`Currencies`). Without hardware, register
a simulated reader (registration code `simulated-wpe`) with a test key and call `PresentTestCard()` where a customer would
tap (https://docs.stripe.com/terminal/references/testing).

```osy
new StripeTerminal { ApiKey = Secret.StripeApiKey, ReaderId = "tmr_…", Currencies = ["SEK"] }
```

## Three things that bite everyone

| | |
|---|---|
| **Amounts are integer minor units** | `2000` is 20.00, never `20`. The API takes `long` for exactly that reason — there is no rounding decision to get wrong because there is nothing to round. |
| **Stripe takes FORM bodies and answers JSON** | It will not accept JSON. A JSON request comes back as a complaint about a missing parameter, which reads like you forgot a field. `FormBody` does the encoding, brackets and all. |
| **A declined card is a 200** | Only transport and configuration faults throw. A refused card comes back successfully with `Status = "requires_payment_method"`. **Read `Status`** — a caller that only checks for an exception has written a flow that treats a decline as a sale. |

## What is in it

| | |
|---|---|
| `StripeClient` | payment intents (create · get · capture · cancel), hosted Checkout sessions (create · get), customers, refunds |
| `StripeWebhook` | HMAC-SHA256 signature verification with a replay window and rotation support |
| `FormBody` | Stripe's form encoding — nested brackets, indexed lists, sorted metadata |
| `StripeWire` | the response shapes, with every `id` mapped through `[ExternalName]` |

## Pin your API version once you are live

`ApiVersion` is empty by default, which means *your account's default*. Stripe moves that when you upgrade in the
dashboard, and it changes response shapes under a running app with no deploy on your side. This kit reads only
long-stable fields so it does not pin one on your behalf — but you should:

```osy
new StripeClient { ApiKey = Secret.StripeApiKey, ApiVersion = "2025-08-27.basil" }
```

## Source and tests

Everything is in `model/` — four files, no C# anywhere. `tests/` is the producer's own project: tests that
assert what went on the wire rather than that a call returned, including the whole webhook-forgery surface. It does
not ship, and `osy publish` runs it and refuses a version that fails.

