---
layout: with-comments
title: "A Clever Approach to API Call Response Delivery in Android"
author: Marc Zych
author_url: https://github.com/marczych
summary: This is how we achieved reliable delivery of asynchronous API call
         results in Android while maintaining loose coupling of components
         using the Otto event bus.
---

iFixit's [Android app], like most mobile applications, is driven by content downloaded asynchronously from web APIs.
Since we currently have over 3 dozen API calls littered throughout the codebase, an easy-to-use and reliable method of performing API calls is crucial.

Over the past few years we have made many iterations of our API call interface; each one better than the last.
In chronological order: raw HTTP requests in Activities, abstracted HTTP requests with asynchronous callbacks, and finally a service performing requests and returning results through BroadcastReceivers.
Our current implementation replaces BroadcastReceivers with [Otto] because its interface is much cleaner.

Otto, and even BroadcastReceivers, mitigated the problem of leaking Activities and updating the UI after the Activity has been destroyed.
By registering for events in `onResume()` and unregistering in `onPause()`, the Activity only receives API call results if it is able to update the UI.
This gracefully handles orientation changes because the new Activity will receive results of API calls initiated by the previous Activity.
This system works remarkably well in terms of reliability and developer friendliness.

# The Problem

However, there are 2 major flaws present in this system which were made painfully obvious when developing [guide edit].
When editing a step in `StepEditActivity`, users can add images taken from their camera.
The user's image is uploaded in `StepEditActivity` which blocks step save until all pending image uploads are complete.
Since multiple images can be uploaded to a single step, the user commonly bounces between the camera and `StepEditActivity`.
Since Activities are unregistered from Otto in `onPause`, `StepEditActivity` doesn't receive image upload results when the user is taking a picture with the camera.
This is good because the app would crash if the event was received since it would try to update the UI.
This is bad because that was the one and only time to receive the event which was dropped on the floor because nobody was listening for it.
This caused step save to hang indefinitely because `StepEditActivity` didn't receive the image upload response and still thinks that there is a pending image upload.
Needless to say, this was unacceptable.

The other issue was fairly minor.
Many of our Otto `@Subscribe` methods have the same arguments and thus will receive the same events.
For example, `TeardownsActivity` and `FeaturedGuidesActivity` both listen for events of type `APIResult<Guide>` since they both display a list of guides, albeit from different sources.
This opens up the possibility of receiving results of API calls initiated by other Activities.

Both of these issues are concisely demonstrated in the following example:

1. Open `TeardownsActivity`.
1. `TeardownsActivity` initiates an API call to retrieve a list of teardown guides.
1. Navigate to `FeaturedGuidesActivity`.
1. The API call for teardown guides completes.
1. `FeaturedGuidesActivity` receives the list of teardowns, believing that it is a list of featured guides. The content is erroneously displayed to the user.
1. Navigate back to `TeardownsActivity` by pressing the back button.
1. `TeardownsActivity` patiently waits for its API call to finish, oblivious to the fact that it has already come and gone.

# The Solution

I realized 2 things when thinking about this problem:

1. API calls, and subsequently their results, need to be tied to the Activity that initiated it.
   This includes Activity instances created during orientation changes which the user considers to be the same screen.
   I'm going to call this an "activity session."
1. Unhandled API results should be saved and retried when its initiating Activity resumes.

Fortunately, implementing both of these is simple, straightforward, and doesn't involve modifying each and every API call site.

Defining an activity session is trivial because the platform practically does it for you.
The technique is to create a unique id for a new Activity session and use saved instance state to pass it to new Activities in the same session.
The code can be put in the base Activity class like so:

{% highlight java %}
public abstract class BaseActivity extends
 SherlockFragmentActivity {
   private int mActivityid;

   @Override
   public void onCreate(Bundle savedState) {
      super.onCreate(savedState);

      if (savedState != null) {
         mActivityid = savedState.getInt(ACTIVITY_ID);
      } else {
         mActivityid = generateActivityid();
      }
   }

   @Override
   public void onSaveInstaneState(Bundle outState) {
      super.onSaveInstanceState(outState);

      outState.putInt(ACTIVITY_ID, mActivityid);
   }

   private static int sActivityidCounter = 0;
   private int generateActivityid() {
      return sActivityidCounter++;
   }
}
{% endhighlight %}

Now we can use the `activityid` to tie API results to an activity session.
In addition to the endpoint, URL, auth token, etc., the `activityid` that initiated the request is stored in the `APICall` which is accessible to API result receivers.
Requiring each `@Subscribe` API result method to check the `activityid` before proceeding is too cumbersome with as many API calls that we have.
`BaseActivity` is the best place to perform such validation but receiving the event is tricky.
The usual `APIResult<?>` object can't be posted because the derived Activity will receive it as well.
We can instead wrap the `APIResult<?>` in a proxy class that only `BaseActivity` will listen for so it can check the `activityid`:

{% highlight java %}
@Subscribe
public void onApiCall(APIEvent.ActivityProxy activityProxy) {
   if (activityProxy.getActivityid() == mActivityid) {
      // Send the actual result off to the real handler.
      MainApplication.getBus().post(activityProxy.getApiEvent());
   } else {
      // Send the event back to APIService so it can retry it for
      // the intended Activity.
      MainApplication.getBus().post(new DeadEvent(
       MainApplication.getBus(), activityProxy.getApiEvent()));
   }
}
{% endhighlight %}

If the `activityid` matches, the actual `APIResult<?>` is posted to the bus so the Activity can receive it like normal.
If it doesn't match, then it is as if the event wasn't handled at all.
For Otto, this results in a `DeadEvent` that wraps the object that wasn't handled.
`APIService` listens for `DeadEvent`s and hangs on to ones containing an `APIResult<?>` or `APIResult.ActivityProxy`.
When an Activity registers to Otto, dead events with the same `activityid` are posted to the bus so the Activity can receive the results.

This approach gives us reliable delivery of API results while maintaining loose coupling of Activities by using Otto.
In particular, when resumed, Activities receive API call results that completed when the Activity was paused.
Additionally, Activities only receive results for API calls that they initiated.

The [full changeset] weighs in at 180 additions and 84 deletions; a decent portion of which was genericizing a proof of concept implementation that was tailored to uploading images on step edit.
Our app is, of course, [open source] so feel free to fork it and hack away!

[Android app]: https://play.google.com/store/apps/details?id=com.dozuki.ifixit
[guide edit]: http://ifixit.org/5280/whole-new-android-experience-on-ifixit/
[full changeset]: https://github.com/iFixit/iFixitAndroid/compare/2219cb8...983be9b
[open source]: https://github.com/iFixit/iFixitAndroid
[Otto]: http://square.github.io/otto/
