# تست‌اپ فلاتر PushPanel

اپ نمونه فلاتر برای اتصال کتابخانه پوش PushPanel در اندروید.

کتابخانه: `ir.push-panel:push-sdk:1.8.3` از MavenCentral

## ۱. افزودن کتابخانه

`android/app/build.gradle.kts` — انتهای فایل:

```kotlin
dependencies {
    implementation("ir.push-panel:push-sdk:1.8.3")
}
```

## ۲. مین‌اکتیویتی

`android/app/src/main/kotlin/com/pushpanel/test/MainActivity.kt`:

```kotlin
import ir.pushpanel.sdk.PushPanel

class MainActivity : FlutterActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        requestNotificationPermission()
        PushPanel.init(this)
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

> در نسخه 1.8.3 نقطه ورود به `PushPanel` تغییر نام داده (قبلاً `PushSdk`) و به `handleIntent` و `onNewIntent` و کد جداگانه برای اکتیویتی اسپلش نیازی نیست؛ فقط `PushPanel.init` کافی است.

## ۳. دسترسی‌ها

`android/app/src/main/AndroidManifest.xml` — بعد از `<manifest>`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

## ۴. فایربیس (برای دریافت واقعی پوش — اجباری)

بدون این مرحله `google-services.json` نادیده گرفته می‌شود، توکن FCM ساخته نمی‌شود و پوشی دریافت نمی‌کنی (بیلد موفق می‌شود ولی خبری از پوش نیست).

۱. فایل `google-services.json` پروژه فایربیس خودت را از کنسول فایربیس بگیر و در `android/app/` بگذار (این فایل عمداً در ریپو نیست چون حاوی کلید خصوصی است). `package_name` داخل آن باید برابر پکیج برنامه (`applicationId`) باشد. بدون این فایل بیلد خطا می‌دهد.

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

## ۵. سرویس فایربیس اختصاصی (اختیاری — فقط اگر کتابخانه پوش دیگری هم داری)

FCM در هر اپ فقط به **یک** `FirebaseMessagingService` پیام تحویل می‌دهد. اگر کتابخانه دیگری هم سرویس خودش را دارد (یا خودت سرویس فایربیس داری)، باید سرویس داخلی SDK را حذف کنی و همه پیام‌ها را از سرویس خودت به هر کتابخانه فوروارد کنی — پیام‌هایی که مال پنل نیستند توسط SDK نادیده گرفته می‌شوند (مارکر `pushpanel=pushpanel`).

۱. در `android/app/src/main/AndroidManifest.xml` سرویس داخلی SDK را حذف کن (`xmlns:tools` را هم به تگ `manifest` اضافه کن):

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <application ...>
        <!-- حذف سرویس داخلی SDK تا فقط سرویس خودمان پیام بگیرد -->
        <service
            android:name="ir.pushpanel.sdk.PushMessagingService"
            tools:node="remove" />
        <!-- سرویس خودمان -->
        <service
            android:name=".MyFirebaseService"
            android:exported="false">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
            </intent-filter>
        </service>
    </application>
</manifest>
```

۲. سرویس خودت همه پیام‌ها را فوروارد کند:

```kotlin
import com.google.firebase.messaging.FirebaseMessagingService
import com.google.firebase.messaging.RemoteMessage
import ir.pushpanel.sdk.PushPanel

class MyFirebaseService : FirebaseMessagingService() {
    override fun onMessageReceived(message: RemoteMessage) {
        super.onMessageReceived(message)
        PushPanel.forwardMessage(this, message) // فقط پیام‌های پنل هندل می‌شود، بقیه نادیده گرفته می‌شود
        // ... هندل پیام‌های خودت / کتابخانه دیگر ...
    }

    override fun onNewToken(token: String) {
        super.onNewToken(token)
        PushPanel.forwardToken(this, token)
    }
}
```

نکته: `PushPanel.init` (بخش ۲) همچنان لازم است؛ فوروارد قبل از init نادیده گرفته می‌شود. اگر سرویس اختصاصی نداری، این بخش را رد کن — سرویس داخلی SDK به‌صورت پیش‌فرض کار می‌کند.

## ۶. بیلد و تست

```powershell
flutter analyze
flutter build apk --debug
```

## ۷. عیب‌یابی (اگر پوش نرسید)

- اپ را کامل uninstall و دوباره نصب کن تا توکن تازه با SDK جدید ثبت شود.
- دسترسی نوتیفیکیشن را بده (اندروید ۱۳+) و روی دیوایس/امولاتور دارای Play Services تست کن.
- لاگ‌کت را با فیلتر `PushSDK` / `FirebaseMessaging` ببین؛ باید ثبت توکن دیده شود.
- مطمئن شو سرور با همان پروژه فایربیس داخل `google-services.json` ارسال می‌کند.
