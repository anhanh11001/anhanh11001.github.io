---
layout: post
title:  "Implement PayPal SDK Android for payments in Eventyay Attendee"
date:   2019-09-12 00:00:00 +0200
categories: jekyll update
---

In [Eventyay Attendee](https://github.com/fossasia/open-event-attendee-android), one of our core features is paying for tickets for events. PayPal is one of the most famous payment gateways with millions of users around the world, and it is a part of our system. Let’s take a look at the implementation:

- Why using the PayPal SDK for Android?
- Implementing PayPal SDK for Android in Eventyay
- Results and conclusion
- Resources

## Why using the PayPal SDK for Android?

The first important note to be told is that Android SDK is already deprecated. PayPal is moving into a new system called Braintree, so it is my recommendation that developers should take a look at Braintree before implementing PayPal with Android SDK. We are still using Android SDK in our system as a lot of work about PayPal is already implemented in our server.

## Implementing PayPal Android SDK

**Step 1:** Set up the dependency in build.gradle

```
def PAYPAL_CLIENT_ID= System.getenv('PAYPAL_CLIENT_ID') ?: "YOUR_API_KEY"
//PayPal
implementation 'com.paypal.sdk:paypal-android-sdk:2.16.0'
```

**Step 2:** Start PayPal Payment

When users click on the PayPal button, pending order is made and the PayPal payment process should be started by the PayPal SDK. The following set up is needed:

```
private fun startPaypalPayment() {
    val paypalEnvironment = if (BuildConfig.DEBUG) PayPalConfiguration.ENVIRONMENT_SANDBOX
        else PayPalConfiguration.ENVIRONMENT_PRODUCTION
    val paypalConfig = PayPalConfiguration()
        .environment(paypalEnvironment)
        .clientId(BuildConfig.PAYPAL_CLIENT_ID)
    val paypalIntent = Intent(activity, PaymentActivity::class.java)
    paypalIntent.putExtra(PayPalService.EXTRA_PAYPAL_CONFIGURATION, paypalConfig)
    activity?.startService(paypalIntent)

    val paypalPayment = paypalThingsToBuy(PayPalPayment.PAYMENT_INTENT_SALE)
    val payeeEmail = attendeeViewModel.event.value?.paypalEmail ?: ""
    paypalPayment.payeeEmail(payeeEmail)
    addShippingAddress(paypalPayment)
    val intent = Intent(activity, PaymentActivity::class.java)
    intent.putExtra(PayPalService.EXTRA_PAYPAL_CONFIGURATION, paypalConfig)
    intent.putExtra(PaymentActivity.EXTRA_PAYMENT, paypalPayment)
    startActivityForResult(intent, PAYPAL_REQUEST_CODE)
}
```

In here, a PayPal payment object is created, and then the users will be redirected to a PayPal fragment screen for entering their account information and make a payment.

**Step 3:** Set up the PayPalPayment Object

```
private fun paypalThingsToBuy(paymentIntent: String): PayPalPayment =
    PayPalPayment(BigDecimal(safeArgs.amount.toString()),
        Currency.getInstance(safeArgs.currency).currencyCode,
        getString(R.string.tickets_for, attendeeViewModel.event.value?.name), paymentIntent)
```

**Step 4 (Additional):** Add billing/shipping information for payment

```
private fun addShippingAddress(paypalPayment: PayPalPayment) {
    if (rootView.billingEnabledCheckbox.isChecked) {
        val shippingAddress = ShippingAddress()
            .recipientName("${rootView.firstName.text} ${rootView.lastName.text}")
            .line1(rootView.billingAddress.text.toString())
            .city(rootView.billingCity.text.toString())
            .state(rootView.billingState.text.toString())
            .postalCode(rootView.billingPostalCode.text.toString())
            .countryCode(getCountryCodes(rootView.countryPicker.selectedItem.toString()))
        paypalPayment.providedShippingAddress(shippingAddress)
    }
}
```

**Step 5:** Handle the PayPal Payment result:

If the payment is successful, the client will receive a PaymentConfirm object, which contains information about the payment like paymentId, billing details,… This can be used to verify the payment in the server.

```
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    if (requestCode == PAYPAL_REQUEST_CODE && resultCode == Activity.RESULT_OK) {
        val paymentConfirm =
            data?.getParcelableExtra<PaymentConfirmation>(PaymentActivity.EXTRA_RESULT_CONFIRMATION)
        if (paymentConfirm != null) {
            val paymentId = paymentConfirm.proofOfPayment.paymentId
            attendeeViewModel.sendPaypalConfirm(paymentId)
        }
    }
}
```

**Step 6:** Verify the payment in the server

This can be done by sending the paymentId created after paying with PayPal. The server-side implementation below is in Python:

```
@staticmethod
def verify_payment(payment_id, order):
    """
    Verify Paypal payment one more time for paying with Paypal in mobile client
    """
    PayPalPaymentsManager.configure_paypal()
    try:
        payment_server = paypalrestsdk.Payment.find(payment_id)
        if payment_server.state != 'approved':
            return False, 'Payment has not been approved yet. Status is ' + payment_server.state + '.'

        # Get the most recent transaction
        transaction = payment_server.transactions[0]
        amount_server = transaction.amount.total
        currency_server = transaction.amount.currency
        sale_state = transaction.related_resources[0].sale.state

        if float(amount_server) != order.amount:
            return False, 'Payment amount does not match order'
        elif currency_server != order.event.payment_currency:
            return False, 'Payment currency does not match order'
        elif sale_state != 'completed':
            return False, 'Sale not completed'
        elif PayPalPaymentsManager.used_payment(payment_id, order):
            return False, 'Payment already been verified'
        else:
            return True, None
    except paypalrestsdk.ResourceNotFound:
        return False, 'Payment Not Found'

@staticmethod
def used_payment(payment_id, order):
    """
    Function to check for recycling of payment IDs
    """
    if Order.query.filter(Order.paypal_token == payment_id).first() is None:
        order.paypal_token = payment_id
        save_to_db(order)
        return False
    else:
        return True
```

## Results and Conclusions

<center><img src="/assets/images/img_14.gif"/></center>

PayPal is a great payment gateway that is used by millions of users around the world. It supports a lot of currency, has good security and has a lot of additional features. It is really easy to implement and test.

## Resources

- [Github Android SDK](https://github.com/paypal/PayPal-Android-SDK)

- [Open Event Attendee Android](https://github.com/fossasia/open-event-attendee-android/pull/2223)

- [Open Event Server](https://github.com/fossasia/open-event-server/commit/ead0ba06c771ec452f11d01d88d08042474be291)