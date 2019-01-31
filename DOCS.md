![TrueX logo](media/truex.png)


_TruexAdRenderer tvOS Documentation_

_Version 3.5.0_

_Updated Dec 18, 2018_


# **<span style="text-decoration:underline;">1. Overview</span>**

In order to support interactive ads on tvOS, TrueX has created a renderer library that can renderer TrueX ads natively, while still leveraging the same VPAID architecture available in Uplynk and FreeWheel.

With this library, the host player app can defer to the TruexAdRenderer when it is required to display a TrueX ad.

For simplicity, publisher implemented code will be referred to as "app code" while TrueX implemented code will be referred to as "renderer code.

The approach here is nearly identical as TruexAdRenderer-iOS ([documentation]( https://docs.google.com/document/d/1GwK9Yrd1OQ24imPpcktb9WX1UBafxZKXLtq1klw9N1M/edit?usp=sharing)).

TrueX will provide an Objective-C **<code>TruexAdRenderer</code></strong> library that can be loaded into the app. This library will offer a class, <strong><code>TruexAdRenderer</code></strong>, that will need to be instantiated, initialized and given certain commands (described below in Section 4) by the app code. It will also contain a singleton of shared constants, <strong><code>TruexConstants</code></strong>.

At this point, the renderer code will take on the responsibility of requesting ads from TrueX server, creating the native UI for the TrueX choice card and interactive ad unit, as well as communicating events to the app code when action is required.

The app code will still need to parse out the Uplynk ad response, detect when a TrueX ad is supposed to display, pause the stream, instantiate **<code>TruexAdRenderer</code></strong> and handle any events emitted by the renderer code. It will also need to call pause, resume and stop methods on the <strong><code>TruexAdRenderer</code></strong> when certain external events happen, like if the app is backgrounded or if the user has requested to exit the requested stream via back buttons.

It will also need to handle skipping ads in the current ad pod, if it is notified to do so.


# **<span style="text-decoration:underline;">2. Product Flows</span>**

There are two distinct product flows supported by TruexAdRenderer: Sponsored Stream (full-stream ad-replacement) and Sponsored Ad Break (mid-roll ad-replacement).

In a Sponsored Ad Break flow, once the user hits a mid-roll break with a TrueX tag flighted, they will be shown a "choice-card" offering them the choice between watching a normal set of video ads or a fully interactive trueX ad:

 \


![choice card](media/choice_card.png)


**_Fig. A  true[X] mid-roll choice card_**

If the user opts for a normal ad break, or if the user does not make a selection before the 15-second countdown timer expires, the TrueX UI will close and playback of normal video ads can continue as usual.

If the user opts to interact with TrueX, an interactive ad unit will be shown to the user: \




![ad](media/ad.png)


**_Fig. B  true[X] interactive ad unit_**

The requirement for the user to "complete" this ad is for them to spend at least 30 seconds on the unit and for at least one interaction (tapping the Siri remote or navigating anywhere through the ad).

Once the user fulfills both requirements, an "I'm Done" button will appear in the top right, which the user can select to exit the TrueX ad. Having completed a trueX ad, the user will be returned directly to content, skipping over rest of the current mid-roll ad payload.

The Sponsored Stream flow is quite similar. In this scenario, a user will be shown a choice-card in the preroll: \




![choice card](media/choice_card.png)


**_Fig. C  true[X] preroll choice card (full-stream replacement)_**

Similarly, once if the user opts-in and completes the TrueX ad, they will be skipped over the remainder of the preroll ad break. However, every subsequent mid-roll break in the current stream will also be skipped over.

For each following mid-roll ad break, the user will be shown a "hero card" (also known as a "skip card"): \




![skip card](media/skip_card.png)


**_Fig. D  true[X] mid-roll skip card_**

This messaging will be displayed to the user for several seconds, after which they will be returned directly to content.


# **<span style="text-decoration:underline;">3. How to use TruexAdRenderer</span>**

When referring to data returned via Uplynk Preplay, I will reference this example JSON response in order to clearly explain which piece of data is required: 


```
    https://gist.github.com/jalbini/7c0f908064ae1468beb0e4a11fd187ec
```


I will refer to entire object as **<code>response</code></strong>, and will point to specific pieces of data using standard JSON notation. 


### **3.1 When to show True[X]**

Upon receiving an ad schedule from Uplynk, you should be able to detect whether or not TrueX is returned in any of the pods. TrueX ads should have `apiFramework` set to "`VPAID`". 

In the example response, you will see those values at **<code>response.ads.breaks[0].ads[0].apiFramework</code></strong> and <strong><code>response.ads.breaks[0].ads[0].creative</code></strong> respectively.

The Uplynk PrePlay documentation will have more information on determining exactly when the VPAID ad unit should fire -- from what I've ascertained, you'd simply reference **<code>response.ads.placeholderOffsets</code></strong>, using <code>adIndex</code> and <code>breakIndex</code> to correlate them to each separate VPAID unit.

Once the player reaches a placeholder, it should pause, instantiate the `TruexAdRenderer` and immediately call `init()` followed by `start()`.

Alternatively, you can instantiate and `init()` the `TruexAdRenderer` in preparation for an upcoming placeholder. This will give the `TruexAdRenderer` more time to complete its initial ad request, and will help optimize TrueX load time. Once the player reaches a placeholder, it can call start() to notify the renderer that it can display the unit to the user.


### **3.2 Handling Events from TruexAdRenderer**

Once `start()` has been called on the renderer, it will start to emit events (A full list of events is available in Section 4.2)

One of the first events you will receive is **<code>AD_STARTED</code></strong>. This notifies the app that the renderer has received an ad for the user and has started to show the unit to the user. The app does not need to do anything in response, however it can use this event to facilitate a timeout. If an <strong><code>AD_STARTED</code></strong> event has not fired within a certain amount of time, the app can call <code>stop()</code> on the renderer and proceed to normal video ads. 

If there were no ads available to the user or if there was an error making the ad request, **<code>NO_ADS_AVAILABLE</code></strong> or <strong><code>AD_ERROR</code></strong> will fire respectively. At this point, the app should resume playback without skipping any ads, so the user receives a normal video ad payload.

Another important event to listen for is **<code>AD_FREE_POD</code>. </strong>This signifies that the user has earned a credit with TrueX and, once the renderer signals it is complete, all linear video ads remaining in the current pod should be skipped. It's important to note that the player should not immediately resume playback once receiving this event -- rather it should note that it was fired and continue to wait for an <strong><code>AD_COMPLETED</code></strong> event.

The last major event to fire will be an **<code>AD_COMPLETED</code></strong> event. This will signal to the app that the renderer has finished showing the TrueX unit to the user. The user may have opted out of TrueX, let the choice card countdown expire, or may have completed a TrueX ad unit and earned an ad-free credit (an <strong><code>AD_FREE_POD</code></strong> event would have already been fired in the last case). At this point, the app should check if an <strong><code>AD_FREE_POD</code></strong> event has fired -- if so, the user should be fast-forwarded past all remaining linear video ads and playback should resume. If not, playback still should resume, but without fast-forwarding so the user receives the remaining payload of video ads.


### **3.3 Handling Ad Elimination**

Skipping video ads is completely the responsibility of the app code. The Uplynk Preplay API should provide enough information for the app to determine where the current pod end-point is, and, when appropriate, should fast-forward directly to this point when resuming playback.


# 


# **<span style="text-decoration:underline;">4. TruexAdRenderer tvOS API</span>**

This is an outline of public `TruexAdRenderer` methods and delegate methods (referred to as events). 


## **<span style="text-decoration:underline;">4.1 TruexAdRenderer Methods</span>**

When referring to data returned via Uplynk Preplay, I will reference this example JSON response in order to clearly explain which piece of data is required: 


```
    https://gist.github.com/jalbini/79a6fcd4e12049b6f5ee1d883d372b81
```


I will refer to entire object as **<code>response</code></strong>, and will point to specific pieces of data using standard JSON notation. 


### 
    **4.1.1 initWithUrl()**


```


##     - (id)initWithUrl:(NSString *)creativeUrl adParameters:(NSDictionary*)adParameters slotType:(NSString *)slotType optimizelyProjectId:(NSString*)optimizelyProjectId optimizelyUserId:(NSString*)optimizelyUserId
```



        This method will be called by the app code in order to instantiate the `TruexAdRenderer`. The renderer will parse out the `creativeURL` and `adParameters` passed to it and make a request to the TrueX ad server to see what ads are available.


        You may instantiate `TruexAdRenderer` early (a few seconds before the next pod even starts) in order to give it extra time to make the ad request.


        The parameters for this method call are:



*   **creativeURL:** TrueX asset url returned by Uplynk. In the example response, this would correspond to **<code>response.ads.breaks[0].ads[0].creative</code></strong>
*   <strong>adParameters</strong>: AdParameters as returned by Uplynk. In the example response, this would correspond to <strong><code>response.ads.breaks[0].ads[0].adParameters</code></strong>
*   <strong>podType</strong>: the type of the current ad pod, <strong><code>PREROLL</code></strong> or <strong><code>MIDROLL</code></strong>
*   <strong><code>optimizelyProjectId</code></strong>: (OPTIONAL) only pass in an Optimizely Project ID if you intend to integrate the functionality of TruexAdRenderer with your existing Optimizely experimentation instance. Otherwise please pass in <strong><code>nil</code></strong>.
*   <strong><code>optimizelyUserId</code></strong>: (OPTIONAL) only pass in an Optimizely User ID if you intend to integrate the functionality of TruexAdRenderer with your existing Optimizely experimentation instance. Otherwise please pass in <strong><code>nil</code></strong>.

### 
    <strong>4.1.2 start()</strong>


    ```


##     - (void)start:(UIViewController *)baseViewController
    ```



        This method will be called by the app code when the TrueX unit is ready to be displayed to the user. This can be called anytime after the unit is initialized. 


        The app should have as much extraneous UI hidden as possible, including player controls, status bars and soft buttons/keyboards, where possible.


        Once `start()` is called, the renderer will wait for the ad request triggered in `init()` to finish. 


        If the request returns an ad, the renderer will immediately instantiate a ViewController and present it on the baseViewController. Once this is complete, it will fire the **<code>AD_STARTED</code></strong> event. The user will be shown the TrueX choice card once all relevant assets have loaded. <strong><code>AD_FREE_POD</code> </strong>and<strong> <code>AD_COMPLETED</code></strong> may fire after this point, depending on the user's choices.


        If the request returns no ads, the renderer will fire the **<code>NO_ADS_AVAILABLE</code></strong> event.


        If the request signals that the user is in an ad-free state, then the renderer will immediately instantiate a ViewController and present it on the baseViewController. Once this is complete, it will fire the **<code>AD_STARTED</code></strong> event. After 3 seconds of a "skip card" being shown to the user, the <strong><code>AD_FREE_POD</code></strong> event will fire, followed immediately by the <strong><code>AD_COMPLETED</code></strong> event.


        If the request fails, the renderer will fire the **<code>AD_ERROR</code></strong> event.


        The parameters for this method call are:

*   **baseViewController:** The viewController that `TruexAdRenderer` should present its ViewController to. The baseViewController should cover the entire screen.

### 
    **4.1.3 stop()**


    ```


##     - (void)stop
    ```



        The `stop()` method is only called when the app needs the renderer to immediately stop and destroy all related resources.


        For example, if a user backs out of the video stream to return to the normal app UI (perhaps by using a back button) or if there was an unrelated error that requires immediate halt of the current ad unit.


        In contrast to `pause()`, there is no way to resume the ad after `stop()` is called.


### 
    **4.1.4 pause()**


    ```


##     - (void)pause
    ```



        `pause()` is required whenever the renderer needs to pause the choice card or ad unit (including all video/audio and countdowns).


### 
    **4.1.5 resume()**


    ```


##     - (void)resume
    ```



        `resume()` should be invoked when ad playback can resume



## **<span style="text-decoration:underline;">4.2 TruexAdRendererDelegate methods  </span>**

This delegate protocol can be found in **<code>TruexShared.h</code></strong>


### 
    **4.2.1 onAdStarted()**


```


##     -(void) onAdStarted:(NSString *)campaignName
```



        This event will fire in response to the `start()` method when the TrueX UI is ready and has been added to the view hierarchy.


        The app code may use this event in order to facilitate a timeout on TrueX load.


        The parameters for this method call are:



*   **campaignName:** The name of the ad campaign available to the user (e.g. "_US Air Force - Special OPS (TA M) - Q1 2017_")

### 
    **4.2.2 onAdCompleted()**


    ```


##     -(void) onAdCompleted:(NSInteger)timeSpent
    ```



        This event will fire when the TrueX unit is complete -- at this point, the app should resume playback and remove all references to the `TruexAdRenderer` instance.


        Here are some examples where **<code>AD_COMPLETED</code></strong> will fire:

*   User opts for normal video ads (not TrueX)
*   15 second choice card countdown runs out
*   User completes TrueX ad unit (or taps I'm done)
*   After a "skip card" has been shown to a user for 3 seconds

        The parameters for this method call are:

*   **timeSpent:** The amount of time (in seconds) the user spent on the TrueX interactive ad unit -- set to `0` if the user did not earn an ad free credit or if the user was shown a "skip card".

### 
    **4.2.3 onAdError()**


    ```


##     -(void) onAdError:(NSString *)errorMessage
    ```



        This event will fire when the TrueX unit has encountered an error it cannot recover from. The app code should handle this the same way as an **<code>AD_COMPLETED</code></strong> event -- resume playback and remove all references to the <code>TruexAdRenderer</code> instance.


### 
    **4.2.4 onNoAdsAvailable()**


    ```


##     -(void) onNoAdsAvailable
    ```



        This event will fire when the TrueX unit has determined it has no ads available to show the current user. The app code should handle this the same way as an **<code>AD_COMPLETED</code></strong> event -- resume playback and remove all references to the TruexAdRenderer instance.


### 
    **4.2.5 onAdFreePod()**


    ```


##     -(void) onAdFreePod
    ```



        This event will fire when the all remaining ads in the current ad pod need to be skipped. The app code should notate that this event has fired, but should not take any further action until it receives an **<code>AD_COMPLETED</code></strong> or <strong><code>AD_ERROR</code></strong> event.


_<span style="text-decoration:underline;">All following events are used mostly for tracking purposes -- no action is generally required:</span>_


### 
    **4.2.7 onOptIn() _(optional)_**

	`-(void) onOptIn:(NSString *)campaignName adId:(NSInteger)adId `


        This event will fire if the user selects to interact with the TrueX interactive ad.


        Note that this event may be fired multiple times if a user opts in to the TrueX interactive ad and subsequently backs out.


### 
    **4.2.8 onOptOut() _(optional)_**


```
-(void) onOptOut:(BOOL)userInitiated
```



        This event will fire if the user opts for a normal video ad experience -- `userInitiated` will be set to `true` if this was actively selected by the user, `false` if the user simply allowed the choice card countdown to expire.


### 
    **4.2.9 onSkipCardShown() _(optional)_**


```
-(void) onSkipCardShown
```



        This event will fire anytime a "skip card" is shown to a user as a result of completing a TrueX Sponsored Stream interactive in an earlier preroll.


### 
    **4.2.9 onUserCancel() _(optional)_**


```
-(void) onUserCancel
```



        This event will fire when a user backs out of the TrueX interactive ad unit after having opted in. This would be achieved by tapping the "Nevermind" link inside the TrueX interactive ad. The user will be subsequently taken back to the Choice Card (with the countdown timer reset to full).


        Note that after a **<code>USER_CANCEL</code></strong>, another <strong><code>OPT_IN</code></strong> or <strong><code>OPT_OUT</code></strong> event may be fired.


### 
    **4.2.10 onUserCancelStream() _(optional)_**


```
-(void) onUserCancelStream
```



        This delegate represents that a user has decided to cancel the stream entirely. The app, at this point, should treat this the same way it would handle any other "exit" action from within the stream -- in most cases this will result in the user being returned to an episode/series detail page.


        This is a "terminal" delegate with similar ramifications as **<code>AD_COMPLETED</code></strong> -- meaning that once this delegate is fired, the app can safely remove all references to <strong><code>TruexAdRenderer</code></strong>. The only difference is that the app should exit the stream, rather than resume playback.


        If this optional delegate is not implemented, certain UI elements and user flows may be excluded from the trueX flow, and any attempts for the user to exit the trueX ad via the "MENU" button will result in  **<code>AD_COMPLETED</code></strong> being fired as they were in previous versions (< 3.2.0) of TruexAdRenderer.


## **<span style="text-decoration:underline;">4.3 TruexAdRenderer Constants</span>**

These constants will be available in **<code>TruexConstants.h</code></strong>.


### 
    **4.3.1 PREROLL**


        This value is used to pass into the `init()` call, for the `podType` parameter


### 
    **4.3.2 MIDROLL**


        This value is used to pass into the `init()` call, for the `podType` parameter
