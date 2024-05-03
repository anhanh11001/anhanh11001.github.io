---
layout: post
title:  "Enhancing Network Requests by Chaining or Zipping with RxJava"
date:   2019-09-11 00:00:00 +0200
categories: blog
---

<center><img src="/assets/images/img_13.png"></center>

*The original blog post is posted in FOSSASIA blog site [here](https://blog.fossasia.org/enhancing-network-requests-by-chaining-or-zipping-with-rxjava/).*

In [Eventyay Attendee](https://github.com/fossasia/open-event-attendee-android), making HTTP requests to fetch data from the API is one of the most basic techniques used. RxJava comes in as a great method to help us making asynchronous requests and optimize the code a lot. This blog post will deliver some advanced RxJava used in Eventyay Attendee.

- Why using RxJava?
- Advanced RxJava Technique — Chaining network calls with RxJava
- Advanced RxJava Technique — Merging network calls with RxJava
- Conclusions
- Resources

## Why using RxJava?

There are many reasons why RxJava is a great API in Android Development. RxJava is an elegant solution to control data flow in programming, where developers can cache data, get data, update the UI after getting the data, handle asynchronous tasks. RxJava also works really well with MVVM architectural pattern.

## Chaining network calls with RxJava

Chaining RxJava is a technique using `flatMap()` operator of Rxjava. It will use the result from one network call in order to make the next network call.

In Eventyay Attendee, this technique is used when we want to update the user profile image. First, we need to upload the new profile image to the server in order to get the image URL, and then we use that URL to update the user profile

```
compositeDisposable += authService.uploadImage(UploadImage(encodedImage)).flatMap {
   authService.updateUser(user.copy(avatarUrl = it.url))
}.withDefaultSchedulers()
   .doOnSubscribe {
       mutableProgress.value = true
   }
   .doFinally {
       mutableProgress.value = false
   }
   .subscribe({
       mutableMessage.value = resource.getString(R.string.user_update_success_message)
       Timber.d("User updated")
   }) {
       mutableMessage.value = resource.getString(R.string.user_update_error_message)
       Timber.e(it, "Error updating user!")
   }
```

In conclusion, `chaining` RxJava helps to make HTTP requests more continuous and reduce unnecessary codes.


## Zipping network calls with RxJava

Zipping `RxJava` is a technique using `zip()` operator of `Rxjava`. It will wait for items from two or more Observables to arrive and then merge them together for emitting. This technique would be useful when two observables emit the same type of data.

In Eventyay Attendee, this technique is used when fetching similar events by merging events in the same location and merging events in the same event type.

```
var similarEventsFlowable = eventService.getEventsByLocationPaged(location, requestedPage, 3)
if (topicId != -1L) {
   similarEventsFlowable = similarEventsFlowable
       .zipWith(eventService.getSimilarEventsPaged(topicId, requestedPage, 3),
           BiFunction { firstList: List<Event>, secondList: List<Event> ->
               val similarList = mutableSetOf<Event>()
               similarList.addAll(firstList + secondList)
               similarList.toList()
           })
}
compositeDisposable += similarEventsFlowable
   .take(1)
   .withDefaultSchedulers()
   .subscribe({ response ->
       ...
   }, { error ->
       ...
   })
```

In conclusion, `zipping` RxJava helps running all the tasks in parallel and return all of the results in a single callback.

## Conclusion

Even though RxJava is pretty hard to understand and master, it is a really powerful tool in Android Development and MVVM models.


## Resources

- Eventyay Attendee Source Code: [#2010](https://github.com/fossasia/open-event-attendee-android/pull/2010), [#2117](https://github.com/fossasia/open-event-attendee-android/pull/2117)
- [RxJava Documentation](http://reactivex.io/documentation)