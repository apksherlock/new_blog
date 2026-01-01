---
title: "Android apps as reconnaissance tools"
datePublished: Thu Nov 28 2024 23:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cmjvk9lxw000g02l2apq263s4
slug: android-apps-as-reconnaissance-tools
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1737919023389/0c63595d-2afc-4618-b9f9-8079b8e16b39.jpeg

---


When targeting\[1\] a web surface—whether it’s a web application or a web server API—gathering intelligence and information is a crucial step before constructing payloads for any identified vulnerability. If the reconnaissance phase is executed correctly, the likelihood of errors during stages of the attack is significantly reduced.

Tools like [Burp Suite](https://portswigger.net/burp), [OWASP ZAP,](https://www.zaproxy.org/) and others allow testers to map a web application effectively. These tools enable both automated and manual mapping, helping testers gather critical information about the application. The success of this phase depends not only on the quality of the tools used but also on the tester’s expertise. By utilizing them, testers can start looking at potential vulnerabilities in the application’s directory structure. However, additional reconnaissance can often uncover details that may have been missed initially.

As many of us know, mobile applications are essentially web applications packaged in a native framework. In essence, they typically follow the same architectural structure as their web counterparts, connecting to the same endpoints and accessing identical data. This makes knowledge gained from web application testing highly transferable to mobile pentesting.

With that said, one unique aspect of mobile applications is the implementation of deep links. These are essentially routing configurations defined in the app’s entry point (`AndroidManifest.xml`), mapping browser links to specific native application pages. While deep links themselves can be a subject for attacks, when implemented correctly, they pose minimal risk.

Here is a very simple example how a deep link could be implemented:

```xml
<data android:scheme="https"
    android:host="api.example.com"/>
```

*This is always part of an* `Activity`.

However, this alone does not secure the application against potential attacks. An attacker could exploit this configuration by constructing payloads that use additional, unspecified paths under `https://api.example.com`. Blacklisting is not a viable defense mechanism here since attackers can generate an infinite number of path variations.

To mitigate this risk, developers typically use a whitelisting approach, explicitly specifying which patterns are allowed. While this can result in a large number of `<data>` blocks, it is the most effective way to secure the application. Below is an example:

```xml
<data android:scheme="https"
    android:host="api.example.com" />
 
    <data android:pathPattern="/auth/login" />
    <data android:pathPattern="/auth/register" />
    <data android:pathPattern="/auth/reset-password" />
 
    <data android:pathPattern="/users" />
    <data android:pathPattern="/users/.*" />
    <data android:pathPattern="/users/.*/profile" />
    <data android:pathPattern="/users/.*/settings" />
 
    <data android:pathPattern="/products" />
    <data android:pathPattern="/products/.*" />
    <data android:pathPattern="/products/.*/details" />
    <data android:pathPattern="/products/.*/reviews" />
 
    <data android:pathPattern="/orders" />
    <data android:pathPattern="/orders/.*" />
    <data android:pathPattern="/orders/.*/details" />
    <data android:pathPattern="/orders/.*/status" />
 
    <data android:pathPattern="/support/tickets" />
    <data android:pathPattern="/support/tickets/.*" />
    <data android:pathPattern="/support/tickets/.*/messages" />
    <data android:pathPattern="/support/tickets/.*/status" />
 
    <data android:pathPattern="/notifications" />
    <data android:pathPattern="/notifications/.*" />
    <data android:pathPattern="/notifications/.*/read" />
    <data android:pathPattern="/notifications/.*/delete" />
 
    <data android:pathPattern="/settings" />
    <data android:pathPattern="/settings/preferences" />
    <data android:pathPattern="/settings/security" />
    <data android:pathPattern="/settings/privacy" />
 
    <data android:pathPattern="/analytics" />
    <data android:pathPattern="/analytics/reports" />
    <data android:pathPattern="/analytics/reports/.*/details" />
    <data android:pathPattern="/analytics/reports/.*/export" />
```

Constructing deep links this way, makes it very difficult to tamper with an Android application. Any attempt to contact the app via a deep link that is not listed as above, would result into an error log from the `Android` system (the malicious app would basically crash).

However, this detailed configuration indirectly becomes a valuable resource for reconnaissance. By decompiling the APK, testers can discover paths that may not have been identified during the web application’s mapping or scanning phases.

Tools like [apkleaks](https://github.com/dwisiswant0/apkleaks) perform a thorough scan of the entire `.apk` with the intention of extracting secrets that could be helpful during the reconnaissance phase. This includes deep links and much more: hardcoded strings, string resources etc.

![](https://user-images.githubusercontent.com/25837540/111927529-a4ade080-8ae3-11eb-800a-b764ab1242e1.jpg align="left")

The usage is pretty straightforward:

```bash
$ apkleaks -f ~/path/to/file.apk -o results.txt
```

While it’s common practice to investigate a mobile app from a fresh perspective—aiming to tamper with or attack it—there are numerous examples where mobile apps have served as critical entry points for larger-scale attacks.

As demonstrated above, mobile apps are not just potential black-box vulnerabilities for testers to exploit. They can also serve as pieces of a larger puzzle, often revealing valuable information about web applications or backend servers.

Stavro Xhardha

*Footnotes*

*\[1\] The mention of “attacks” in this article, as well as across the entire blog, refers exclusively to scenarios where the tester is authorized to conduct such activities.*
