# Osysharp.Payments.Klarna — pay later, pay now and instalments, in about 200 lines of Osy#

Take payment through [Klarna](https://klarna.com) — a session, an order, and the capture and refund half that comes
after — from classes written entirely in Osy# over the public `Http.*` surface.

## Use case

Any Nordic or European shop where "pay in 30 days" or "pay in three" is the difference between a sale and an
abandoned basket. Klarna is the default expectation in Sweden and close to it across the Nordics and Germany.

Pairs with `Osysharp.Payments.Stripe` rather than competing: cards through one, Klarna through the other, and an app can offer
both.

## No certificate, and no BankID

Worth saying beside `Osysharp.BankId`: Klarna authenticates a **merchant** with ordinary HTTP Basic over TLS. This kit
needs no `clientCert` and no `serverCa`, so nothing about it waits on a bank's private authority.

Klarna may put BankID in front of the **shopper** inside its own flow — that happens on Klarna's side of the redirect
and is not your integration.

```osy
// app.osy
app Shop {
  model "**/*.osy";
  use Osysharp.Payments.Klarna@0 {
    egress "api.klarna.com";                 // the kit's declared default
    egress "api.playground.klarna.com";      // …and the one you choose while building
  }
}

app.Secrets = [ new Secret("KlarnaUsername"), new Secret("KlarnaPassword") ];
```

Credentials come from Klarna's merchant portal — **Settings → API credentials**, self-service, no bank agreement to
start. Playground and live are separate credentials.

## The flow, and the step that is not yours

```osy
using Osysharp.Payments.Klarna;

// ONE factory, credentials and all. It compiles because nothing can read `Username`/`Password` back out of what it
// returns — the compiler tracks where the secrets went — and a page or a request cannot receive the client at all.
KlarnaClient Klarna(Order order) {
  return new KlarnaClient {
    Username = Secret.KlarnaUsername, Password = Secret.KlarnaPassword,
    BaseUrl = "https://api.playground.klarna.com",
    PurchaseCountry = "SE", PurchaseCurrency = "SEK", Locale = "sv-SE",
  };
}

// 1. On the server: open a session. Hand `ClientToken` to the page.
string StartCheckout(Order order) {
  var session = Klarna(order).CreateSession(Lines(order));
  return session.ClientToken ?? "";
}

// 2. In the BROWSER: Klarna's widget takes over and hands your page an authorization token.

// 3. On the server: turn that token into an order.
string Place(Order order, string authorizationToken) {
  var placed = Klarna(order).CreateOrder(authorizationToken, Lines(order));
  return placed.OrderId ?? "";
}

// …and later, when you SHIP:
void Ship(string klarnaOrderId) {
  var klarna = new KlarnaClient {
    Username = Secret.KlarnaUsername, Password = Secret.KlarnaPassword,
    BaseUrl = "https://api.playground.klarna.com",
    PurchaseCountry = "SE", PurchaseCurrency = "SEK", Locale = "sv-SE",
  };
  klarna.Capture(klarnaOrderId);
}
```

⚑ **One country, one currency.** Klarna takes each country's own currency and no other — `purchase_country` and
`purchase_currency` must pair: SE with SEK, DE/FI/NL/AT… with EUR, NO with NOK, DK with DKK, GB with GBP, US with USD
(Klarna, *Purchase countries, currencies and locales*:
https://docs.klarna.com/klarna-payments/in-depth-knowledge/puchase-countries-currencies-locales/ — "We only support the
local currency of each country"). As a way to pay under the payments contract, `KlarnaPayments.CurrencyRefusal` says so
before Klarna is asked: a shop selling in euros offers Klarna to a buyer in a euro country, and a Swedish market
(`Country = "SE"`) is offered only for kronor. `KlarnaMarkets()` is the table.

⛔ **The order is not the money.** Creating an order reserves it; `Capture` charges the shopper and is a separate call
you make on dispatch. Capturing at checkout is legal but it is not what Klarna's model assumes, and an uncaptured
order eventually expires.

⛔ **A refund is against a capture, not the order's face value.** An order never captured is **cancelled**, not
refunded — there is nothing to give back.

## What the kit exists to get right

Klarna validates the arithmetic of a basket and refuses a mismatch with a message naming a **field** rather than the
sum. It is the single most common way a Klarna integration fails, so the kit computes all of it:

| | |
|---|---|
| **`BasketLine` has no `TotalAmount`** | Klarna requires `unit_price × quantity`. A field a caller can get wrong is a field the kit should not offer. |
| **`order_amount` is the sum of the lines** | Computed, never accepted from the caller. |
| **Tax is the PORTION of an inclusive price** | At 25%, the tax inside 25000 is **5000** — `total × rate ÷ (10000 + rate)` — not 6250. Getting it backwards inflates every order by the tax. |
| **`tax_rate` is hundredths of a percent** | 2500 is 25%. Not a fraction, not a percentage. |
| **Amounts are minor units** | 2000 is 20.00, as `long`. There is no rounding decision to get wrong because there is nothing to round. |

Four things are refused before the call, each naming the fix: a basket with no lines, a line with no name (the shopper
reads it), a quantity below one, and a negative unit price — which is the natural thing to reach for and wrong in
Klarna's model, where **a discount is its own line with a negative price**.

## Two failures the kit explains and Klarna does not

**A 401 is usually the environment, not the password.** Playground credentials work only against
`api.playground.klarna.com` and live ones only against `api.klarna.com`. The message says so and names the host the
client is pointed at.

**Every error message, not the first.** Klarna returns `error_messages` as a *list* — a basket with three bad lines is
refused once naming all three — plus a `correlation_id`, which is what Klarna's support asks for before anything else.

## The Hosted Payment Page — Klarna's checkout, for a shop without its own

```osy
var session = Klarna().CreateSession(Lines(order), order.Reference);        // your reference rides on the order
var page = Klarna().CreateHostedPage(session.SessionId ?? "",
    $"{App.BaseUrl}/paid/{order.Reference}/klarna-{{{{order_id}}}}",          // Klarna fills {{order_id}}
    $"{App.BaseUrl}/paid/{order.Reference}/cancelled",
    $"{App.BaseUrl}/paid/{order.Reference}/failed");
// send the shopper to page.RedirectUrl
```

A sixth argument, `statusUrl`, is where Klarna POSTs a `KlarnaHostedPageEvent` when the session completes. Give one:
a shopper who pays and then closes the tab never comes back to your success address, and without the callback you
never hear about that order. The callback is unsigned, so treat it exactly like the success address.

With `placeOrder` (the default) Klarna creates the order when the shopper finishes and puts its id in the success
address. ⛔ **That id is a claim, not a payment** — anybody can type one. Read it back with `GetOrder(id)` and check
`MerchantReference` is your order and `OrderAmount` what you expect before you mark anything paid.

## Keeping the way to pay — a customer token, for renewals

As an `Osysharp.Payments` provider, `KlarnaPayments` answers `SavesMethods = true`. A request with `SaveMethod` opens a
`buy_and_tokenize` session, and the Hosted Payment Page only AUTHORIZES (`placeOrder: false`). On the return, `Check`
reads the page and the session back from Klarna, makes the customer token from the authorization
(`CreateCustomerToken`) and only THEN places the order with the session's own basket (`PlaceSessionOrder`) — the order
closes the session, so the token has to come first. The token is `PaymentCheck.SavedMethod`, named from Klarna's own
record of it ("Klarna · Visa •••• 1234").

`ChargeSaved` places an order on the token (`ChargeToken`, keyed with `Klarna-Idempotency-Key`, captured at once since a
renewal ships nothing) and answers Paid, Pending (Klarna still checking) or Failed with a sentence — a 4xx or a rejected
order is an answer, never an exception. `ForgetMethod` cancels the token.

⚠ A kept payment completes on the RETURN, not on the status callback: the callback names the hosted page, not the
payment session the order is placed from. The customer-token calls follow Klarna's published OpenAPI descriptions; run a
kept payment through Klarna's playground with your own credentials before you take one live.

## Checked against Klarna's playground

A session opened against Klarna's playground answers

```
session_id=cfb41881-…  client_token=present(1868 chars)  methods=[pay_later, pay_now]
```

— every declared field populated, which is the one thing a stubbed test structurally cannot prove. And because
Klarna validates basket arithmetic server-side and answers 400 on a mismatch, a 200 also means the computed
`order_amount` (29900), the per-line totals and the inclusive-tax figures (5000 and 980) matched what Klarna worked
out independently — so the tax-inclusive formula is confirmed by Klarna itself.

## Source and tests

Three files in `model/`, with no C# anywhere. `tests/` is the producer's own project, most of it about the arithmetic
above. It does not ship, and
`osy publish` runs it and refuses a version that fails.

