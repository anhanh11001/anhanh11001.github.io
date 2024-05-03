---
layout: post
title:  "Implementing places autosuggestion with Mapbox for searching events in Eventyay Attendee"
date:   2019-09-05 00:00:00 +0200
categories: blog
---

<center><img src="/assets/images/img_3.png"></center>

*The original blog post is posted in FOSSASIA blog site [here](https://blog.fossasia.org/implementing-places-autosuggestion-with-mapbox-for-searching-events-in-eventyay-attendee/).*

In [Eventyay Attendee](https://github.com/fossasia/open-event-attendee-android), searching for events has always been a core function that we focus on. When searching for events based on location, autosuggestion based on user input really comes out as a great feature to increase the user experience. Let’s take a look at the implementation

- Why using Mapbox?
- Integrating places autosuggestion for searching
- Conclusion
- Resources


## Why using Mapbox?

There are many Map APIs to be taken into consideration but we choose Mapbox as it is really to set up and use, good documentation and reasonable pricing for an open-source project compared to other Map API.

## Integrating places autosuggestion for searching

**Step 1:** Setup dependency in the build.gradle + the MAPBOX key

```
//Mapbox java sdk
implementation 'com.mapbox.mapboxsdk:mapbox-sdk-services:4.8.0'
```

**Step 2:** Set up functions inside ViewModel to handle autosuggestion based on user input:

```
private fun loadPlaceSuggestions(query: String) {
   // Cancel Previous Call
   geoCodingRequest?.cancelCall()
   doAsync {
       geoCodingRequest = makeGeocodingRequest(query)
       val list = geoCodingRequest?.executeCall()?.body()?.features()
       uiThread { placeSuggestions.value = list }
   }
}
private fun makeGeocodingRequest(query: String) = MapboxGeocoding.builder()
   .accessToken(BuildConfig.MAPBOX_KEY)
   .query(query)
   .languages("en")
   .build()
```

Based on the input, the functions will update the UI with new inputs of auto-suggested location texts. The `MAPBOX_KEY` can be given from the Mapbox API.

**Step 3:** Create an XML file to display autosuggestion strings item and set up RecyclerView in the main UI fragment


**Step 4:** Set up `ListAdapter` and `ViewHolder` to bind the list of auto-suggested location strings. Here, we use CamenFeature to set up with `ListAdapter` as the main object. With the function `.placeName()`, information about the location will be given so that ViewHolder can bind the data

```
class PlaceSuggestionsAdapter :
   ListAdapter<CarmenFeature,
       PlaceSuggestionViewHolder>(PlaceDiffCallback()) {
   var onSuggestionClick: ((String) -> Unit)? = null
   override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PlaceSuggestionViewHolder {
       val itemView = LayoutInflater.from(parent.context)
           .inflate(R.layout.item_place_suggestion, parent, false)
       return PlaceSuggestionViewHolder(itemView)
   }
   override fun onBindViewHolder(holder: PlaceSuggestionViewHolder, position: Int) {
       holder.apply {
           bind(getItem(position))
           onSuggestionClick = this@PlaceSuggestionsAdapter.onSuggestionClick
       }
   }
   class PlaceDiffCallback : DiffUtil.ItemCallback<CarmenFeature>() {
       override fun areItemsTheSame(oldItem: CarmenFeature, newItem: CarmenFeature): Boolean {
           return oldItem.placeName() == newItem.placeName()
       }
       override fun areContentsTheSame(oldItem: CarmenFeature, newItem: CarmenFeature): Boolean {
           return oldItem.equals(newItem)
       }
   }
}
fun bind(carmenFeature: CarmenFeature) {
   carmenFeature.placeName()?.let {
       val placeDetails = extractPlaceDetails(it)
       itemView.placeName.text = placeDetails.first
       itemView.subPlaceName.text = placeDetails.second
       itemView.subPlaceName.isVisible = placeDetails.second.isNotEmpty()
       itemView.setOnClickListener {
           onSuggestionClick?.invoke(placeDetails.first)
       }
   }
}
```

**Step 5:** Set up RecyclerView with Adapter created above:

```
private fun setupRecyclerPlaceSuggestions() {
   rootView.rvAutoPlaces.layoutManager = LinearLayoutManager(context)
   rootView.rvAutoPlaces.adapter = placeSuggestionsAdapter
   placeSuggestionsAdapter.onSuggestionClick = {
       savePlaceAndRedirectToMain(it)
   }
}
```

## Results

<center><img src="/assets/images/autolocation.gif"></center>

## Conclusions

**Place Autocorrection** is a really helpful and interesting feature to include in your next project. With the help of Mapbox SDK, it is really easy to implement to enhance your user experience in your application.

## Resources

- [Eventyay Attendee Android Codebase](https://github.com/fossasia/open-event-android)
- Eventyay Attendee [PR: #1594 — feat: Mapbox Autosuggest](https://github.com/fossasia/open-event-android/pull/1594)
- [Documentation](https://docs.mapbox.com/android/plugins/overview/places/)