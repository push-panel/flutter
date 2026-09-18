# تست‌اپ فلاتر PushPanel

اپ نمونه فلاتر برای اتصال کتابخانه پوش PushPanel در اندروید.

کتابخانه: `ir.push-panel:push-sdk:1.7.2` از MavenCentral

## ۱. افزودن کتابخانه

`android/app/build.gradle.kts` — انتهای فایل:

```kotlin
dependencies {
    implementation("ir.push-panel:push-sdk:1.7.2")
}
```

## ۲. مین‌اکتیویتی

`android/app/src/main/kotlin/com/pushpanel/test/MainActivity.kt`:

```kotlin
import ir.pushpanel.sdk.PushSdk

class MainActivity : FlutterActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        requestNotificationPermission()
        PushSdk.init(this)
    }

    private fun requestNotificationPermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            if (checkSelfPermission(Manifest.permission.POST_NOTIFICATIONS) != PackageManager.PERMISSION_GRANTED) {
                requestPermissions(arrayOf(Manifest.permission.POST_NOTIFICATIONS), 100)
            }
        }
    }
}
```

> در نسخه 1.7.2 به `PushSdk.handleIntent` و `onNewIntent` و کد جداگانه برای اکتیویتی اسپلش نیازی نیست؛ فقط `PushSdk.init` کافی است.

## ۳. دسترسی‌ها

`android/app/src/main/AndroidManifest.xml` — بعد از `<manifest>`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

## ۴. فایربیس (برای دریافت واقعی پوش — اجباری)

بدون این مرحله `google-services.json` نادیده گرفته می‌شود، توکن FCM ساخته نمی‌شود و پوشی دریافت نمی‌کنی (بیلد موفق می‌شود ولی خبری از پوش نیست).

۱. فایل `google-services.json` را در `android/app/` بگذار؛ `package_name` داخل آن باید برابر پکیج برنامه (`applicationId`) باشد.

۲. در `android/settings.gradle.kts` داخل بلاک `plugins` این خط را اضافه کن:

```kotlin
plugins {
    id("dev.flutter.flutter-plugin-loader") version "1.0.0"
    id("com.android.application") version "9.0.1" apply false
    id("org.jetbrains.kotlin.android") version "2.3.20" apply false
    id("com.google.gms.google-services") version "4.4.2" apply false
}
```

۳. در `android/app/build.gradle.kts` داخل بلاک `plugins` این خط را اضافه کن:

```kotlin
plugins {
    id("com.android.application")
    id("com.google.gms.google-services")
    // The Flutter Gradle Plugin must be applied after the Android and Kotlin Gradle plugins.
    id("dev.flutter.flutter-gradle-plugin")
}
```

۴. برای اطمینان بعد از بیلد، این فایل باید تولید شده باشد و شامل `google_app_id` باشد:

```
build/app/generated/res/processDebugGoogleServices/values/values.xml
```

## ۵. بیلد و تست

```powershell
flutter analyze
flutter build apk --debug
```

## ۶. عیب‌یابی (اگر پوش نرسید)

- اپ را کامل uninstall و دوباره نصب کن تا توکن تازه با SDK جدید ثبت شود.
- دسترسی نوتیفیکیشن را بده (اندروید ۱۳+) و روی دیوایس/امولاتور دارای Play Services تست کن.
- لاگ‌کت را با فیلتر `PushSDK` / `FirebaseMessaging` ببین؛ باید ثبت توکن دیده شود.
- مطمئن شو سرور با همان پروژه فایربیس داخل `google-services.json` ارسال می‌کند.
