# Extending the UnityPlayerActivity on Android

You can find the resulting project of this guide here on my Github collection [Unity Good Practices](https://github.com/guneyozsan/UnityGoodPractices).

**Note:** This process is specific to the *Android* platform and not applicable to other platforms.

On Android, it is possible to extend `UnityPlayerActivity` to override existing interactions between Unity and Android OS or introduce new behaviors.

From [the Unity documentation](https://docs.unity3d.com/Manual/AndroidUnityPlayerActivity.html):

> When you develop a Unity Android application, you can use plug-ins to extend the standard UnityPlayerActivity class (the primary Java class for the Unity Player on Android, similar to AppController.mm on Unity iOS). An application can override all basic interactions between the Android OS and the Unity Android application.

The suggested method in Unity documentation is to extend *UnityPlayerActivity* is to open the exported Android project in Android Studio, and do modifications from there. However, this method may introduce friction and difficulties in workflow, such as maintaining an isolated repository for the extension plugin and maintaining other modules that you may want to introduce.

Another manual method suggested around the web is to copy `classes.jar` from the Unity directory to your working directory. However, this requires including the classes.jar within the plugin or needs extra effort to exclude it from builds.

On the other hand, with this method, you can have an isolated Android Studio project for your Unity plugin, by referencing the Unity classes without a need to copy and maintain `classes.jar`, and keeping the reference `import com.unity3d.player.UnityPlayerActivity;` intact.

The whole idea is to mock `com.unity3d.player.UnityPlayerActivity;` by including a *compile-only* module for `UnityPlayerActivity`. This way you can work on a fresh isolated Android Studio project as if you are working on an exported Android Studio project.

## Step-by-step guide

You can use [the project provided](https://github.com/guneyozsan/UnityGoodPractices) as a template, or follow this guide to start fresh.

### 1) Create your activity

1. In Android Studio create a new project with *"No activity"*.
    1. Use package name that matches your Unity project `com.mycompany.myunityproject.player` (e.g. `com.awesomegamestudio.veryfungame.player`).
2. In `AndroidManifest.xml`:
    1. Delete all lines referencing themes and icons:
       `android:icon=...`
       `android:roundIcon=...`
       `android:theme=...`
3. *(Optional)* Delete folders `app/src/androidTest` and `app/src/test`.
4. Delete everything under the folder `res` except `res/values/strings.xml`.
5. In `build.gradle` of the module:
    1. Replace `id 'com.android.application'` with `id 'com.android.library'`.
    2. Delete line `applicationId "com.mycompany.myunityproject"`.
    3. Delete all lines related to tests.
    4. Delete all dependencies.
    5. Add local Unity installation as a *compile-only* dependency (so that it is not included in the builds).

       ```js
       dependencies {
           compileOnly files('C:/Program Files/Unity/Hub/Editor/2019.4.33f1/Editor/Data/PlaybackEngines/AndroidPlayer/Variations/mono/Release/Classes/classes.jar)
       }
       ```

       *NOTE: Unity version in URLs should match yours.*

       *NOTE: You only need to update the Unity version number when you want to modify the extension plugin. Builds are independent of this dependency and will be using the reference from the Unity version that they were imported.*

### 2) Mock Unity activity

1. From `File/New/New Module`, create a new **module** with package name `com.unity3d.player`.
2. Perform the same steps above (Create your activity) for this module as well (clean-up and adding local Unity dependency).
3.
    1. Create file `UnityPlayerActivity.java` in `/player/java/com/unity3d/player/`. Use simple mock content or copy it from Unity directory.
        1. Paste the simplified mock content into `UnityPlayerActivity.java` you created:

           ```c#
           package com.unity3d.player;
           
           import android.app.Activity;
           
           public class UnityPlayerActivity extends Activity implements IUnityPlayerLifecycleEvents
           {
               // When Unity player unloaded move task to background
               @Override public void onUnityPlayerUnloaded() {
               }
  
               // When Unity player quited kill process
               @Override public void onUnityPlayerQuitted() {
               }
           }
           ```

        2. Or, in order to access original functionality, copy `UnityPlayerActivity.java` from `C:\Program Files\Unity\Hub\Editor\2019..4.33f1\Editor\Data\PlaybackEngines\AndroidPlayer\src\com\unity3d\player\` to `/player/java/com/unity3d/player/`.
4. In `build.gradle` of module `MyUnityPlayerActivity` add a compile-only dependency to the module `player` we just created.

   ```js
   dependencies {
       compileOnly files('C:/Program Files/Unity/Hub/Editor/2019.4.33f1/Editor/Data/PlaybackEngines/AndroidPlayer/Variations/mono/Release/Classes/classes.jar)
       compileOnly project(':player')
   }
   ```

    * NOTE: Unity version in URLs should match yours.*

### 3) Extend Unity activity

1. Create new class `MyUnityPlayerActivity` in `app/java/com.mycompany.myunityproject.player` with this content:

   ```c#
   package com.mycompany.myunityproject.player;
   
   import android.os.Bundle;
   import android.util.Log;
   
   import com.unity3d.player.UnityPlayerActivity;

   public class MyUnityPlayerActivity extends UnityPlayerActivity {
      private static final String TAG = "Unity";

      // Overriding `onCreate()` is only for demo purposes, and not required for extending the Unity activity.
      @Override
      protected void onCreate(Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
         Log.d(TAG, "Running MyUnityPlayerActivity.");
      }
   }
   ```

   *NOTE: Overriding `onCreate()` is only for demo purposes, and not required for extending the Unity activity.*

### 4) Export plugin from Android Studio

1. Select build variant `debug` or `release` from `Build/Select Build Variant...`. *(Use build variant `debug` to see the logs in logcat.)*
2. In Android Studio `Build/Make` will build `.aar` to `/app/build/outputs/aar/app-release.aar`.

### 5) Import plugin to Unity
1. Copy the `.aar` file built in previous step to Unity project into the folder `Assets/Plugins/Android/libs/MyUnityPlayerActivity/`.
2. Customize Unity AndroidManifest *(need to be done only once)*:
    1. Enable `Custom Main Manifest` at `Project Settings/Player/Publishing Settings/Build`.
    2. Open `Assets/Plugins/Android/AndroidManifest.xml` for editing. Edit activty attribute
       from `<activity android:name="com.unity3d.player.UnityPlayerActivity" ...`
       to `<activity android:name="com.mycompany.myunityproject.player.MyUnityPlayerActivity" ... >`.
3. If you introduced any *non-compile-only* dependencies in `build.gradle (app)` of MyUnityPlayerActivity:
    1. Create a file `/Assets/MyUnityPlayerActivity/Editor/MyUnityPlayerActivityDependencies.xml`.
    2. Declare the dependencies in this file using the format below.
    3. Use [External Dependency Manager for Unity](https://github.com/googlesamples/unity-jar-resolver) to resolve dependencies.

       Example `MyUnityPlayerActivityDependencies.xml`:
       ```xml
       <dependencies>
           <androidPackages>
               <androidPackage spec="androidx.appcompat:appcompat:1.3.1"/>
               <androidPackage spec="com.google.android.gms:play-services-auth:19.2.0"/>
               ...
           </androidPackages>
       </dependencies>
       ```

### 6) Run on device

1. In Unity, make sure *Android* platform is selected in `Build Settings`.
2. Use `Build Settings/Build And Run` to run the application on a connected Android device.
3. If you used build variant `debug` while building the plugin in Android Studio, you should see the demo log `Running MyUnityPlayerActivity.` from overridden `onCreate()` in `adb logcat` (e.g. by running `adb logcat -s Unity` in command prompt, or by using Android Logcat package in Unity).

### Further improvements

- Exporting Android Studio build and copying the built plugin into the Unity project can be automated for even a smoother workflow.
