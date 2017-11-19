# Overview

[Cisco Spark iOS SDK](https://developer.ciscospark.com/sdk-for-ios.html) enables you to embed [Cisco Spark](https://www.ciscospark.com/) calling and meeting experience into your iOS mobile application. The SDK provides APIs to make and receive audio/video calls. In order to receive audio/video calls, the user needs to be notified when someone is calling the user.

This sample Webhook/Push Notification Server demonstrates how to write a server application to receive [Incoming Call Notification](https://developer.ciscospark.com/sdk-for-ios.html) from Cisco Spark and use [Apple Push Notification Service](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html#//apple_ref/doc/uid/TP40008194-CH8-SW1) to notify the mobile application.

This sample is built upon [apns4j](https://github.com/teaey/apns4j) and [Sprint Boot](https://projects.spring.io/spring-boot). It is designed to be deployed and run on [Heroku](https://www.heroku.com). But it can be deployed and run on other environments with minimal changes.

For more information about iOS remote notification, please see [Apple developer guide](https://developer.apple.com/notifications/).

# How it works

Assuming this sample Webook/Push Notification Server has been deployed on the public Internet, the following describes the webhooks and push notification workflow step by step.

![Spark-IOSSDK-APNS](https://dsc.cloud/hello/Spark-IOSSDK-APNS-1509615302.png)

1. Register to the Apple Push Notification Service (APNs) when your iOS application is launching.

2. The APNs returns a device token to the application.

3. After the user logs into Cisco Spark，use [Webhook API](https://ciscospark.github.io/spark-ios-sdk/Classes/WebhookClient.html) to create an webhook at Cisco Spark cloud. The target URL of the webhook must be the /webhook REST endpoint of this server. The URL has to be publicly accessible from the Internet.
	```
	spark.webhooks.create(name: "Message Webhook", targetUrl: targetUrl, resource: "messages", event: "all") { response in
		switch response.result {
		case .success(let webhook):
			// ...
		case .failure(let error):
			// ...
		}
	}
	```

4. Register the device token returned by the APNs and the user Id of current user to the  Webhook/Push Notification Server. The Server stores these information locally in a database.
	```
	let paramaters: Parameters = [
		"email": email,
		"voipToken": voipToken,
		"msgToken": msgToken,
		"personId": personId
	]
	Alamofire.request("https://example.com/register", method: .post, parameters: paramaters, encoding: JSONEncoding.default).validate().response { res in
		// ...
	}
	```

5. The remote party makes a call via Cisco Spark.

6. Ciso Spark receives the call and triggers the webhook. The incoming call event is sent to the target URL, which should be /webhook REST endpoint of this Webhook/Push Notification server.

7. The Webhook/Push Notification Server looks up the device token from the database by the user Id in the incoming call event, then sends the notification with the device token and incoming call information to the APNs.

8. The APNs pushs notification to the iOS device.

9. Your iOS application [gets the push notification](https://github.com/ciscospark/spark-ios-sdk-example-buddies/blob/9a0d51a9f6564f04e84d3868815b366ccb11425e/Buddies/AppDelegate.swift#L170) and uses the SDK API to accept the call from Spark Cloud.

For more details about Step 1 and 2, please see [Apple Push Notifications Guide]((https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/index.html#//apple_ref/doc/uid/TP40008194-CH3-SW1))

For more details about Step 3 and 6, please see Cisco Spark [Webhooks Explained](https://developer.ciscospark.com/webhooks-explained.html)

# Deployment

The sample application can be easily deployed as a [Java application on the Heroku](https://devcenter.heroku.com/categories/java).

1. Create an Herko account and [set up](https://devcenter.heroku.com/articles/getting-started-with-java#set-up) the Heroku environment.

2. Create a new Heroku app.

3. Clone the [sample code](https://github.com/ciscospark/spark-ios-sdk-example-push-notification-server/) to a local directory.
	```
	git clone git@github.com:ciscospark/spark-ios-sdk-example-push-notification-server.git 
	```

4. Copy your Apple Push Certificates to `./spark-ios-sdk-example-push-notification-server/.jdk-overlay/jre/lib/security`.

	Sending and receiving push notifications requires you to create Apple Push Certificates. For this sample application, you should create and upload three certificates corresponding, one for each type: Development, Production and VoIP Services.
	
	Apple Push Certificates are generated from the [Apple Developer Member Center](https://developer.apple.com/account/overview.action) which requires a valid Apple ID to login. 
	
5. Deploy the application to Heroku.
	```
	heroku git:remote -a YOUR_APP_NAME
	git add .
	git commit -am "First Deploy"
	git push heroku master
	```

# REST API endpoints and Usage

The sample Webhook/Push Notification server provides three REST API endpoints.

* `POST /webhook` -- This REST API endpoint should be used as the target URL for Cisco Spark [Webhooks Explained](https://developer.ciscospark.com/webhooks-explained.html). Cisco Spark post the incoming call event to this endpoint.

	Please see [the implementation](https://github.com/ciscospark/spark-ios-sdk-example-push-notification-server/blob/6cd76f222c3c438ade55ae1cde8b8616e311da61/src/main/java/com/ciscospark/iossdk/example/pns/Main.java#L117) for more details.

* `POST /register` -- This REST API endpoint should be used by the moible application to register the device token and user id to this sample server.

	Please see [the implementation](https://github.com/ciscospark/spark-ios-sdk-example-push-notification-server/blob/6cd76f222c3c438ade55ae1cde8b8616e311da61/src/main/java/com/ciscospark/iossdk/example/pns/Main.java#L163) for more details.
	
* `DELETE /register/{device_token}` -- This REST API endpoint should be used to delete the registered device token.

	Please see [the implementation](https://github.com/ciscospark/spark-ios-sdk-example-push-notification-server/blob/6cd76f222c3c438ade55ae1cde8b8616e311da61/src/main/java/com/ciscospark/iossdk/example/pns/Main.java#L182) for more details.