DotNetPush v5.0
===============

DotNetPush is a server-side library for sending Push Notifications to iOS/OSX (APNS) and
Android/Chrome (GCM/FCM), a fork of PushSharp.

 - Read more on my blog http://redth.codes/pushsharp-3-0-the-push-awakens/

---

## Sample Usage

The API in v3.x+ series is quite different from 2.x.  The goal is to simplify things and focus on the core functionality of the library, leaving things like constructing valid payloads up to the developer.

### APNS Sample Usage
Here is an example of how you would send an APNS notification:

```csharp
// Configuration (NOTE: .pfx can also be used here)
var config = new ApnsConfiguration (ApnsConfiguration.ApnsServerEnvironment.Sandbox, 
    "push-cert.p12", "push-cert-pwd");

// Create a new broker
var apnsBroker = new ApnsServiceBroker (config);
    
// Wire up events
apnsBroker.OnNotificationFailed += (notification, aggregateEx) => {

	aggregateEx.Handle (ex => {
	
		// See what kind of exception it was to further diagnose
		if (ex is ApnsNotificationException notificationException) {
			
			// Deal with the failed notification
			var apnsNotification = notificationException.Notification;
			var statusCode = notificationException.ErrorStatusCode;

			Console.WriteLine ($"Apple Notification Failed: ID={apnsNotification.Identifier}, Code={statusCode}");
	
		} else {
			// Inner exception might hold more useful information like an ApnsConnectionException			
			Console.WriteLine ($"Apple Notification Failed for some unknown reason : {ex.InnerException}");
		}

		// Mark it as handled
		return true;
	});
};

apnsBroker.OnNotificationSucceeded += (notification) => {
	Console.WriteLine ("Apple Notification Sent!");
};

// Start the broker
apnsBroker.Start ();

foreach (var deviceToken in MY_DEVICE_TOKENS) {
	// Queue a notification to send
	apnsBroker.QueueNotification (new ApnsNotification {
		DeviceToken = deviceToken,
		Payload = JObject.Parse ("{\"aps\":{\"badge\":7}}")
	});
}
   
// Stop the broker, wait for it to finish   
// This isn't done after every message, but after you're
// done with the broker
apnsBroker.Stop ();
```

#### Apple Notification Payload

More information about the payload sent in the ApnsNotification object can be found [here](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/TheNotificationPayload.html).


#### Apple APNS Feedback Service

For APNS you will also need to occasionally check with the feedback service to see if there are any expired device tokens you should no longer send notifications to.  Here's an example of how you would do that:

```csharp
var config = new ApnsConfiguration (
    ApnsConfiguration.ApnsServerEnvironment.Sandbox, 
    Settings.Instance.ApnsCertificateFile, 
    Settings.Instance.ApnsCertificatePassword);

var fbs = new FeedbackService (config);
fbs.FeedbackReceived += (string deviceToken, DateTime timestamp) => {
    // Remove the deviceToken from your database
    // timestamp is the time the token was reported as expired
};
fbs.Check ();
```


### GCM/FCM Sample Usage

Here is how you would send a GCM/FCM Notification:

```csharp
// Configuration GCM (use this section for GCM)
var config = new GcmConfiguration ("GCM-SENDER-ID", "AUTH-TOKEN", null);
var provider = "GCM";

// Configuration FCM (use this section for FCM)
// var config = new GcmConfiguration("APIKEY");
// config.GcmUrl = "https://fcm.googleapis.com/fcm/send";
// var provider = "FCM";

// Create a new broker
var gcmBroker = new GcmServiceBroker (config);
    
// Wire up events
gcmBroker.OnNotificationFailed += (notification, aggregateEx) => {

	aggregateEx.Handle (ex => {
	
		// See what kind of exception it was to further diagnose
		if (ex is GcmNotificationException notificationException) {
			
			// Deal with the failed notification
			var gcmNotification = notificationException.Notification;
			var description = notificationException.Description;

			Console.WriteLine ($"{provider} Notification Failed: ID={gcmNotification.MessageId}, Desc={description}");
		} else if (ex is GcmMulticastResultException multicastException) {

			foreach (var succeededNotification in multicastException.Succeeded) {
				Console.WriteLine ($"{provider} Notification Succeeded: ID={succeededNotification.MessageId}");
			}

			foreach (var failedKvp in multicastException.Failed) {
				var n = failedKvp.Key;
				var e = failedKvp.Value;

				Console.WriteLine ($"{provider} Notification Failed: ID={n.MessageId}, Desc={e.Description}");
			}

		} else if (ex is DeviceSubscriptionExpiredException expiredException) {
			
			var oldId = expiredException.OldSubscriptionId;
			var newId = expiredException.NewSubscriptionId;

			Console.WriteLine ($"Device RegistrationId Expired: {oldId}");

			if (!string.IsNullOrWhiteSpace (newId)) {
				// If this value isn't null, our subscription changed and we should update our database
				Console.WriteLine ($"Device RegistrationId Changed To: {newId}");
			}
		} else if (ex is RetryAfterException retryException) {
			
			// If you get rate limited, you should stop sending messages until after the RetryAfterUtc date
			Console.WriteLine ($"{provider} Rate Limited, don't send more until after {retryException.RetryAfterUtc}");
		} else {
			Console.WriteLine ("{provider} Notification Failed for some unknown reason");
		}

		// Mark it as handled
		return true;
	});
};

gcmBroker.OnNotificationSucceeded += (notification) => {
	Console.WriteLine ("{provider} Notification Sent!");
};

// Start the broker
gcmBroker.Start ();

foreach (var regId in MY_REGISTRATION_IDS) {
	// Queue a notification to send
	gcmBroker.QueueNotification (new GcmNotification {
		RegistrationIds = new List<string> { 
			regId
		},
		Data = JObject.Parse ("{ \"somekey\" : \"somevalue\" }")
	});
}
   
// Stop the broker, wait for it to finish   
// This isn't done after every message, but after you're
// done with the broker
gcmBroker.Stop ();
```

#### Components of a GCM/FCM Notification

GCM notifications are much more customizable than Apple Push Notifications. More information about the messaging concepts and options can be found [here](https://developers.google.com/cloud-messaging/concept-options#components-of-a-message).


License
-------
Copyright 2012-2016 Jonathan Dick

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
