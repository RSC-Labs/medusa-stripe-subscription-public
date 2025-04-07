# Medusa Stripe Subscription

## What is it?

Medusa Stripe Subscription is a payment provider, which allows you sell your products as subscription.

Interested? Go to: [What I need to do to have it?](#i-am-interested-what-i-need-to-do). It is a commercial software so it requires a licence to be used.

This repository has been created for instruction purposes and as a place for raising bugs and improvements.

## Found a bug or looking for an improvement?

Please raise an issue in Github issues.

## Installation

**Info**

This plugin is supported from `2.6.2` release of MedusaJS. If `2.6.2` is not yet available, you can use some of the snapshots, e.g. `2.6.2-snapshot-20250310153842`.

1. Install plugin by adding to your `package.json`:

```json
...
"@rsc-labs/medusa-stripe-subscription": "0.1.0" // or other available version
...
```
and execute install, e.g. `yarn install`.

2. Add plugin to your `medusa-config.js` with the licence key, which you received (**Note** - please notice that you need to add it to payment plugin):

```js
...
  plugins: [
    {
      resolve: "@rsc-labs/medusa-stripe-subscription",
      options: {
        apiKey: <stripe-api-key>,
        webhookSecret: <stripe-webhook-secret>
        licenceKey: <licence-key>
      },
    }
  ],
  modules: [
    {
      resolve: "@medusajs/medusa/payment",
      options: {
        providers: [
          {
            resolve: "@rsc-labs/medusa-stripe-subscription/providers/stripe-subscription",
            id: "stripe-subscription",
            options: {
              apiKey: <stripe-api-key>,
              webhookSecret: <stripe-webhook-secret>
              licenceKey: <licence-key>
            },
          }
        ]
      },
    },
...
```

All options are required. Stripe API Key and Stripe Webhook Secret you can get from your Stripe Dashboard.

### Database migration

Medusa Stripe Subscription introduces new models in database. To have it working, you need to firstly execute migrations:
```bash
npx medusa db:migrate
```

## Functionalities overview

Stripe Subscription provides following functionalities.

### Payment via Stripe hosted page

This plugin provides new payment method, which leds to Stripe hosted page. Your customer can fill credit card's details there and confirm payment.

### Cart to subscription

The whole cart is transformed into subscription. In other words - after using payment method provided by this plugin, the amount of subscription is equal to sum of items in your cart. Because of that, if you would like to provide subscription mechanism in your storefront, we recommend to limit items to 1 for the cart.

### Admin Subscriptions Dashboard

Plugin provides dedicated Admin Subscriptions Dashboard. It is visible as a new sidebar item on the left called `Subscriptions`.

### Handling subscription changes

Plugin implements many internal subscribers, which helps to sync information between your database and Stripe. It handles different statuses, failed payments and others.

### Sync accounts and products

Plugin also automatically creates customer in Stripe if your customer is logged-in in your storefront. Plugins also provides possibility to pass the product and variant via API to create the same product in Stripe.

### Managing subscription

Plugin lets you and your customer to manage the subscription. It includes cancelling subscription, resuming incomplete payment or changing payment method.

## Configuration and management

### Add payment provider.

After successful installation, you need to add `stripe-subscription` payment provider to the region, which wants support subscription.

### Adjust your checkout flow

Payment provider is added, but now you need to adjust your checkout flow in your storefront to properly handle Stripe Subscription.

Here are the important steps to know:

**1. Initiate payment**

Before making actual payment, you need to initiate it. The initialization is needed to create proper resources in Stripe. As an example from Next.JS starter, you can use `initiatePaymentSession` from `@lib/data/cart`. Here is the example of the call:
```tsx
await initiatePaymentSession(cart, 
  {
    provider_id: selectedPaymentMethod,
    data: {
      cartId: cart?.id,
      successUrl: `${getBaseURL()}/order/success`,
      cancelUrl: `${getBaseURL()}/order/cancel`,
      product: {
        product_id: "prod_01JN3SV0Z9G6AX9EGB1ZFKQSR1",
        variant_id: "variant_01JN3SV10X7Y95TRBXV690GMKE",
        name: "Monthly subscription"
      }
    }
  })
```

Additional information which is needed (besides usual parameters), is `data` parameter. The required parameters are:
- `cartId`
- `successUrl` - this is the URL where you would like to navigate user after successful payment from Stripe
- `cancelUrl` - this is the URL where you would like to navigate user after cancelled payment

The optional parameter is:
- `product` - here you can provide IDs from Medusa, which will be passed to Stripe to create a product. The name is your choice - you can provide title or subtitle of your product. The name will be reflected in Stripe, while `product_id` and `variant_id` will be saved as metadata.

This step you can call when your customer chooses payment method, before placing the order.

**2. Subscription order**

After initializing, you can place subscription order. This step shall be executed instead of regular order placing. The recommended approach is to check what payment method has been chosen, and if it is Stripe Subscription, then placing subscription order.
This step will leads your customer to Stripe hosted page. Here is the example of the call:
```tsx
export async function placeSubscriptionOrder() {
  const cartId = getCartId()
  if (!cartId) {
    throw new Error("No existing cart found when placing an order")
  }
  const result = await fetch(`http://localhost:9000/store/carts/${cartId}/complete-subscription`, {
    method: "POST",
    credentials: "include",
    headers: {
      "Content-Type": "application/json",
      ...getAuthHeaders(),
      "x-publishable-api-key": process.env.NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY || "temp"
    },
  })
  .then((res) => res.json())
  .then(( result ) => {
    return result;
  })

  return result;
}
```

This is actually calling `store/carts/${cartId}/complete-subscription` API which this plugin provides. In the result you are getting following object:
```ts
{
  subscription: <subscription-object>
  order: <order-object>
  stripeCheckoutSessionId: <checkout-session-id>
}
```

For your normal flow you need only the last one - `stripeCheckoutSessionId`. It is required to redirect user to Stripe hosted page. To do this, you need to use Stripe library (how to include Stripe in your Storefront you can find here: https://docs.medusajs.com/resources/storefront-development/checkout/payment/stripe).


Here is the example how you can use it:
```tsx
import { useElements, useStripe } from "@stripe/react-stripe-js"
// rest of your imports
...
// stripe initialization
const stripe = useStripe()
...
// actual placing the order and redirection
const result = await placeSubscriptionOrder();

await stripe?.redirectToCheckout({
  sessionId: result.stripeCheckoutSessionId as string,
})
```

Finally, your customer will be redirected to Stripe hosted page and, depends on the result of payment (success or cancel), it will be redirected to urls defined earlier - `successUrl` or `cancelUrl`.

### Manage subscription from Admin

Plugin provides new sidebar item called `Subscriptions`. After clicking it, you will see list of subscriptions. You can click every subscription to see its details.

Here are explanation of the fields:

- `Subscription Date` - this date is equal to the date when the subscription order is placed
- `Expiry Date` - when subscription starts, this date is empty. It is filled when the subscription is canceled or has been expired.
- `Status` - there are 5 statuses available:
  - `Active` - subscription is active. It means customer paid for the current period.
  - `Payment waiting` - subscription still waits to be paid. It happens when the first payment is not succesful and still waits for payment from the customer.
  - `Canceled` - subscription has been canceled by the customer. **Note** Such subscription can be resumed if customer wants to do this before end of the paid period.
  - `Expired` - subscription has been expired. It is non-recovable state. It happens when customer di
  - `Failed` - subscription failed means that customer did not paid for the last period. This state is recovable, for instance customer might want to update payment method or try again. However, this state can lead also to `Expired` status if subscription is permanently not paid.
  - `Force canceled` - this can happen when you, as admin, will cancel a subscription. It is very temporary status, until Stripe will delete subscription. After that, it will go to `Expired`.
- `Last order date` - this date says when the last order has been created. It changes based on the period.
- `Next order date` - this date says when the next order will be created. It changes based on the period.

#### Force cancel

As admin, you can force cancel subscription, even if it is in `Active` state. It might happen, when your customer broke some rules or wants to make some offline refund. When you click `Force cancel` you cannot undo this action - the subscription will finally be `Expired` and your customer needs to make next order manually.

### Manage subscription as customer

The plugin offers many APIs which can be used in your storefront to let customer manager their subscriptions.

#### Customer portal 

The easiest way is to create a button in customer's dashboard to proceed to Stripe hosted Customer Portal (https://docs.stripe.com/customer-management). The button shall execute the [API Customer Portal](./api/store-customers-me-subscriptions-portal.yaml). Your customer can then manage their subscriptions through Stripe. All changes will be reflected in your database and admin dashboard.

#### Cancel and resume subscription

Your customer can also cancel the subscription using the [API Customer Cancel](./api/store-customers-me-subscription-cancel.yaml). The subscription will change to state `canceled`, but subscription itself will be still active until next paid period. It means that your customer can resume subscription before end of the paid period using [API Customer Resume](./api/store-customers-me-subscription-resume.yaml). If it won't happen, then subsciption will go to state `expired`.

#### Continue checkout

If your customer is not able to successfully pay for the subscription after placing the order (first payment), she/he can try again in 24 hours. In such time window the checkout session is available, so it can be resumed. After 24 hours, the checkout session is ended and the subscription itself goes to `expiry` date and your customer needs to place another order. Use this [API Customer Checkout](./api/store-customers-me-subscription-checkout.yaml)

#### Update payment method

If recurring payment failed, then it might indicate that your customer needs change the payment method. The plugin exposes API for that, but you can also use Customer Portal.
Assuming that new payment method is correct one, two things might happen:
- you can wait until Stripe will try payment again
- your customer can call Retry Payment API which takes default payment method and tries to pay. **Note** Retry Payment API takes only default payment method, so the new payment method shall be set as default. If not, then payment might fail again.
Related APIs:

[API Customer Update Method](./api/store-customers-me-subscriptions-payment-method-update.yaml)

[API Customer Retry Payment](./api/store-customers-me-subscription-retry-payment.yaml)

### Architectural assumptions

#### Whole cart is subscription

At this moment, the whole cart is going to be subscription (in Stripe it is represent as one product). Because of that, we recommend to limit number of items in cart to 1.

#### Subscription is montly based

At this moment, the subscription is created as montly one. If you would like to have different time periods, please contact us.

### I am interested, what I need to do?

As mentioned, this plugin is behind the licence. We offer two types of licences:
- subscription - recommended for people who create the store for their own, so for very low monthly cost you can have Stripe subscription functionality.
- lifetime - recommended for people who create the store for the client (so cannot spend money on monthly basis). For one price, you can get this plugin forever.


We offer also 14-day free trial if you are not sure if you need this plugin - please reach us via labs@rsoftcon.com or go to https://www.medusa-plugins.com.