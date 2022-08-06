# AboutAndroid12
A Public Repo For Android- 12,Updates &amp; Rules, Migration of Android apps to Android 12.

This repo is a comprehensive guide to enable Android developers to upgrade their apps codebase in order to support Android 12 devices. In this we see the steps and challenges involve to update the app.

#### Step 1: Update Target SDK Version and Compile SDK Version to 31

compileSdkVersion 31
...
targetSdkVersion 31

#### Step 2: Safer component exporting

If you try to build your app after doing step 1, you will come across the following error,
**“Apps targeting Android 12 and higher are required to specify an explicit value for ‘android:exported’ when the corresponding component has an intent filter defined.”**

Here starts your actual journey with **Android 12 :)** Now to avoid this follow these common applied **rules**, although you will get other similar solutions on internet as well.

## Rule 1-
— If your app targets Android 12 or higher and contains activities, services, or broadcast receivers that use intent filters, you must explicitly declare the android:exported attribute for these app components.
Give a chance to read more [here](https://developer.android.com/guide/topics/manifest/activity-element#exported) as well.

## Rule 2—
If the app component includes the LAUNCHER category, set android:exported = “true”. In mostly other cases, set “android:exported” as false.

In a nutshell, we need to set exported as true, if any components need to be accessible from outside of our app (either through the OS or other apps), else they need to be set as false(when only used by our app)

```xml
<activity
    android:name=".app.main.view.LauncherActivity"
    android:configChanges="orientation|keyboardHidden|screenSize"
    android:label="@string/app_name"
    android:launchMode="singleTop"
    ...
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    ...
</activity>
```
This is a major security improvement introduced in Android 12.After completing this step, you should be able to install your app on Android 12 devices/emulators. Else their must some other problem in your code regarding installations or any build issues, we will discuss more in upcoming steps.

#### Step 3: Pending Intents Mutability

After completing step 2, you would see your app getting installed successfully, however, it would not launch on android 12 devices. You would come across the following crash while launching your app on Android 12 devices,

```
Caused by: java.lang.IllegalArgumentException: <application id>: Targeting S+ (version 31 and above) requires that one of FLAG_IMMUTABLE or FLAG_MUTABLE be specified when creating a PendingIntent.
    Strongly consider using FLAG_IMMUTABLE, only use FLAG_MUTABLE if some functionality depends on the PendingIntent being mutable, e.g. if it needs to be used with inline replies or bubbles.
        at android.app.PendingIntent.checkFlags(PendingIntent.java:375)
        at android.app.PendingIntent.getActivityAsUser(PendingIntent.java:458)
        at android.app.PendingIntent.getActivity(PendingIntent.java:444)
        at android.app.PendingIntent.getActivity(PendingIntent.java:408)
...

```
The above means that if your app targets Android 12, you must specify the mutability of each PendingIntent object that your app creates. This requirement also improves your app's security.

```java
var pendingIntent: PendingIntent
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    pendingIntent = PendingIntent.getActivity(
        context, 0, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )
} else {
    pendingIntent = PendingIntent.getActivity(context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT)
}

```
It is recommended to use FLAG_IMMUTABLE. However, certain use cases specified [here](https://developer.android.com/guide/components/intents-filters#CreateImmutablePendingIntents) might require you to use mutable Pending Intent, so check if your app falls in any of these.
After completing this step, you should be able to see your app getting launched on Android 12 devices successfully.


#### Step 4: Exact Alarm Permission

To encourage apps to conserve system resources, apps that target Android 12 and higher and set exact alarms must have access to the “Alarms & reminders” capability. This capability appears in the Special app access screen in system settings and to obtain this access, apps need to mention the following permission in manifest,

<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM"/>
Exact alarms should be used for certain customer-facing features. Check the acceptable use cases for setting up an exact alarm here.

#### Step 5: Approximate Location

On apps targeting Android 12, users can request that your app retrieve only approximate location information, even when your app requests the ACCESS_FINE_LOCATION permission.

To handle this potential user behaviour, don’t request the ACCESS_FINE_LOCATION permission by itself. Instead, request both the ACCESS_FINE_LOCATION permission and the ACCESS_COARSE_LOCATION permission in a single runtime request.

When your app requests both ACCESS_FINE_LOCATION and ACCESS_COARSE_LOCATION, the system permissions dialog includes the following options for the user:

Precise: This allows your app to get precise location information.
Approximate: This allows your app to get only approximate location information.

#### Step 6: Foreground Service Launch Restrictions

Apps that target Android 12 or higher aren’t allowed to start foreground services while they are running in the background, barring few exceptional situations. If your app attempts to start a foreground service while running in the background, an exception would occur (except for the few cases).

Consider using WorkManager to schedule and start work while your app runs in the background.

#### Step 7: Unsafe intent launches

While targeting Android 12, we need to be sure that we don’t use unsafe intent launches.

Few help to be followed here are as follows,

Help 1 : Check for overuse of putExtras(Intent) or putExtras(Bundle) calls. Also, watch out for bundles that are too large. The upper bound of the possible buffer storage is only 1MB.

Never try to pass a whole Bitmap for example to an Intent. In that case, you will come across a TransactionTooLargeException.

Help 2 : While triggering internal Intent launches, make sure “exported” flag of the respective components is set as false.

Help 3 : Avoid using nested Intents. This means we should understand passing an Intent to another Intent as an extra, which on the other hand calls startActivity(). Use PendingIntents instead of using nested intents.

To check for unsafe intent launches in your app, call detectUnsafeIntentLaunch() when you configure your VmPolicy in the onCreate() of your Application class.

```java
StrictMode.VmPolicy.Builder vmPolicyBuilder =
    new StrictMode.VmPolicy.Builder()
...
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
  vmPolicyBuilder.detectUnsafeIntentLaunch();
}
StrictMode.setVmPolicy(vmPolicyBuilder.build());
```
#### Step 8: Web Intent Resolution

Starting in Android 12 (API level 31), a web intent resolves to an activity in your app only if your app is approved for the specific domain contained in that web intent. If your app isn’t approved for the domain, it resolves to the user’s default browser app instead.

Apps can get this approval by verifying the domain using Android App Links. If your app supports deep links, then to convert them to App links and make your app default handler for all URLs, perform the following steps,

1- Add the “autoVerify” attribute in the intent filter

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />

    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />

    <data android:scheme="https" />
    <data android:scheme="http" />
    <!-- If your website is www.example.com, then host will be example.com-->
    <data android:host="your site host name" />
  
    <data android:path="/" />
 
    <data android:pathPrefix="/product/" />
    ...
</intent-filter>
```
2- Associate your android app with your website : Create a Digital Asset Links file as demonstrated [here](https://developer.android.com/studio/write/app-link-indexing#associatesite) and upload to your site, with read-access for everyone, at https://<yoursite>/.well-known/assetlinks.json .

Post completion of the above, whenever the user clicks on an Android App Link, your app opens immediately if it’s installed on the device, the disambiguation dialog will not appear.

#### Step 9: New Android System Splash Screen

Starting in Android 12, the system always applies the new default splash screen on cold and warm starts for all apps. By default, this system default splash screen is constructed using your app’s launcher icon element and the windowBackground of your theme (if it's a single colour).

![image](https://user-images.githubusercontent.com/10571832/183246026-ba4a66e0-59e3-4153-9f2a-738f3ac152db.png)

The system splash screen consists of your app icon in the centre surrounded by iconBackground within the window having windowBackground colour.

Completing the migration of our android app to Android 12 and launching it in production has been an exciting and a great learning experience. At the end of blog I would encourage the readers to get started towards upgrading their apps to android 12 and keep in mind the Google Play deadline for the same stated [here](https://support.google.com/googleplay/android-developer/answer/11926878?hl=en-IN).

Thanks for reading! Have a good day.

#### References:
* [Android Developers Site](https://developer.android.com/about/versions/12)
* Stakeoverflow :)
* Medium
* Github Sources   



