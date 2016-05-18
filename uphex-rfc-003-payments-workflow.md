# Overview

We want to capture the relevant information necessary to complete a payments workflow and complete a simple Stripe integration.

# Components

After a user signs up, there will now be a second step in the workflow: associating a `Subscription` with each user that we can use for recurring billing. The Subscription will contain only an external customer identifier that we can use to look things up in Stripe.

This relationship will be created by means of a credit card form that posts to Stripe. You arrive at the form immediately after a successful signup, rather than landing on the dashboard.

Here, we'll capture the necessary information and have Stripe validate everything via the drop-in checkout.js. If successful, we'll create the associated Stripe `Customer` and `Subscription`, and hold onto the subscription ID that was created.

It will still be legal to create an UpHex `User` (say, programmatically) without an associated Subscription -- such a user is still valid. However, that user won't be able to do anything interesting; they are redirected to the payment form each time if they land anywhere else.

# Names on users

We want to be able to address users by name in the future, and since they'll be adding their information to Stripe, we have an opportunity to fetch it here.

Names are a free-text field with no restrictions. Existing users will have a name equal to their email address.

When a user successfully adds a subscription, we also update their name to the value they used on the card.

# Coupon codes

We want to add the ability to pass coupon codes to Stripe so that it can calculate the appropriate amount to charge users. This calculation happens on Stripe's side and we're not responsible for anything other than transmitting the coupon code.

It doesn't look like Stripe's simple integration, [checkout.js](https://stripe.com/docs/checkout#integration-simple), supports that readily, so we'll probably have to use a custom integration to make this work.

# Grandfathered users

Users existing in UpHex prior to this will be placed on a grandfathered plan that will give them a Stripe subscription that lasts in perpetuity (or at least, some far-future date).

This will have a different plan ID than the stabndard plan, so we'll need to create the associated plan in Stripe to map the subscriptions to and migrate users accordingly.
