---
layout: post
title:  "Dependency Injection with Kotlin Koin in Eventyay Attendee"
date:   2019-09-04 00:00:00 +0200
categories: jekyll update
---

<center><img src="/assets/images/img_2.png"></center>


*The original blog post is posted in FOSSASIA blog site [here](https://blog.fossasia.org/dependency-injection-with-kotlin-koin-in-eventyay-attendee/).*

[Eventyay Attendee Android](https://github.com/fossasia/open-event-attendee-android) app contains a lot of shared components between classes that should be reused. Dependency Injection with Koin really comes in as a great problem solver.

**Dependency Injection** is a common design pattern used in various projects, especially with Android Development. In short, dependency injection helps to create/provide instances to the dependent class, and share it among other classes.

- Why using Koin?
- Process of setting up Koin in the application
- Results
- Conclusion
- Resources
Let’s get into the details

## Why using Koin?

Before Koin, dependency injection in Android Development was mainly used with other support libraries like Dagger or Guice. Koin is a lightweight alternative that was developed for Kotlin developers. Here are some of the major things that Koin can do for your project:

- Modularizing your project by declaring modules
- Injecting class instances into Android classes
- Injecting class instance by the constructor
- Supporting with Android Architecture Component and Kotlin
- Testing easily


## Setting up Koin in Android application

Adding the dependencies to build.gradle
```
// Koin
implementation "org.koin:koin-android:$koin_version"
implementation "org.koin:koin-androidx-scope:$koin_version"
implementation "org.koin:koin-androidx-viewmodel:$koin_version"
```
Create a folder to manage all the dependent classes.

<center><img src="/assets/images/img_1.png"></center>

Inside this Modules class, we define modules and create “dependency” class instances/singletons that can be reused or injected. For Eventyay Attendee, we define 5 modules: commonModule, apiModule, viewModelModule, networkModule, databaseModule. This saves a lot of time as we can make changes like adding/removing/editing the dependency in one place.

Let’s take a look at what is inside some of the modules:

***DatabaseModule***

```
val databaseModule = module {
   single {
       Room.databaseBuilder(androidApplication(),
           OpenEventDatabase::class.java, "open_event_database")
           .fallbackToDestructiveMigration()
           .build()
   }
   factory {
       val database: OpenEventDatabase = get()
       database.eventDao()
   }
   factory {
       val database: OpenEventDatabase = get()
       database.sessionDao()
   }
```

***CommonModule***

```
val commonModule = module {
   single { Preference() }
   single { Network() }
   single { Resource() }
   factory { MutableConnectionLiveData() }
   factory<LocationService> { LocationServiceImpl(androidContext()) }
}

```

***ApiModule***

```
val apiModule = module {
   single {
       val retrofit: Retrofit = get()
       retrofit.create(EventApi::class.java)
   }
   single {
       val retrofit: Retrofit = get()
       retrofit.create(AuthApi::class.java)
   }
```

***NetworkModule***

```
single {
   val connectTimeout = 15 // 15s
   val readTimeout = 15 // 15s
   val builder = OkHttpClient().newBuilder()
       .connectTimeout(connectTimeout.toLong(), TimeUnit.SECONDS)
       .readTimeout(readTimeout.toLong(), TimeUnit.SECONDS)
       .addInterceptor(HostSelectionInterceptor(get()))
       .addInterceptor(RequestAuthenticator(get()))
       .addNetworkInterceptor(StethoInterceptor())
   if (BuildConfig.DEBUG) {
       val httpLoggingInterceptor = HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BODY }
       builder.addInterceptor(httpLoggingInterceptor)
   }
   builder.build()
}
single {
   val baseUrl = BuildConfig.DEFAULT_BASE_URL
   val objectMapper: ObjectMapper = get()
   val onlineApiResourceConverter = ResourceConverter(
       objectMapper, Event::class.java, User::class.java,
       SignUp::class.java, Ticket::class.java, SocialLink::class.java, EventId::class.java,
       EventTopic::class.java, Attendee::class.java, TicketId::class.java, Order::class.java,
       AttendeeId::class.java, Charge::class.java, Paypal::class.java, ConfirmOrder::class.java,
       CustomForm::class.java, EventLocation::class.java, EventType::class.java,
       EventSubTopic::class.java, Feedback::class.java, Speaker::class.java, FavoriteEvent::class.java,
       Session::class.java, SessionType::class.java, MicroLocation::class.java, SpeakersCall::class.java,
       Sponsor::class.java, EventFAQ::class.java, Notification::class.java, Track::class.java,
       DiscountCode::class.java, Settings::class.java, Proposal::class.java)
   Retrofit.Builder()
       .client(get())
       .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
       .addConverterFactory(JSONAPIConverterFactory(onlineApiResourceConverter))
       .addConverterFactory(JacksonConverterFactory.create(objectMapper))
       .baseUrl(baseUrl)
       .build()
}
```

As described in the code, Koin support single for creating a singleton object, factory for creating a new instance every time an object is injected.

With all the modules created, it is really simple to get Koin running in the project with the function startKoin() and a few lines of code. We use it inside the application class:

```
startKoin {
   androidLogger()
   androidContext(this@OpenEventGeneral)
   modules(listOf(
       commonModule,
       apiModule,
       viewModelModule,
       networkModule,
       databaseModule
   ))
}
```

Injecting created instances defined in the modules can be used in two way, directly inside a constructor or injecting into Android classes.

Here is an example of dependency injection to the constructor that we used for a ViewModel class and injecting that ViewModel class into the Fragment:

```
class EventsViewModel(
   private val eventService: EventService,
   private val preference: Preference,
   private val resource: Resource,
   private val mutableConnectionLiveData: MutableConnectionLiveData,
   private val config: PagedList.Config,
   private val authHolder: AuthHolder
) : ViewModel() {
class EventsFragment : Fragment(), BottomIconDoubleClick {
   private val eventsViewModel by viewModel<EventsViewModel>()
   private val startupViewModel by viewModel<StartupViewModel>()
For testing, it is also really easy with support library from Koin.

@Test
fun testDependencies() {
   koinApplication {
       androidContext(mock(Application::class.java))
       modules(listOf(commonModule, apiModule, databaseModule, networkModule, viewModelModule))
   }.checkModules()
}

```

## Conclusion

Koin is really easy to use and integrate into Kotlin Android project. Apart from some of the basic functionalities mention above, Koin also supports other helpful features like Scoping or Logging with well-written documentation and examples. Even though it is only developed a short time ago, Koin has proved to be a great use in the Android community. So the more complicated your project is, the more likely it is that dependency injection with Koin will be a good idea.

## Resources

- [Documentation](https://insert-koin.io/)
- [Eventyay Attendee Android Codebase](https://github.com/fossasia/open-event-android)