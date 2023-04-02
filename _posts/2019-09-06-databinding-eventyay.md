---
layout: post
title:  "Data Binding with Kotlin in Eventyay Attendee"
date:   2019-09-06 00:00:00 +0200
categories: jekyll update
---

<center><img src="./images/img_4.png"></center>

***The original blog post is posted in FOSSASIA blog site [here](https://blog.fossasia.org/data-binding-with-kotlin-in-eventyay-attendee/).***

Databinding is a common and powerful technique in Android Development. [Eventyay Attendee](https://github.com/fossasia/open-event-attendee-android) has found many situations where data binding comes in as a great solution for our complex UI. Let’s take a look at this technique.

- Problems without data binding in Android Development
- Implementing Databinding with Kotlin inside Fragment
- Implementing Databinding with Kotlin inside RecyclerView/Adapter
- Results and GIF
- Conclusions

## Problems without databinding in Android Development

Getting the data and fetching it to the UI is a basic work in any kind of application. With Android Development, the most common way to do is it to call function like `.setText()`, `isVisible = true/false`,.. in your fragment. This can create many long boilerplate codes inside Android classes. Databinding removes them and moves to the UI classes (XML).

## Implementing databinding in Fragment view

**Step 1:** Enabling data binding in the project build.gradle

```
android {
   dataBinding {
       enabled = true
   }
```

Step 2: Wrap the current layout with `<layout></layout> `tag. Inside that, put `<data></data>` to indicate any variables needed for data binding. For example, this code here display an event variable for our fragment about event details:

```
<layout xmlns:android="http://schemas.android.com/apk/res/android"
   xmlns:app="http://schemas.android.com/apk/res-auto"
   xmlns:bind="http://schemas.android.com/tools">
   <data>
       <variable
           name="event"
           type="org.fossasia.openevent.general.event.Event" />
   </data>
   <androidx.coordinatorlayout.widget.CoordinatorLayout
       android:id="@+id/eventCoordinatorLayout"
       android:layout_width="match_parent"
       android:layout_height="match_parent"
       android:background="@android:color/white">
```

**Step 3:** Bind your data in the XML file and create a Binding Adapter class for better usage

With the setup above, you can start binding your data with `“@{<data code here>}”`

```
<TextView
   android:id="@+id/eventName"
   android:layout_width="0dp"
   android:layout_height="wrap_content"
   android:layout_marginLeft="@dimen/layout_margin_large"
   android:layout_marginTop="@dimen/layout_margin_large"
   android:layout_marginRight="@dimen/layout_margin_large"
   android:text="@{event.name}"
   android:fontFamily="sans-serif-light"
   android:textColor="@color/dark_grey"
   android:textSize="@dimen/text_size_extra_large"
   app:layout_constraintEnd_toEndOf="@+id/eventImage"
   app:layout_constraintStart_toStartOf="@+id/eventImage"
   app:layout_constraintTop_toBottomOf="@+id/eventImage"
   tools:text="Open Source Meetup" />
```

Sometimes, to bind our data normally we need to use a complex function, then creating Binding Adapter class really helps. For example, Eventyay Attendee heavily uses Picasso function to fetch image to `ImageView`:

```
@BindingAdapter("eventImage")
fun setEventImage(imageView: ImageView, url: String?) {
   Picasso.get()
       .load(url)
       .placeholder(R.drawable.header)
       .into(imageView)
}
<ImageView
   android:id="@+id/eventImage"
   android:layout_width="@dimen/layout_margin_none"
   android:layout_height="@dimen/layout_margin_none"
   android:scaleType="centerCrop"
   android:transitionName="eventDetailImage"
   app:layout_constraintDimensionRatio="2"
   app:layout_constraintEnd_toEndOf="parent"
   app:layout_constraintHorizontal_bias="0.5"
   app:layout_constraintStart_toStartOf="parent"
   app:eventImage="@{event.originalImageUrl}"
   app:layout_constraintTop_toBottomOf="@id/alreadyRegisteredLayout" />
```

**Step 4:** Finalize data binding setup in Android classes. We can create a binding variable. The binding root will serve as the root node of the layout. Whenever data is needed to be bind, set the data variable stated to that binding variable and call function `executePendingBingdings()`

```
private lateinit var rootView: View
private lateinit var binding: FragmentEventBinding
binding = DataBindingUtil.inflate(inflater, R.layout.fragment_event, container, false)
rootView = binding.root
binding.event = event
binding.executePendingBindings()
```

## Some notes

- In the example mentioned above, the name of the binding variable class is auto-generated based on the name of XML file + “Binding”. For example, the XML name was `fragment_event` so the DataBinding classes generated name is `FragmentEventBinding`.
- The data binding class is only generated only after compiling the project.
- Sometimes, compiling the project fails because of some problems due to data binding without any clear log messages, then that’s probably because of error when binding your data in XML class. For example, we encounter a problem when changing the value in **Attendee** data class from `firstname` to `firstName` but XML doesn’t follow the update. So make sure you bind your data correctly

```
<TextView
   android:id="@+id/name"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"
   android:layout_marginBottom="@dimen/layout_margin_large"
   android:textColor="@color/black"
   android:textSize="@dimen/text_size_expanded_title_large"
   android:text="@{attendee.firstname + ` ` + attendee.lastname}"
   tools:text="@string/name_preview" />
```

## Conclusion

Databinding is the way to go when working with a complex UI in Android Development. This helps reducing boilerplate code and to increase the readability of the code and the performance of the UI. One problem with data binding is that sometimes, it is pretty hard to debug with an unhelpful log message. Hopefully, you can empower your UI in your project now with data binding.

## Resources

- Eventyay Attendee Android Codebase: https://github.com/fossasia/open-event-android
- Eventyay Attendee Android [PR: #1961 — feat: Set up data binding for Recycler/Adapter](https://github.com/fossasia/open-event-attendee-android/pull/1961)
- Documentation: https://developer.android.com/topic/libraries/data-binding
- Google Codelab: https://codelabs.developers.google.com/codelabs/android-databinding/#0