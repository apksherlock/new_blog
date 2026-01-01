---
title: "Common Android Vulnerabilities"
datePublished: Sun Oct 27 2024 23:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cmjvk9qpj000302l5bg8r3w0b
slug: common-android-vulnerabilities
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1737922574674/8d728b49-8c6e-4605-ae22-f2ed7cce0cfe.jpeg

---


Nowadays, Android is everywhere—not just on our smartphones. More manufacturers are finding ways to integrate Android into everything from automotive infotainment systems to smart TVs and home security devices. There are good reasons for this trend. First, since Android is built on the Linux kernel, developers can easily transfer their skills and knowledge. Plus, Android comes packed with a wealth of pre-built components, which means developers can spend less time worrying about performance issues or reinventing the wheel for specific tasks like device compatibility or resource optimization. However, this widespread adoption also comes with its own set of security challenges. It’s crucial to address these challenges to keep all the different devices powered by Android safe and secure.

Let us now focus on some common security concerns I frequently encounter while performing penetration tests or hunting for bug bounties.

## Storing sensitive information insecurely

Take a look at the example below:

```kotlin
val prefs = context.getSharedPreferences("user_prefs", Context.MODE_PRIVATE)
prefs.edit().putString("password", "user_password").apply()
```

This code doesn’t just have one problem; it has two. First, it stores highly sensitive information (a password) in SharedPreferences, and second, it doesn’t implement any encryption.

While Android’s internal storage, where SharedPreferences is kept in `Context.MODE_PRIVATE`, restricts access from other apps (similar to a browser’s sandbox), this security is limited. If the device is rooted, this file could be accessed easily. Even if your app is secure now, future code changes and updates could introduce vulnerabilities over time. Storing a password in SharedPreferences is not recommended—even if you have implemented encryption. Furthermore, if you are encrypting data, it’s crucial to ensure that it is done correctly. But are you doing it correctly?

## **Inadequate Encryption**

Consider the following code:

```kotlin
val encoded = Base64.encodeToString("sensitive_data".toByteArray(), Base64.DEFAULT) 
prefs.edit().putString("encoded_data", encoded).apply()
```

How long would it take for an inexperienced hacker, who simply pulls your preferences file via ADB, to realize that this is merely basic encoding—without even needing to reverse engineer your Android app? And what about the following:

```kotlin
public class EncryptionUtil {
    private static final String ENCRYPTION_KEY = "mySuperSecretKey123";
 
    public static String encrypt(String data) {
        ...
    }
}
```

Even after reverse engineering your application, how difficult would it be for them to access sensitive user data if they encounter code like this?

Inadequate encryption is a major issue today. Developers often overlook best practices when it comes to proper encryption. This vulnerability is also highlighted in the [OWASP Top 10](https://owasp.org/www-project-mobile-top-10/2023-risks/m9-insecure-data-storage.html) for mobile as one of the most common security concerns.

## Insecure Communication

It might sound unbelievable, but many mobile apps still use HTTP to communicate with servers. This oversight leaves applications vulnerable to man-in-the-middle (MITM) attacks, where an attacker intercepts and potentially alters the communication between the app and the server. Just like personal computers, mobile phones are equally open to these types of attacks; therefore, it’s crucial to apply the same level of security measures to the network layer.

Here is one of many examples, that I have encountered while pentesting android mobile applications.

```json
//Revealed in Burpsuite after intercepting a POST method via HTTP
{
  "username": "someUserName",
  "password": "veryObviousPassword"
}
```

In a typical MITM attack, an attacker could intercept sensitive data such as usernames, passwords, or credit card information being transmitted over an unencrypted HTTP connection.

## Implicit Android Intents

Intents are Android’s serializable objects used for inter-process communication. Apps communicate with each other via intents, which can be classified into two types: explicit and implicit.

```kotlin
// Example of an explicit intent
val intent = Intent(this, TargetActivity::class.java)
intent.setData(someData)
startActivity(intent)
```

The snippet above demonstrates a case where the intent specifies the target activity as the intended receiver of the `someData` information. There is no inherent security issue here (unless the `TargetActivity` is exported and expects other data from outside, but that’s beyond the scope of this example).

However, there is a concerning scenario if not handled properly:

```kotlin
val intent = Intent("com.example.SEND_DATA")
intent.putExtra("userToken", userToken)
context.sendBroadcast(intent)
```

This intent is implicit; it does not have a specific target to which the broadcast should be sent. It is extremely easy for another malicious app to intercept this request:

```xml
<receiver android:name=".MaliciousReceiver">
            <intent-filter>
                <action android:name="com.example.SEND_DATA" />
            </intent-filter>
        </receiver>
```

```kotlin
class MaliciousReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        if (intent?.action == "com.example.SEND_DATA") {
            val userToken = intent.getStringExtra("userToken")
            Log.d("MaliciousApp", "Captured userToken: $userToken")
            userToken?.let { sendToRemoteServer(it) }
        }
    }
 
    private fun sendToRemoteServer(token: String) {
        ...
    }
}
```

In this case, sensitive data intended for a specific `BroadcastReceiver` has been intercepted and stolen by a malicious app.

## Webviews

WebViews are a significant security concern, particularly if not configured properly. They can be vulnerable to remote code execution. Consider the following example:

```kotlin
class WebAppInterface(private val context: Context) {
    @JavascriptInterface
    fun updateUserData(userName: String) {
        preferencesUtils.updateUserName(userName)
    }
}
 
// configuration
val webView = WebView(this)
webView.addJavascriptInterface(WebAppInterface(this), "Android")
webView.loadUrl("https://trusted-website.com")
```

If an attacker intercepts the network request, they could inject the following payload into the JavaScript section:

```xml
<script>Android.updateUserData("CLOWN")</script>
```

This scenario highlights a serious security risk associated with improperly implemented JavaScript interfaces in `WebView`s, which can lead to unauthorized data manipulation and other security breaches.

## Insufficient Input Validation

This is a classic case of insufficient input validation. In the snippet below:

```kotlin
val cursor = db.rawQuery("SELECT * FROM medical_data WHERE username = '$input'", null)
```

The code is vulnerable to SQL Injection (yes, it still exists).

A malicious app can steal all the records from this table if it can manipulate the `input` variable in a certain way. Let’s suppose the input is coming from an `EditText` or `TextInput`, and an attacker types:

```kotlin
' OR 1='1' --
```

This input would expose all the records in the `medical_data` table in the app’s database.

## Impropper logging

Improper logging can be simple, but it is just as easy to shoot yourself in the foot with logs. Let’s take a look at the code snippet below:

```kotlin
fun authenticateUser(username: String, password: String) {
    try {
        val token = authenticate(username, password)
        Log.d("Auth", "User authenticated successfully: $username")
    } catch (e: Exception) {
        Log.e("AuthError", "Authentication failed for user: $username. Token: $token", e)
    }
}
```

The use of `Log.d` would not be a major concern here, as debug logging is filtered out by the system in release builds. However, in the error case during a failed authentication, where the developer logs the error via `Log.e`, there is a significant problem.

Not only does this expose sensitive user data, but it also provides a hint about what went wrong. An attacker could definitely benefit from both of these issues, potentially exploiting not just the mobile client but also the server.

## Deep links

Deep links are one of the first areas web developers explore when pentesting mobile apps, as they are similar to web application penetration testing. Deep links allow the system to recognize that a certain link in the browser points to a specific screen in the Android app. However, significant mistakes can occur with deep links, making the app easily exploitable.

Consider the following snippet:

```xml
<activity android:name=".DeepLinkActivity" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="https"/>
        <data android:host="example.com"/>
        <data android:path="/open"/>
    </intent-filter>
</activity>
```

The `DeepLinkActivity` is exported and can be triggered by any app or browser that sends an intent matching the specified scheme, host, and path. If this activity handles sensitive data or performs critical actions without proper validation, it can be exploited.

An attacker could craft a malicious link that triggers the deep link and potentially performs unintended actions. For example:

```kotlin
val maliciousIntent = Intent(Intent.ACTION_VIEW)
maliciousIntent.data = Uri.parse("https://example.com/open?param=malicious")
startActivity(maliciousIntent)
```

If query parameters are not properly handled in the app (and in web applications they are easily discoverable), attackers have many combinations to exploit such vulnerabilities.

## Closing thoughts

Securing your Android app is crucial in today’s cyber battlefield. We’ve discussed several common vulnerabilities, from storing sensitive data insecurely to the risks of improper logging practices. These issues can leave your application open to attacks, and it’s essential to take them seriously.

If you’re developing a mobile app, it’s vital to follow best practices to protect user data and prevent potential exploits. But don’t worry; you don’t have to do this alone. If you’re looking for someone to help identify vulnerabilities in your app and strengthen its security, I’m here to assist.

Feel free to reach out if you want to discuss how I can help secure your mobile applications. Let’s ensure your app is safe for your users!
