Titanium Facebook Module for Android
=====================================

Overview
------------
* Please read thie carefully since this module is (by necessity) different from previous Facebook Android modules
* This module is based on Facebook's latest Android SDK (3.18.0, currently), and in accordance with Facebook's samples and recommendations.
* The API is similar to the iOS Facebook module I wrote, differing in some places where required.
* The module uses the Facebook SDK with no modifications

Current Functionality
---------------------
* Login/logout (Read Facebook docs for changes in this space)
* Additional permissions requests
* Graph API (version 2.1, read Facebook's docs for the changes, in particular regarding the user_friends permission).
* Share dialog
* Additional functionality will be implemented over time, pull requests greatly appreciated

Installation Details
--------------------
* This minimum required SDK version is 3.5.0
* You must implement this pull request: [synchronous activity callbacks](https://github.com/appcelerator/titanium_mobile/pull/6160)
* In tiapp.xml or AndroidManifest.xml you must declare the following inside the `<application>` node
`<activity android:name="com.facebook.LoginActivity" android:theme="@android:style/Theme.Translucent.NoTitleBar" 
	android:label="YourAppName"/>`
* You must also reference the string containing your Facebook app ID, inside the `<application>` node as well: 
`<meta-data android:name="com.facebook.sdk.ApplicationId" android:value="@string/app_id"/>`
* The app id goes into the the file `/platform/android/res/values/strings.xml`, where you should define
`<resources><string name="app_id">1234567890123456</string></resources>`, where the number is of course the app ID. The app ID is not set programmatically.

Module Versioning
-----------------

x.y.zt, where x.y.z is the Facebook iOS SDK version, t denotes the Titanium module version for this SDK.
This module version is 3.18.01 - i.e. uses Facebook Android SDK 3.18.0

Proxy required per Android activity
-----------------------------------
Unlike iOS, where the entire app is active in memory, in Android only a single Activity is active at any time. An Activity corresponds to a Ti.UI.Window or Ti.UI.TabGroup. The Facebook SDK contains tools to synchronize state between the various activities in the app, and this module implements that functionality, but for this to work we need to tell the module which is the currently active Activity. Thus the following is required:
* All Windows/TabGroup that require Facebook functionality must create a proxy: `var fb = fbModule.createActivityWorker();`
* To tell the module which is the current activity we must call `fb.activityResumed();` during the window/tabGroup.activity.onResume function. See code example below.

Module API
----------

* Require the module: `var fbModule = require('com.ti.facebook');`
* Create a proxy for each Activity (Window or TabGroup) that needs Facebook functionality: `var fb = fbModule.createActivityWorker();`
* You must call `fb.activityResumed();` during the Activity's onResume function. See example below. This may change when [TIMOB-15443](https://jira.appcelerator.org/browse/TIMOB-15443) is resolved. Until then this call is required, as is this fix to the SDK: [sync activity lifecycle calls](https://github.com/appcelerator/titanium_mobile/pull/6160)
* The permissions array may be set by calling `setPermissions(['public_profile', 'email', etc])` on the proxy object, or in the proxy creation dictionary `var fb = fbModule.createActivityWorker({permissions: [.....]});`, or passed to the `initialize` function detailed below. Note that these are just the requested read permissions, and not necessarily the permissions granted to the app. These permissions will only be used when initially authenticating with Facebook. You need to set the permissions only once, for the initial proxy used in the app.
* The actual permissions granted to the app may be read at any time by checking `var permissions = fb.permissions;` 
* Add login and logout event listeners on the proxy object. The syntax and functionality is identical to the current Titanium Facebook module.
* After setting up the login and logout listeners, call `fb.initialize([optional permissions array]);`. If there is a cached Facebook session available, the login event will be fired immediately. `initialize` should only be called once, for the initial proxy in the app.
* `fb.requestNewReadPermissions([new read permissions], function(e) {....});` The callback will indicate `success`, `error` or `cancelled`. If `success`, then you need to get `fb.permissions` to check the actually active permissions on the session.
* `requestNewPublishPermissions` - same as for requestNewReadPermissions. Note these functions take on added importance since users can decline individual permissions when initially logging in, except for `public_profile` which is mandatory.
* `fb.requestWithGraphPath` - same as in Appcelerator's Titanium module.
* Share Dialog: see below.

Events and error handling
-------------------------

The `login` event behavior is normalized to behave similarly to iOS. If you're logged in after initialization,
you *will* get a `login` event. It's important to understand the reason for this: The module checks the SDK for a cached token.
However, there is a possibility this token is invalid without the SDK's knowledge (e.g. the user changed her password, etc).
Thus, the only way to verify the token's and session's validity is to make a call to Facebook's servers. 
You will not get the `login` event if there was no cached session. 
In that case - the module will close the session. Below is the integrated flow for both Android and iOS:
```javascript
var fbModule = require('com.ti.facebook');
var fb = fbModule.createActivityWorker();
window.activity.onResume = function() {
	fb.activityResumed();
};
fb.permissions = ['public_profile', 'email', 'user_friends'];
// now set up listeners
fb.addEventListener('login', function(e) {
	if(e.success) {
		// do your thang.... 
	} else if (e.cancelled) {
		// login was cancelled, just show your login UI again
	} else if (e.error) {
		if (Ti.Platform.name === 'iPhone OS') {
			var loginAlert = Ti.UI.createAlertDialog({title: 'Login Error'});
			if (e.error.indexOf('OTHER:') !== 0){
				// guaranteed a string suitable for display to user
				loginAlert.message = e.error;
			} else {
				//alert('Please check your network connection and try again.');
				// after "OTHER:" there may be useful error data, not suitable for user display
				loginAlert.message = 'Please check your network connection and try again';
			}
		} else {
			loginAlert.message = e.error;
		}
	} else {
		// if not success, nor cancelled, nor error message then pop a generic message
		// e.g. "Check your network, etc" . This is per Facebook's instructions
	}
});

fb.addEventListener('logout', function(e)........
		
fb.initialize(); // the module will do nothing prior to this. This enabled you to set up listeners anywhere in the app	

if (!fb.loggedIn) {
	// you want to show your login UI in this case
	// if the user is loggedIn, just wait for the login event
	// when you want the user to login, call fb.authorize()
}	
```

Share Dialog
-------------

See the [Facebook docs](https://developers.facebook.com/docs/android/share-dialog/)
Use it! You don't need permissions, you don't even need the user to be logged into your app with Facebook!
*	First check if you can use it - check the properties `fb.canPresentShareDialog` or `fb.canPresentOpenGraphActionDialog`, depending upon your desired sharing action.
Unfortunately this is less concise than the iOS module, due to the SDK.
*	To share a user's status just call `fb.share({});` Note: this is documented for iOS but not Android, so use with caution.
*	To share a link call `fb.share({url: 'http://example.com' });`
*	To post a graph action call:

```javascript
fb.share({url: someUrl, namespaceObject: 'myAppnameSpace:graphObject', objectName: 'graphObject', imageUrl: someImageUrl, 
		title: aTitle, description: blahBlah, namespaceAction: 'myAppnameSpace:actionType', placeId: facebookPlaceId}`
```
For the graph action apparently only placeId is optional. Note: There appears to be [a bug](https://developers.facebook.com/bugs/363119770486799) currently where place tagging doesn't work for Android.


Feel free to comment and help out! :)
-------------------------------------
