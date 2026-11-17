# BookMediaPlayer

An Android app for long-form audio. See [design notes](BookMediaPlayerDesignNotes.md)
for the planned features and architecture.

## Project setup

- Install JDK 17 and Android Studio with the Android SDK Platform 35 and build
  tools. The app requires Android 14 (API 34) or newer.
- Open this directory in Android Studio and allow Gradle to sync, or set
  `ANDROID_HOME` to your SDK directory and run `./gradlew assembleDebug`.
- The debug APK is written to `app/build/outputs/apk/debug/`.

The project is an initial scaffold: the library screen is a placeholder, and
importing books, playback controls, and saved positions are not implemented.
The Media3 service is present as a starting point but is not connected to the UI.
