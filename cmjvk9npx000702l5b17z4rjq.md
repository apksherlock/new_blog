---
title: "Reverse Engineering React Native Apps"
datePublished: Sat Dec 21 2024 23:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cmjvk9npx000702l5b17z4rjq
slug: reverse-engineering-react-native-apps
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1737919633531/c95b966f-2af3-473d-b316-e4fd0d303697.jpeg

---


Cross-platform frameworks are becoming increasingly popular. For mobile apps that do not require extensive interaction with hardware and primarily focus on accessing data sources or interacting with users, many businesses find it cheaper and faster to develop their apps using platforms like React Native, Flutter, Ionic, or Xamarin. Interestingly, reverse engineering these cross-platform compiled apps is more complex than reverse engineering natively compiled APKs. The primary reason for this is that most APK decompilers primarily revert machine code into Smali, which is then converted back into Java bytecode. This bytecode is relatively easy to represent in Java, making it more readable for reverse engineers.

But what about apps compiled natively, such as those built with React Native or Flutter? Fortunately, there is hope for reverse engineering these as well. In this post, we will explore the case for React Native applications.

## **Recon**

As usual, the very first step in reverse engineering Android applications is examining the **AndroidManifest.xml** file. When an app is built using a framework like React Native, the engineer can often spot keywords such as “facebook,” “reactnative,” or “expo” appearing in services, intent filters, and other components. Even if these keywords are not immediately apparent, the implementation of the **MainActivity** typically reveals the framework used:

```java
/* loaded from: classes5.dex */
public final class MainActivity extends ReactActivity {...}
```

The code under the package name being tested is not particularly helpful, as it primarily contains entry points for the real app to interact with Android. The actual implementation code is hidden elsewhere, though it is certainly located within the APK.

It is still worth mentioning that investigating this code could be valuable, as some classes might have been written in native Android for various reasons. Additionally, it is a good idea to run other reconnaissance tools, such as [apkleaks](https://apksherlock.com/2024/11/29/android-apps-as-reconnaissance-tools/) , to gather more information.

## **Reverse Engineering**

As we already know, APKs must contain information to instruct Android on how to execute the code for a specific app, so this information is always present. If we closely examine the `assets` folder after decompiling the APK using a preferred tool like `jadx` or `apktool,` we can find a special file called [`index.android`](http://index.android)`.bundle`. This file is the most crucial component of a compiled React Native application, as it contains all the code information.

However, when we attempt to read this file directly, we see raw bytes:

```bash
cat index.android.bundle
 
GM:@:@:@F��M�����F����c�����z��$&K\Ky{��7W?�'6T��F�����F��F������MRO�����in
����=�8=F��,1B�F����!������5�K�
```

This doesn’t provide much useful information, but fortunately, there are a few tools that can be used to extract meaningful data from the bundle.

## **First attempt: React Native Decompiler**

The simplest and most basic tool is an npm package called **react-native-decompiler**, or **rnd** for short. It is straightforward and easy to use:

```bash
rnd -i <index.android.bundle path> -o output_folder
```

However, most React Native apps nowadays are not that simple. The app I was trying to reverse engineer for a bug bounty was no exception. When I tried to decompile it with **rnd**, it immediately threw the following error:

```bash
[!] No modules were found!
[!] Possible reasons:
[!] - The React Native app is unbundled. If it is, export the "js-modules" folder from the app and provide it as the --js-modules argument
[!] - The bundle is a Hermes/binary file (ex. Facebook, Instagram). These files are not supported
[!] - The provided Webpack bundle input is not or does not contain the entrypoint bundle
[!] - The provided Webpack bundle was built from V5, which is not supported
[!] - The file provided is not a React Native or Webpack bundle.
```

There was potentially some room for further research, but based on past experiences, the second option was more likely true:

```bash
The bundle is a Hermes/binary file (ex. Facebook, Instagram). These files are not supported
```

As the documentation says: *Hermes is a JavaScript engine optimized for fast start-up of* [*React Native apps. It fe*](https://reactnative.dev/)*atures ahead-of-time static optimization and compact bytecode.*

Since it is a tool designed for ahead-of-time (AOT) optimization, similar to the Android AOT compile[r, it can be](https://reactnative.dev/) inferred that many modern React Native apps are likely using the Hermes compiler by default.

## **Second attempt: Hermes decompiler**

This tool is also fairly easy and straightforward to install, but it comes with some overhead in terms of usage. The decompiler itself can be found [here](https://github.com/P1sec/hermes-dec), though I personally prefer the [command-line tools for it](https://github.com/bongtrop/hbctool).

```bash
hbctool decomp <target index.android.bundle> -o output_folder
```

If successful, it should give the correct result:

```bash
ls -alt
drwxrwxr-x 8 apksherlock apksherlock     4096 Dec 22 20:14 ..
-rw-rw-r-- 1 apksherlock apksherlock 64754259 Dec 22 16:27 instruction.hasm
drwxrwxr-x 2 apksherlock apksherlock     4096 Dec 22 16:26 .
-rw-rw-r-- 1 apksherlock apksherlock  6744610 Dec 22 16:26 string.json
-rw-rw-r-- 1 apksherlock apksherlock 36127631 Dec 22 16:26 metadata.json
```

Unfortunately, the second attempt was also unsuccessful for me, as there appears to be a compatibility issue between the bytecode version and the compiled bundle. The latest official version is quite old (version 76), while this particular app was using version 96.

## **Third attempt: Hermes decompiler** bytecode compatibility

After conducting further research, I was able to find a fork of the original [**hbctool** (use w](https://github.com/Kirlif/HBC-Tool)ith caution) repository that supports many more versions.

*Note: Both tools are built in python and a special tool called* `poetry` is needed to built the package after cloning the repo.

There’s only one thing left to do at this point: retry the decompilation. This time, it worked:

```bash
hbctool decomp <target index.android.bundle> -o output_folder
```

I was able to see the `instructions.hasm` file, which contains the Hermes assembly representation of the JavaScript code written by the developer.

```bash
Function<global>0(1 params, 19 registers, 0 symbols):
    DeclareGlobalVar        UInt32:38294
    ; Oper[0]: String(38294) '__BUNDLE_START_TIME__'
    DeclareGlobalVar        UInt32:41952
    ; Oper[0]: String(41952) '__DEV__'
 
    DeclareGlobalVar        UInt32:201
    ; Oper[0]: String(201) 'process'
 
    DeclareGlobalVar        UInt32:38301
    ; Oper[0]: String(38301) '__METRO_GLOBAL_PREFIX__'
 
    CreateEnvironment       Reg8:3
    LoadThisNS              Reg8:4
    GetById                 Reg8:1, Reg8:4, UInt8:1, UInt16:37712
    ; Oper[3]: String(37712) 'nativePerformanceNow'
```

Assembly code can certainly be overwhelming, but it’s far better than having nothing. There’s a lot that can be done with it, such as:

* Extracting secrets
    
* Investigating hardcoded values
    
* Analyzing encryption and decryption methods
    
* Examining the information being sent to and from the backend
    
* Investigating error cases
    

## Conclusion

As a native Android developer myself, I hate to admit that a React Native app can be more difficult to reverse engineer than a native app written in Kotlin or Java. However, this does not mean that cross-platform solutions are bulletproof. The biggest vulnerability in any software is the developer — humans themselves. Therefore, the same mistakes that occur in native apps can also happen in cross-platform frameworks like React Native, Flutter, or Xamarin.
