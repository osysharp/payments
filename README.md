# Osysharp.Payments — one contract for taking money, whichever provider moves it

## Use case

A shop, a booking system or an invoice page has to take payments, and the provider will change: card through Stripe
today, Klarna next month, a bank transfer for wholesale customers, Swish for Swedish buyers. This package is the
contract between the code that sells and whatever moves the money. The app depends on it and on no provider in
particular; each provider is a class implementing `IPaymentProvider`, published as its own package or written in the
app. Adding a provider is one line in a list and never touches the checkout.

It also ships the one provider that needs no service: `ManualPayment` — a bank transfer, cash on collection, an
invoice paid later. The customer is told how to pay, and staff mark the payment received.

## Install and use

```osy
app MyShop {
  use Osysharp.Payments@0;
  use Osysharp.Payments.Stripe@0 { egress "api.stripe.com"; }   // a provider package, when you want one
  …
}
```

With `Osysharp.Shop`, the providers are the shop's list:

```osy
app.Shop = new ShopSetup {
  …
  Payments = [
    new StripePayments { ApiKey = Secret.StripeApiKey, WebhookSecret = Secret.StripeWebhookSecret },
    new ManualPayment  { Key = "banktransfer", Label = "Bank transfer",
                         Instructions = "Pay to account 1234-5678, marked with your order number." },
  ],
};
```

## The contract

An `IPaymentProvider` answers:

| member | what it does |
|---|---|
| `Key`, `Label`, `Description` | how it is stored and how the customer sees it |
| `IsReady()`, `CurrencyRefusal(currency, country, sellers)` | whether it can be offered now, and in this currency |
| `Start(PaymentRequest)` | where to send the customer, or instructions when there is nowhere |
| `Check(reference, amount, token)` | what the provider itself says about the payment — never the return address's word |
| `Capture`, `Refund`, `Cancel` | take authorised money, give it back, release it |
| `HandleEvent(body, headers)` | read and verify a webhook |
| `ChargeSaved(savedMethod, request)` | charge a way to pay kept earlier, for a renewal |

Amounts are in minor units (öre, cents) throughout.

Provider packages: `Osysharp.Payments.Stripe`, `Osysharp.Payments.Klarna`, `Osysharp.Payments.Swish`.

## Source and package

- Source: `model/Contract.osy` (the contract and its records) and `model/ManualPayment.osy`.
- Tests: `tests/`.
- Package: `use Osysharp.Payments@0;` — `osy lock` fetches the release archive from this repository's releases.

To write your own provider, implement `IPaymentProvider` in your app (or in a package of your own named
`<You>.Payments.<Provider>`) and add an instance to the list.
