---
layout: post
title:  "Implementing pagination with Retrofit in Eventyay Attendee"
date:   2019-09-08 00:00:00 +0200
categories: blog
---

<center><img src="/assets/images/img_9.png"></center>

*The original blog post is posted in FOSSASIA blog site [here](https://blog.fossasia.org/implementing-pagination-with-retrofit-in-eventyay-attendee/).*

**Pagination (Paging)** is a common and powerful technique in Android Development when making HTTP requests or fetching data from the database. [Eventyay Attendee](https://github.com/fossasia/open-event-attendee-android) has found many situations where data binding comes in as a great solution for our network calls with Retrofit. Let’s take a look at this technique.

- Problems without Pagination in Android Development
- Implementing Pagination with Kotlin with Retrofit
- Results and GIF
- Conclusions


## Problems without pagination in Android Development

Making HTTP requests to fetch data from the API is a basic work in any kind of application. With the mobile application, network data usage management is an important factor that affects the loading performance of the app. Without paging, all of the data are fetched even though most of them are not displayed on the screen. Pagination is a technique to load all the data in pages of limited items, which is much more efficient

## Implementing pagination in Android

**Step 1:** Set up dependency in `build.gradle`

```
// Paging
implementation "androidx.paging:paging-runtime:$paging_version"
implementation "androidx.paging:paging-rxjava2:$paging_version"
```

**Step 2:** Set up retrofit to fetch events from the API

```
@GET("events?include=event-sub-topic,event-topic,event-type")
fun searchEventsPaged(
   @Query("sort") sort: String,
   @Query("filter") eventName: String,
   @Query("page[number]") page: Int,
   @Query("page[size]") pageSize: Int = 5
): Single<List<Event>>
```

**Step 3:** Set up the DataSource

DataSource is a base class for loading data in the paging library from Android. In Eventyay, we use `PageKeyedDataSource`. It will fetch the data based on the number of pages and items per page with our default parameters. With `PageKeyedDataSource`, three main functions `loadInitial()`, `loadBefore()`, `loadAfter()` are used to to load each chunks of data.

```
class EventsDataSource(
   private val eventService: EventService,
   private val compositeDisposable: CompositeDisposable,
   private val query: String?,
   private val mutableProgress: MutableLiveData<Boolean>
) : PageKeyedDataSource<Int, Event>() {
   override fun loadInitial(
       params: LoadInitialParams<Int>,
       callback: LoadInitialCallback<Int, Event>
   ) {
       createObservable(1, 2, callback, null)
   }
   override fun loadAfter(params: LoadParams<Int>, callback: LoadCallback<Int, Event>) {
       val page = params.key
       createObservable(page, page + 1, null, callback)
   }
   override fun loadBefore(params: LoadParams<Int>, callback: LoadCallback<Int, Event>) {
       val page = params.key
       createObservable(page, page - 1, null, callback)
   }
   private fun createObservable(
       requestedPage: Int,
       adjacentPage: Int,
       initialCallback: LoadInitialCallback<Int, Event>?,
       callback: LoadCallback<Int, Event>?
   ) {
       compositeDisposable +=
           eventService.getEventsByLocationPaged(query, requestedPage)
               .withDefaultSchedulers()
               .subscribe({ response ->
                   if (response.isEmpty()) mutableProgress.value = false
                   initialCallback?.onResult(response, null, adjacentPage)
                   callback?.onResult(response, adjacentPage)
               }, { error ->
                   Timber.e(error, "Fail on fetching page of events")
               }
           )
   }
}
```

**Step 4:** Set up the Data Source Factory

`DataSourceFactory` is the class responsible for creating `DataSource` object so that we can create `PagedList` (A type of List used for paging) for events.

```
class EventsDataSourceFactory(
   private val compositeDisposable: CompositeDisposable,
   private val eventService: EventService,
   private val query: String?,
   private val mutableProgress: MutableLiveData<Boolean>
) : DataSource.Factory<Int, Event>() {
   override fun create(): DataSource<Int, Event> {
       return EventsDataSource(eventService, compositeDisposable, query, mutableProgress)
   }
}
```

**Step 5:** Adapt the current change to the ViewModel.

Previously, events fetched in `List<Event>` Object are now should be turned into `PagedList<Event>`.

```
sourceFactory = EventsDataSourceFactory(
   compositeDisposable,
   eventService,
   mutableSavedLocation.value,
   mutableProgress
)
val eventPagedList = RxPagedListBuilder(sourceFactory, config)
   .setFetchScheduler(Schedulers.io())
   .buildObservable()
   .cache()
compositeDisposable += eventPagedList
   .subscribeOn(Schedulers.io())
   .observeOn(AndroidSchedulers.mainThread())
   .distinctUntilChanged()
   .doOnSubscribe {
       mutableProgress.value = true
   }.subscribe({
       val currentPagedEvents = mutablePagedEvents.value
       if (currentPagedEvents == null) {
           mutablePagedEvents.value = it
       } else {
           currentPagedEvents.addAll(it)
           mutablePagedEvents.value = currentPagedEvents
       }
   }, {
       Timber.e(it, "Error fetching events")
       mutableMessage.value = resource.getString(R.string.error_fetching_events_message)
   })
```

**Step 6:** Turn ListAdapter into PagedListAdapter

`PageListAdapter` is basically the same ListAdapter to update the UI of the events item but specifically used for Pagination. In here, List objects can also be null.

```
class EventsListAdapter : PagedListAdapter<Event, EventViewHolder>(EventsDiffCallback()) {
   var onEventClick: EventClickListener? = null
   var onFavFabClick: FavoriteFabClickListener? = null
   var onHashtagClick: EventHashTagClickListener? = null
   override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): EventViewHolder {
       val binding = ItemCardEventsBinding.inflate(LayoutInflater.from(parent.context), parent, false)
       return EventViewHolder(binding)
   }
   override fun onBindViewHolder(holder: EventViewHolder, position: Int) {
       val event = getItem(position)
       if (event != null)
           holder.apply {
               bind(event, position)
               eventClickListener = onEventClick
               favFabClickListener = onFavFabClick
               hashTagClickListAdapter = onHashtagClick
           }
   }
   /**
    * The function to call when the adapter has to be cleared of items
    */
   fun clear() {
       this.submitList(null)
   }
```

## And here are the results

<center>
<img src="/assets/images/img_10.gif">
</center>

## Conclusion

Databinding is the way to go when working with a complex UI in Android Development. This helps reducing boilerplate code and to increase the readability of the code and the performance of the UI. One problem with data-binding is that sometimes, it is pretty hard to debug with unhelpful log messages. Hopefully, you can empower your UI in your project now with data-binding.

Pagination is the way to go for fetching items from the API and making infinite scrolling. This helps reduce network usage and improve the performance of Android applications. And that’s it. I hope you can make your application more powerful with pagination.


## Resources
- [Open Event Codebase](https://github.com/fossasia/open-event-attendee-android/pull/2012)
- [Documentation](https://developer.android.com/topic/libraries/architecture/paging/)
- [Google Codelab](https://codelabs.developers.google.com/codelabs/android-paging/#0)