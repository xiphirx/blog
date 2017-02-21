---
layout: post
title:  "The Perils of Loading Native Libraries on Android"
description: While implementing an encryption scheme for the Keepsafe Photos app, I encountered some very peculiar errors.
date:   2015-11-07 23:10:00 -0800
tags: android jni relinker crypto
---

*[This article was originally published on the Keepsafe Engineering blog](https://medium.com/keepsafe-engineering/the-perils-of-loading-native-libraries-on-android-befa49dce2db)*

Back in 2012, during the early days of [Keepsafe](https://www.getkeepsafe.com/), we sought to implement an encryption scheme for our Android App. Through many iterations and prototypes, we found a sweet spot of sorts by leveraging the power of the [JNI](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/) (Java Native Interface.) We decided to write our interface into the encryption library we utilized in Java, calling into the library via the JNI solely for the purpose of encryption and decryption. We opted for an on-the-fly solution, minimizing the impact on user experience as much as possible. Once we were happy with our solution, we decided to deploy it into our production app. We rigorously tested our code and were confident that everything would go smoothly; that is, until things beyond our control broke.

### Enter the Dreaded “UnsatisfiedLinkError”

As we anxiously refreshed our crash reports following our release, we started to notice a recurring error. Users were running into an `UnsatisfiedLinkError`, which means that either A) the native library we were calling into did not exist or B) the native method we were calling did not exist. Since B) would almost always be caught via compiling and basic testing, we were immediately perplexed at the fact that users’ installations did not have the native libraries we shipped within the APK.

Here are some samples of the exceptions we ran into, they ranged from standard…

    java.lang.UnsatisfiedLinkError: Couldn’t load stlport_shared from loader dalvik.system.PathClassLoader[dexPath=/data/app/com.kii.safe-1.apk,libraryPath=/data/app-lib/com.kii.safe-1]: findLibrary returned null
    at java.lang.Runtime.loadLibrary(Runtime.java:365)
    at java.lang.System.loadLibrary(System.java:535)
    at com.kii.safe.Native.<clinit>(Native.java:16)
    … 63 more

    Caused by: java.lang.UnsatisfiedLinkError: Library stlport_shared not found
    at java.lang.Runtime.loadLibrary(Runtime.java:461)
    at java.lang.System.loadLibrary(System.java:557)
    at com.kii.safe.Native.<clinit>(Native.java:16)
    … 5 more

    Caused by: java.lang.UnsatisfiedLinkError: Cannot load library: get_lib_extents[760]: 1305 — /mnt/asec/com.kii.safe-1/lib/libstlport_shared.so is not a valid ELF object
    at java.lang.Runtime.loadLibrary(Runtime.java:434)
    at java.lang.System.loadLibrary(System.java:554)
    at com.kii.safe.Native.<clinit>(Native.java:15)

    Caused by: java.lang.UnsatisfiedLinkError: Library cryptopp not found
    at java.lang.Runtime.loadLibrary(Runtime.java:461)
    at java.lang.System.loadLibrary(System.java:557)
    at com.kii.safe.Native.<clinit>(Native.java:17)

…to the rather obscene…

    Caused by: java.lang.UnsatisfiedLinkError: Cannot load library: reloc_library[1286]: 1748 cannot locate ‘쯰ҷЦf1Ϙ˗˞ք᣼0Ⱉض夘Ϛ.͏闑㥁ج뭫ර⓻в^ӎ3c`+W#Ҽ?-Bַˌ֕꼠’…
    at java.lang.Runtime.loadLibrary(Runtime.java:370)
    at java.lang.System.loadLibrary(System.java:535)
    at com.kii.safe.Native.<clinit>(Native.java:17)

    Caused by: java.lang.UnsatisfiedLinkError: Cannot load library: reloc_library[1312]: 1327 cannot locate ‘Pܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭXߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭXߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#׶ʭX Ϛߐܝ#
    at java.lang.Runtime.loadLibrary(Runtime.java:434)
    at java.lang.System.loadLibrary(System.java:554)
    at com.kii.safe.Native.<clinit>(Native.java:17)

There was no clear pattern as to *what* library was at fault as it seemed *all* libraries were throwing the exception. It was not specific to an Android version and it did not only occur on specific devices. Moreover, in certain instances some of the native libraries were loaded properly, but not all. At this point, we were furiously searching the internet for answers or help, only to come up empty-handed. We started to release hotfixes, mostly filled with speculative fixes, additional tracking data so we had a better understanding of what exactly the environment was when the crash occured, and a piece of code that specifically checked for the native libraries in their expected install location.

>Moreover, in certain instances some of the native libraries were loaded properly, but not all.

We gathered that, yes, the native libraries were not there, it was not some weird fluke with the System class or file system error, and that user’s devices were seemingly normal.

##### Idea #1 — User Devices are Running out of Space
As we thought through the possible causes for this exception, we started to think that maybe people just had no room for the native library to be installed correctly, and thus it never was installed. A quick diagnostic check before attempting to load the libraries proved this idea wrong *very* quickly. Users had more than enough space on their device for the libraries we were shipping.

##### Idea #2 — Native Libraries were not being included in Updates
The second plausible cause for our problem was that Google Play was mangling our APK when serving it to users’ devices. We had some reinforcement for this idea after reading such reports as [this one](https://code.google.com/p/android/issues/detail?id=35962), detailing that Google Play supposedly sent out a notice to all app developers affected by an issue preventing users from launching their apps after an update due to native libraries being incorrectly installed. The only problem was that this report was in August, and we were dealing with the problem months later. We also never received a notice from Google Play mentioning any sort of error on their part. Additionally, this is something that would be very hard to verify.

##### Idea #3 — Debug the issue directly with a real user
Since we were unable to reproduce the issue on any of the 10+ devices of ranging variety we had on hand, we decided that we would reach out to a user who was experiencing the problem. One user graciously decided to help us and noted that the app was working fine prior to the latest update. The problem here was that the app version the user noted as working was a version that included our encryption code and native libraries, just to add to our confusion and the overarching puzzle. We decided to supply the user with an APK file directly, where we verified that all native libraries were present in the APK file. The user installed the APK, launched the app, and ran directly into the same UnsatisfiedLinkError exception again. This confirmed that Google Play was not the issue, the issue was with Android’s PackageManager installation process. At some point during the installation, something would go wrong and the native libraries inside of the APK would not be extracted.

>[…] the issue was with Android’s PackageManager installation process.

### Getting to a Solution
Since we figured out that the problem was with the installation process, we decided that we would replicate the part of the installation process that extracted the native libraries inside of our App’s code. Luckily you can get a reference to your App’s APK file on the device very easily via
{% highlight java %}
Context.getApplicationInfo().sourceDir;
{% endhighlight %}

which we would then use to extract the native libraries to an internal storage location. Since APK files are just ZIP files, this was a simple matter of writing some ZIP extraction code. We quickly implemented the extraction code and shipped it, resulting in a major drop in the number of crashes!

<img src="/assets/img/relinker-drop.png" alt="" />
<span class="caption">The total number of `UnsatisfiedLinkError` exceptions thrown on a daily basis</span>

Although this was great, we were still seeing the exception popup rather regularly among some of our users. This is where we reached out to Google themselves for help. Thankfully, they led us onto the right path by indicating that users may have installed our app from other sources than the Play Store. Additionally, they pointed us to a nifty function

{% highlight java %}
Context.getPackageManager().getInstallerPackageName(packageName);
{% endhighlight %}

that would tell us what package installed our app.

To reduce our APK size and ensure that our App would run on all possible devices, we have flavors of our App for the x86, Armv7 and Arm architectures. Each flavor only contains the native libraries corresponding to its respective architecture, so it is entirely possible that someone can install an APK that was not made for their device’s architecture.

We started to log the installer package name in our crashes and quickly figured out that, yes, users were installing the app from various sources, and each new `UnsatisfiedLinkError` was coming from a manually installed app where the user mistakenly installed the wrong flavor for their device’s architecture. This was the final “gotcha”, and we were relieved that it had a very simple explanation.

### Introducing ReLinker
We decided to package up the extraction code into a small library that everyone can use. No one should have to go through the debugging process we went through, especially when it involves a very basic function of Android that is not within an App developer’s control.

Using ReLinker is as simple as replacing your standard
{% highlight java %}
System.loadLibrary("mylibrary");
{% endhighlight %}

call with

{% highlight java %}
ReLinker.loadLibrary(context, "mylibrary")
{% endhighlight %}

You can learn more about ReLinker at its [GitHub repository](https://github.com/KeepSafe/ReLinker).

### Finally done
At the point of releasing a fix, the number of people who encountered this crash continuously topped out at about 100,000 unique users. We hope you’ll find ReLinker useful and never run into an `UnsatisfiedLinkerError` randomly again.


#### Acknowledgements
I’d like to thank Google’s support team for pointing us to the right direction and for their continued support with this issue.
Our code is heavily based on the similar workaround published for Chromium. As a bonus, a comment in Chromium’s source code explains what exactly is going wrong in Android’s package manager:
{% highlight java %}
/**
* Try to load a native library using a workaround of
* http://b/13216167.
*
* Workaround for b/13216167 was adapted from code in
* https://googleplex-android-review.git.corp.google.com/#/c/433061
*
* More details about http://b/13216167:
* PackageManager may fail to update shared library.
*
* Native library directory in an updated package is a symbolic link
* to a directory in /data/app-lib/<package name>, for example:
* /data/data/com.android.chrome/lib -> /data/app-lib/com.android.chrome[-1].
* When updating the application, the PackageManager create a new directory,
* e.g., /data/app-lib/com.android.chrome-2, and remove the old symlink and
* recreate one to the new directory. However, on some devices (e.g. Sony Xperia),
* the symlink was updated, but fails to extract new native libraries from
* the new apk.
+
* We make the following changes to alleviate the issue:
* 1) name the native library with apk version code, e.g.,
* libchrome.1750.136.so, 1750.136 is Chrome version number;
* 2) first try to load the library using System.loadLibrary,
* if that failed due to the library file was not found,
* search the named library in a /data/data/com.android.chrome/app_lib
* directory. Because of change 1), each version has a different native
* library name, so avoid mistakenly using the old native library.
*
* If named library is not in /data/data/com.android.chrome/app_lib directory,
* extract native libraries from apk and cache in the directory.
*
* This function doesn’t throw UnsatisfiedLinkError, the caller needs to
* check the return value.
*/
{% endhighlight %}
