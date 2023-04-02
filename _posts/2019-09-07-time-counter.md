---
layout: post
title:  "Adding time counter on ordering tickets in Eventyay Attendee"
date:   2019-09-07 00:00:00 +0200
categories: jekyll update
---

<center><img src="/assets/images/img_5.gif"/></center>

*The original blog post is posted in FOSSASIA blog site [here](https://blog.fossasia.org/adding-time-counter-on-ordering-tickets-in-eventyay-attendee/).*

In [Eventyay Attendee](https://github.com/fossasia/open-event-attendee-android), ordering tickets for events has always been a core functionality that we focus on. When ordering tickets, adding a time counter to make a reservation and release tickets after timeout is a common way to help organizers control their tickets’ distribution and help users save up their tickets. Let’s take a look at how to implement this feature

- Implementing the time counter
- Some notes on implementing time counter
- Conclusion
- Resources

## Integrating time counter to your system
INTEGRATING TIME COUNTER TO YOUR SYSTEM

**Step 1:** Create the UI for your time counter. In here, we made a simple View container with TextView inside to update the time.

<center><img src="/assets/images/img_8.png"></center><br>

**Step 2:** Set up the time counter with Android CountdownTimer with the total countdown time and the ticking time. In Eventyay, the default countdown time is `10 minutes (600,000 ms)` with the ticking time is `(1,000 ms)`, which means the UI is updated every one second.

```
private fun setupCountDownTimer(orderExpiryTime: Int) {
   rootView.timeoutCounterLayout.isVisible = true
   rootView.timeoutInfoTextView.text =
       getString(R.string.ticket_timeout_info_message, orderExpiryTime.toString())
   val timeLeft: Long = if (attendeeViewModel.timeout == -1L) orderExpiryTime * 60 * 1000L
                           else attendeeViewModel.timeout
   timer = object : CountDownTimer(timeLeft, 1000) {
       override fun onFinish() {
           findNavController(rootView).navigate(AttendeeFragmentDirections
               .actionAttendeeToTicketPop(safeArgs.eventId, safeArgs.currency, true))
       }
       override fun onTick(millisUntilFinished: Long) {
           attendeeViewModel.timeout = millisUntilFinished
           val minutes = millisUntilFinished / 1000 / 60
           val seconds = millisUntilFinished / 1000 % 60
           rootView.timeoutTextView.text = "$minutes:$seconds"
       }
   }
   timer.start()
}
```

**Step 3:** Set up creating a pending order when the timer starts counting so that users can hold a reservation for their tickets. A simple POST request about empty order to the API is made

```
fun initializeOrder(eventId: Long) {
   val emptyOrder = Order(id = getId(), status = ORDER_STATUS_INITIALIZING, event = EventId(eventId))
   compositeDisposable += orderService.placeOrder(emptyOrder)
       .withDefaultSchedulers()
       .subscribe({
           mutablePendingOrder.value = it
           orderIdentifier = it.identifier.toString()
       }, {
           Timber.e(it, "Fail on creating pending order")
       })
}
```

**Step 4:** Set up canceling order when the time counter finishes. As time goes down, the user should be redirected to the previous fragment and a pop-up dialog should show with a message about reservation time has finished. There is no need to send an HTTP request to cancel the pending order as it is automatically handled by the server.

**Step 5:** Cancel the time counter in case the user leaves the app unexpectedly or move to another fragment. If this step is not made, the CountdownTimer still keeps counting in the background and possibly call `onFinished()` at some point that could evoke functions and crash the app

```
override fun onDestroy() {
   super.onDestroy()
   if (this::timer.isInitialized)
       timer.cancel()
}
```


## Results

<center>
<img src="/assets/images/img_6.jpeg"> 
<img src="/assets/images/img_7.gif">
</center>

## Conclusion

For a project with a ticketing system, adding a time counter for ordering is a really helpful feature to have. With the help of Android CountdownTimer, it is really to implement this function to enhance your user experience.

## Resources
- [Eventyay Attendee Android Codebase](https://github.com/fossasia/open-event-android)
- Eventyay Attendee Android [PR: #1843 — Add time counter on ordering ticket](https://github.com/fossasia/open-event-android/pull/1843)
- [Documentation](https://developer.android.com/reference/android/os/CountDownTimer)
