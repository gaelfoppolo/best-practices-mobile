# Mobile-specific Best Practices

This guide is aimed at developers for mobile platforms who want to make their native apps more sustainable, that is, both environmentally and socially acceptable. Unlike general rules of thumb, this guide is focused on **code smells**, that is surface symptoms that suggest there might be a
problem, or that there might be a better way of writing the code. Therefore, these low-level best practices offer the great advantage of being detectable by program analysis tools, such as [ecoCode Mobile](https://github.com/green-code-initiative/ecoCode-mobile) (formerly hosted by Cnumr).

# Android Platform

## 🌿 Environmental Code Smells
 Name | Detailed Description
---|---
 *Optimized API* |  
 Fused Location | The fused location provider is one of the location APIs in Google Play services which combines signals from GPS, Wi-Fi, and cell networks, as well as accelerometer, gyroscope, magnetometer and other sensors. It is officially recommended to maximize battery life. Thus, developer has to set up Google Play Service in her gradle file with a dependency to `com.google.android.gms:play-services-location:x.y.z`, and then to import from `com.google.android.gms.location` instead of the `android.location` package of the SDK. 
 Bluetooth Low-Energy | In contrast to classic Bluetooth, Bluetooth Low Energy (BLE) is designed to provide significantly lower power consumption. Its purpose is to save energy on both paired devices but very few developers are aware of this alternative API. From the Android client side, it means append `android.bluetooth.le.*` imports to `android.bluetooth.*` imports in order to benefits from low-energy features. 
 *Leakage* |  
 Media Leak | Creation of a Media Recorder object with `new MediaRecorder()` is used to record audio and video, while creation of a Media Player object with `new MediaPlayer()` can be used to control playback of audio/video files and streams. Both classes own a `release()` method. In addition to unnecessary resources (such as memory and instances of codecs) being held, failure to call this method immediately if a media object is no longer needed may also lead to continuous battery consumption for mobile devices. 
 Sensor Leak | Most Android-powered devices have built-in sensors that measure motion, orientation, and various environmental conditions. In addition to these are the image sensor (a.k.a Camera) and the geopositioning sensor (a.k.a GPS). The common point of all these sensors is that they are expensive while in use. Their common bug is to let the sensor unnecessarily process data when the app enters an idle state, typically when paused or stopped. Consequently, calls must be carefully pairwised: `SensorManager#registerListener()/unregisterListener()` for regular sensors, `Camera#open()/Camera#release()` for the camera and `LocationManager#requestLocationUpdates()/removeUpdates()` for the GPS. Failing to do so can drain the battery in just a few hours. 
 Everlasting Service | If someone calls `Context#startService()` then the system will retrieve the service (creating it and calling its `onCreate()` method if needed) and then call its `onStartCommand(Intent, int, int)` method with the arguments supplied by the client. The service will at this point continue running until `Context#stopService()` or `Service#stopSelf()` is called. Failing to call any of these methods can lead to uncontrolled energy leakage. 
 *Bottleneck* |  
 Internet In The Loop | Opening and closing internet connection continuously is extremely battery-inefficient since HTTP exchange is the most consuming operation of the network. This bug typically occurs when one obtain a new `HttpURLConnection` by calling `URL#openConnection()` within a loop control structure (while, for, do-while, for-each). Also, this bad practice must be early prevented because it is the root of another evil that consists in polling data at regular intervals, instead of using push notifications to save a lot of battery power. 
 Wifi Multicast Lock | Normally the Wifi stack filters out packets not explicitly addressed to the device. Acquiring a Multicast Lock with `WifiManager.MulticastLock#acquire()` will cause the stack to receive packets addressed to multicast addresses. Processing these extra packets can cause a noticeable battery drain and must be disabled when not needed with to a call to `WifiManager.MulticastLock#release()`. 
 Uncompressed Data Transmission | Transmitting a file over a network infrastructure without compressing it consumes more energy than with compression. More precisely, energy efficiency is improved in case the data is compressed at least by 10%, transmitted and decompressed at the other network node. From the Android client side, it means making a post HTTP request using a `GZIPOutputStream` instead of the classical `OutputStream`, along with the `HttpURLConnection` object. 
 Uncached Data Reception | Caching all of the application's HTTP responses to the filesystem (application-specific cache directory) so they may be reused, allow to save energy. To that purpose the class `HttpResponseCache` supports `HttpURLConnection` and `HttpsURLConnection`; there is no platform-provided cache for further clients. Installing the cache at application startup is done with the static method `HttpResponseCache.install(File directory, long maxSize)`. 
 *Sobriety* |  
 Dark UI | Developers are allowed to apply native themes for their app, or derive new ones throught inheritence. This decision has a significant impact on energy consumption since displaying dark colors is particularly beneficial for mobile devices with (AM)OLED screens. By default Android will set Holo to the Dark theme (parent style `Theme.Holo`) and hence switching to the light theme (parent style `Theme.Holo.Light`) within the manifest should be avoided. Of course, custom resources like bright colors values and bitmap images with too high luminance should be avoided too. 
 Thrifty Geolocation | Location awareness is one of the most popular features used by apps. The first thing is to try to get the best possible provider (`LocationManager#getBestProvider()`) based on an energy criteria thanks to `Criteria#setPowerRequirement(int level)`, using constant `POWER_LOW` instead of `POWER_HIGH` or `POWER_MEDIUM`. Next, with a call to `LocationManager#requestLocationUpdates (long minTime, float minDistance, Criteria criteria, PendingIntent intent)`, the provider will only send your application an update when the location has changed by at least `minDistance` meters, AND at least `minTime` milliseconds have passed. So `minTime` should be the primary tool to conserving battery life, and, to a lesser extent, `minDistance`, these two must be imperatively greater than 0. 
 Thrifty BLE | With Bluetooth Low Energy technology (see BLE API smell), a Bluetooth Smart Ready device (the master) will establish a link with a Bluetooth Smart device (the slave). Most often, the slave is a GATT server and the master is a GATT client. GATT capable devices can be discovered using BLE scan process. So you should always use `BluetoothLeScanner#startScan()` with the scan settings set to `SCAN_MODE_LOW_POWER` (Default scan mode). In addition, invoke `BluetoothGatt#requestConnectionPriority(int connectionPriority)` with the value `CONNECTION_PRIORITY_LOW_POWER`. Lastly, the default and preferred advertising mode is `ADVERTISE_MODE_LOW_POWER` when calling `AdvertiseSettings.Builder#setAdvertiseMode(int advertiseMode)`. 
 Thrifty Motion Sensor | The rotation vector sensor is the most frequently used sensor for motion detection and monitoring. When using `SensorManager#getDefaultSensor(int type)`, always prefer the constant `TYPE_GEOMAGNETIC_ROTATION_VECTOR` which is similar to `TYPE_ROTATION_VECTOR`, but using a magnetometer instead of using a gyroscope. This sensor uses lower power than the other rotation vectors, because it doesn't use the gyroscope. However, it is more noisy and will work best outdoors. 
 Thrifty Notification | Giving informations to the end-user through notifications is an important aspect of a modern app. However, a notification does not necessarily need to be loud and vibrant to achieve its purpose. Default mode is enough. That is why when building a notification with `NotificationCompat.Builder`, there should be no extra calls to the methods `setVibrate()` nor `setSound()` on the builder object. Please consider `NotificationChannel#setVibrationPattern()` since API 31. 
 Vibration-free | Shaking of an Android device is possible in all circumstances with a call to `getSystemService(Context.VIBRATOR_SERVICE)` (API 26). Behind this effect stands a specific miniature hardware component, motor or actuator, that consumes power. As a consequence, its usage must be discouraged, especially since its added value is not clear. Please consider `Context.VIBRATOR_MANAGER_SERVICE` since API 31. 
 Torch-free | Turning on the torch mode programmatically with `CameraManager#setTorchMode(..., true)` must absolutely be avoided because the flashlight is one of the most energy-intensive component. 
 High Frame Rate | In Android 11 (API level 30) or higher, a call to `Surface#setFrameRate(float frameRate, int compatibility)` results in a change to the display refresh rate. However, a regular app displays 60 frames per second (60Hz). In order to optimize content refreshes and hence saving energy, this frequency should not be raised to 90Hz or 120Hz, despite this is now supported by many devices. This bad practice can be detected when the value of the `frameRate` argument is greater than 60f. 
 Animation-free | Avoiding extraneous animations in the UI is a good practice for saving battery power. This can be checked either in the Java code when an object is instance of `Animator` (sub)class, or simply through the presence of xml files in the `res/animator/` resource directory. 
 *Idleness* |  
 Keep Screen On | To avoid draining the battery, an Android device that is left idle quickly falls asleep. Hence, keeping the CPU on should be avoided, unless it is absolutely necessary. If so, developers typically use the `FLAG_KEEP_SCREEN_ON` in their activity. Another way to implement this is in their application's layout XML file, by using the `android:keepScreenOn` attribute. 
 Keep CPU On | To avoid draining the battery, an Android device that is left idle quickly falls asleep. Hence, keeping the screen on should be avoided, unless it is absolutely necessary. If so, developers typically use a Power Manager system service feature called wake locks by invoking `PowerManager.WakeLock#newWakeLock(int levelAndFlags, String tag)`, along with the specific permission `WAKE_LOCK` in their manifest. 
 Durable Wake Lock | A wake lock is a mechanism to indicate that your application needs to have the device stay on. The general principle is to obtain a wake lock, acquire it and finally release it. Hence, the challenge here is to release the lock as soon as possible to avoid running down the device's battery excessively. Missing call to `PowerManager#release()` is a built-in check of Android lint (Wakelock check) but that does not prevent abuse of the lock over too long a period of time. This can be avoided by a call to `PowerManager.WakeLock#acquire(long timeout)` instead of `PowerManager.WakeLock#acquire()`, because the lock will be released for sure after the given timeout expires. 
 Rigid Alarm | Applications are strongly discouraged from using exact alarms unnecessarily as they reduce the OS's ability to minimize battery use (i.e. Doze Mode). For most apps prior to API 19, `setInexactRepeating()` is preferable over `setRepeating()`. When you use this method, Android synchronizes multiple inexact repeating alarms and fires them at the same time, thus reducing the battery drain. Similarly, `setExact()` and `setExactAndAllowWhileIdle()` can significantly impact the power use of the device when idle, so they should be used with care. High-frequency alarms are also bad for battery life but this is already checked by Android lint (ShortAlarm built-in check). 
 Continuous Rendering | For developers wishing to display OpenGL rendering, when choosing the rendering mode with `GLSurfaceView#setRenderMode(int renderMode)`, using `RENDERMODE_WHEN_DIRTY` instead of `RENDERMODE_CONTINUOUSLY` (By default) can improve battery life and overall system performance by allowing the GPU and CPU to idle when the view does not need to be updated. 
 Keep Voice Awake | During a voice interaction session, `VoiceInteractionSession#setKeepAwake(boolean keepAwake)` allows to decide whether it will keep the device awake while it is running a voice activity. By default, the system holds a wake lock for it while in this state, so that it can work even if the screen is off. Setting this to `false` removes that wake lock, allowing the CPU to go to sleep and hence does not let this continue to drain the battery. 
 *Power* |  
 Ignore Battery Optimizations | An app holding the `Manifest.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` ask the user to allow it to ignore battery optimizations (that is, put them on the whitelist of apps). Most applications should not use this; there are many facilities provided by the platform for applications to operate correctly in the various power saving modes. 
 Companion in background | A negative effect on the device's battery is when an app is paired with a companion device (over Bluetooth, BLE, or Wi-Fi) and that it has been excluded from battery optimizations (run in the background) using the declaration `Manifest.permission.REQUEST_COMPANION_RUN_IN_BACKGROUND`. 
 Charge Awareness | It's always good that an app has different behavior when device is connected/disconnected to a power station, or has different battery levels. One can monitor the changes in charging state with a broadcast receiver registered on the actions `ACTION_POWER_CONNECTED` and `ACTION_POWER_DISCONNECTED`, or monitor significant changes in battery level with a broadcast receiver registered on the actions `BATTERY_LOW` and `BATTERY_OKAY`. 
 Save Mode Awareness | Taking into account when the device is entering or exiting the power save mode is higly desirable for the battery life. It implies the existence of a broadcast receiver registered on the action `ACTION_POWER_SAVE_MODE_CHANGED`, or programmaticaly with a call to `PowerManager#isPowerSaveMode()`. 
 Constrained Worker | You can use the `WorkManager` library to perform work on an efficient schedule that considers whether specific conditions are met, such as power status. A worker can be scheduled to run, provided the device's battery isn't low for exemple. This can be checked by a call to `WorkRequest.Builder#setConstraints()` where the constraint object built involves `Constraints.Builder#setRequiresBatteryNotLow(true)` or, to a lesser extent, `Constraints.Builder#setRequiresCharging(true)`. 
 *Batch* |  
 Service@Boot-time | Services are long-living operations, as components of the apps. However, they can be started in isolation each time the device is next started, without the user's acknowledgement. This technique should be discouraged because the accumulation of these silent services results in excessive battery depletion that remains unexplained from the end-user's point of view. In addition, end-users know how to kill applications, but more rarely how to kill services. Thus, any developer should avoid having a call to `Context#startService()` from a Broadcast Receiver component that has specified an intent-filter for the `BOOT_COMPLETED` action in the manifest. 
 Sensor Coalesce | With `SensorManager#registerListener(SensorEventListener, Sensor, int)` the events are delivered as soon as possible. Instead, `SensorManager#registerListener(SensorEventListener, Sensor, int, int maxReportLatencyUs)` allows events to stay temporarily in the hardware FIFO (queue) before being delivered. The events can be stored in the hardware FIFO up to `maxReportLatencyUs` microseconds. Once one of the events in the FIFO needs to be reported, all of the events in the FIFO are reported sequentially. Setting `maxReportLatencyUs` to a positive value allows to reduce the number of interrupts the AP (Application Processor) receives, hence reducing power consumption, as the AP can switch to a lower power state while the sensor is capturing the data. 
 Job Coalesce | The Android 5.0 Lollipop (API 21) release introduces a job scheduler API via the `JobScheduler` class. Compared to a custom Sync Adapter or the alarm manager, the Job Scheduler supports batch scheduling of jobs. The Android system can combine jobs so that battery consumption is reduced. It means that at least one job scheduler exists through a call to the method `JobScheduler#shedule(JobInfo job)`. 
 *Release* |  
 Supported Version Range | When looking at the `minSdkVersion` and `targetSdkVersion` attributes for the `<uses-sdk>` in the `AndroidManifest.xml` file, the amplitude of supported platform versions should not be too wide, at the risk of making the app too heavy to handle all cases. It is worth notice that this smell may contradicts with the Aging devices social smell. Pleace notice that in newer versions of Android, corresponding xml properties was replaced by Gradle properties in `build.gradle` file. 
 Same dependencies | This occurs when equivalent libraries are incorporated into the app. If they do quite the same things, choose only one. This can be checked in the dependencies section of `build.gradle` along with a human-curated list of equivalent libraries for Android (e.g., Volley&asymp;Retrofit&asymp;okHttp). 
 Duplicate dependencies | This occurs when several versions of the same library are incorporated into the app. The weight of the app is fatally increased without changing the features. This can be checked in the dependencies section of `build.gradle`. 
 Fat app | When an app exceeds the limit of 65 536 method references, the configuration multidex must be enabled with `multiDexEnabled true` in the `defaultConfig` section of `build.gradle`. Switching to that configuration of multiple dex files  goes against the overall reduction of the weight of the apps and hence must be avoided.		 
 Convert to WebP | WebP is an image format developed by Google, presenting smaller yet more visually-pleasing pictures. Typically, WebP compresses images by an average of 30 percent more than JPEG with no loss in quality. From Android 4.0 (API level 14) and higher, using this image format instead of .gif, .png or .jpeg in the `/res` folder is a good practice since it allows to decrease the size of apks. Notice that Android Studio allows you to easily convert images to the WebP format. 
 Clear cache | A good (but quite rare) practice in the long term is to delete the whole cache directory of the app onto the target device. This requires the source code of the app to contains at least one call to the method `Context.cacheDir#deleteRecursively()`. 
 Shrink Resources | For those that are still not publishing with the Android App Bundle format, it is possible to minimize the app's size via the `build.gradle` file, by using the following lines. Especially useful for popular apps, it reduces the amount of downloaded data required for installation and updating, across millions of devices. Check `buildTypes { release { shrinkResources true } }` 
 Disable Obfuscation | The Proguard tool secure the app for production, including shrinking, code optimization and obfuscation. However, obfuscated code will have a sligthly negative impact on power consumption at runtime. To disable it, in `build.gradle`, replace the setting `minifyEnabled true` by this custom rule: `postprocessing { obfuscate false }` 

## 🤝 Social Code Smells
 Name | Detailed Description
---|---
 *Privacy*
 Crashlytics automatic opt-in | By default, Crashlytics automatically collects error reports for all application users. In order to let the choice to the users, disable the automatic data collection. It means adding a meta-data in the manifest with the attributes `android:name="firebase_crashlytics_collection_enabled"` and `android:value="false"`. From the source code, you may have the call `FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(false)`. 
 Google Tracker | Importing the `com.google.android.gms.analytics.Tracker` class means that the app sends hits to Google Analytics. It is not necessarily sensitive information, but it is a first step towards Google Ads and hence this practice should be discouraged at early stage. 
  Hidden Tracker Risk | An empirical evidence is that the more a project imports third party libraries, the more likely it is that there are hidden trackers. This requires to check the number of non-official dependencies throughtout the directives `implementation` or `api` in the `build.gradle` file. A non-official dependency refers to a package beyond the scope of `android.*`, `androidx.*` and to a lesser extent, `com.android.*` and `com.google.*`. 
 Tracking Id | For some use cases, it might be necessary to get a unique device identifier by a call to `TelephonyManager#getDeviceId()` (returns IMEI on GSM, MEID for CDMA). However, this raises privacy concerns and it is not recommended. Alternatively, you may use `android.provider.Settings.Secure.ANDROID_ID`.
 Explain Permission | Users are increasingly suspicious about the permissions requested by an app. A good practice is to test if `ActivityCompat#shouldShowRequestPermissionRationale()` returns `True`, so you can show an educational UI to the user. In this UI, describe why the feature, which the user wants to enable, needs a particular permission.
*GDPR*
 Google consent | To support publishers in meeting their duties under the EU User Consent Policy, Google offers a Consent SDK. Hence, importing classes from `com.google.android.ads.consent` is considered as a good practice.
*Inclusion*
 Aging devices | The `minSdkVersion` set in the `build.gradle` file determines which APIs are available at build time, and determines the minimum version of the OS that the code will be compatible with. The lower the better so as not to exclude owners of older devices. 

# iOS Platform

🚧 Under Construction...

# Licence
This guide is part of the work of [Dr. Olivier Le Goaër](https://olegoaer.perso.univ-pau.fr/) and protected by [CC BY-NC-ND 4.0](LICENSE.md)
