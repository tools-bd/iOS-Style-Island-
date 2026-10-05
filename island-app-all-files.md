## .github/workflows/build.yml

```yaml
name: Build APK
on:
  push:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: 8.7
      - run: gradle assembleDebug
      - uses: actions/upload-artifact@v4
        with:
          name: island-apk
          path: app/build/outputs/apk/debug/app-debug.apk
```

## app/build.gradle.kts

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.island.app"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.island.app"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = "17"
    }
}
```

## app/src/main/AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />

    <application
        android:label="Island"
        android:theme="@android:style/Theme.DeviceDefault.Light">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".IslandService"
            android:exported="true"
            android:label="Island"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>
            <meta-data
                android:name="android.accessibilityservice"
                android:resource="@xml/accessibility_config" />
        </service>
    </application>
</manifest>
```

## app/src/main/java/com/island/app/IslandService.kt

```kotlin
package com.island.app

import android.accessibilityservice.AccessibilityService
import android.content.Context
import android.content.SharedPreferences
import android.graphics.Color
import android.graphics.PixelFormat
import android.graphics.drawable.GradientDrawable
import android.os.Handler
import android.os.Looper
import android.view.Gravity
import android.view.MotionEvent
import android.view.ScaleGestureDetector
import android.view.ViewGroup
import android.view.WindowManager
import android.view.accessibility.AccessibilityEvent
import android.widget.FrameLayout
import android.widget.TextView
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

class IslandService : AccessibilityService() {

    private lateinit var wm: WindowManager
    private lateinit var prefs: SharedPreferences
    private lateinit var lp: WindowManager.LayoutParams
    private var root: FrameLayout? = null
    private var clock: TextView? = null
    private var bg: GradientDrawable? = null

    private val handler = Handler(Looper.getMainLooper())
    private val fmt = SimpleDateFormat("hh:mm", Locale.getDefault())

    // সব মান dp এ
    private var wDp = 120f
    private var hDp = 34f
    private var xDp = 0f
    private var yDp = 8f

    private val density get() = resources.displayMetrics.density
    private fun px(dp: Float) = (dp * density).toInt()

    private val tick = object : Runnable {
        override fun run() {
            clock?.text = fmt.format(Date())
            handler.postDelayed(this, 1000)
        }
    }

    private val listener = SharedPreferences.OnSharedPreferenceChangeListener { _, _ ->
        load()
        applyLayout()
    }

    override fun onServiceConnected() {
        super.onServiceConnected()
        wm = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        prefs = getSharedPreferences("island", Context.MODE_PRIVATE)
        prefs.registerOnSharedPreferenceChangeListener(listener)
        load()
        removeOverlay()
        createOverlay()
    }

    private fun load() {
        wDp = prefs.getFloat("w", 120f)
        hDp = prefs.getFloat("h", 34f)
        xDp = prefs.getFloat("x", 0f)
        yDp = prefs.getFloat("y", 8f)
    }

    private fun save() {
        prefs.edit()
            .putFloat("w", wDp).putFloat("h", hDp)
            .putFloat("x", xDp).putFloat("y", yDp)
            .apply()
    }

    private fun createOverlay() {
        val drawable = GradientDrawable().apply { setColor(Color.BLACK) }
        bg = drawable

        val tv = TextView(this).apply {
            setTextColor(Color.WHITE)
            gravity = Gravity.CENTER
            text = fmt.format(Date())
        }
        clock = tv

        val container = FrameLayout(this).apply {
            background = drawable
            addView(
                tv,
                FrameLayout.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT,
                    ViewGroup.LayoutParams.MATCH_PARENT
                )
            )
        }
        root = container

        lp = WindowManager.LayoutParams(
            px(wDp),
            px(hDp),
            WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS or
                WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.CENTER_HORIZONTAL
            x = px(xDp)
            y = px(yDp)
        }

        // ড্র্যাগ (উপরে/নিচে/ডানে/বামে) + দুই আঙুলে পিঞ্চ করে ছোট-বড়
        val scaleDetector = ScaleGestureDetector(
            this,
            object : ScaleGestureDetector.SimpleOnScaleGestureListener() {
                override fun onScale(d: ScaleGestureDetector): Boolean {
                    wDp = (wDp * d.scaleFactor).coerceIn(60f, 360f)
                    hDp = (hDp * d.scaleFactor).coerceIn(24f, 120f)
                    applyLayout()
                    save()
                    return true
                }
            }
        )
        var lastX = 0f
        var lastY = 0f
        var rebase = false
        container.setOnTouchListener { _, e ->
            scaleDetector.onTouchEvent(e)
            when (e.actionMasked) {
                MotionEvent.ACTION_DOWN -> {
                    lastX = e.rawX
                    lastY = e.rawY
                    rebase = false
                }
                MotionEvent.ACTION_POINTER_UP -> rebase = true
                MotionEvent.ACTION_MOVE -> {
                    if (!scaleDetector.isInProgress && e.pointerCount == 1) {
                        if (rebase) {
                            lastX = e.rawX
                            lastY = e.rawY
                            rebase = false
                        } else {
                            xDp += (e.rawX - lastX) / density
                            yDp = (yDp + (e.rawY - lastY) / density).coerceAtLeast(0f)
                            lastX = e.rawX
                            lastY = e.rawY
                            applyLayout()
                            save()
                        }
                    }
                }
            }
            true
        }

        try {
            wm.addView(container, lp)
        } catch (_: Exception) {
        }
        applyLayout()
        handler.post(tick)
    }

    private fun applyLayout() {
        val r = root ?: return
        lp.width = px(wDp)
        lp.height = px(hDp)
        lp.x = px(xDp)
        lp.y = px(yDp)
        bg?.cornerRadius = px(hDp) / 2f
        clock?.textSize = hDp * 0.4f
        try {
            wm.updateViewLayout(r, lp)
        } catch (_: Exception) {
        }
    }

    private fun removeOverlay() {
        handler.removeCallbacks(tick)
        root?.let {
            try {
                wm.removeView(it)
            } catch (_: Exception) {
            }
        }
        root = null
    }

    override fun onAccessibilityEvent(event: AccessibilityEvent?) {}

    override fun onInterrupt() {}

    override fun onUnbind(intent: android.content.Intent?): Boolean {
        removeOverlay()
        return super.onUnbind(intent)
    }

    override fun onDestroy() {
        removeOverlay()
        super.onDestroy()
    }
}
```

## app/src/main/java/com/island/app/MainActivity.kt

```kotlin
package com.island.app

import android.app.Activity
import android.content.Intent
import android.content.SharedPreferences
import android.net.Uri
import android.os.Bundle
import android.os.PowerManager
import android.provider.Settings
import android.widget.Button
import android.widget.LinearLayout
import android.widget.ScrollView
import android.widget.SeekBar
import android.widget.TextView

class MainActivity : Activity() {

    private lateinit var status: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val prefs = getSharedPreferences("island", MODE_PRIVATE)
        val pad = (16 * resources.displayMetrics.density).toInt()

        val root = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            setPadding(pad, pad, pad, pad)
        }

        status = TextView(this).apply { textSize = 16f }
        root.addView(status)

        root.addView(Button(this).apply {
            text = "১) Accessibility চালু করুন (Island সার্ভিস)"
            setOnClickListener {
                startActivity(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS))
            }
        })

        root.addView(Button(this).apply {
            text = "২) Battery optimization বন্ধ করুন"
            setOnClickListener { askIgnoreBattery() }
        })

        root.addView(slider(prefs, "প্রস্থ (Width)", "w", 60, 360, 120))
        root.addView(slider(prefs, "উচ্চতা (Height)", "h", 24, 120, 34))
        root.addView(slider(prefs, "উপর-নিচ (Y)", "y", 0, 500, 8))
        root.addView(slider(prefs, "ডান-বাম (X)", "x", -200, 200, 0))

        root.addView(TextView(this).apply {
            text = "টিপস: Island এর উপর আঙুল টেনে সরানো যায়, দুই আঙুলে পিঞ্চ করে ছোট-বড় করা যায়।"
            setPadding(0, pad, 0, 0)
        })

        setContentView(ScrollView(this).apply { addView(root) })
    }

    override fun onResume() {
        super.onResume()
        val enabled = Settings.Secure
            .getString(contentResolver, Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES)
            ?.contains(packageName) == true
        status.text = if (enabled) "✅ Island চালু আছে" else "❌ Island বন্ধ — Accessibility চালু করুন"
    }

    private fun askIgnoreBattery() {
        val pm = getSystemService(POWER_SERVICE) as PowerManager
        if (!pm.isIgnoringBatteryOptimizations(packageName)) {
            startActivity(
                Intent(
                    Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS,
                    Uri.parse("package:$packageName")
                )
            )
        }
    }

    private fun slider(
        prefs: SharedPreferences,
        label: String,
        key: String,
        min: Int,
        max: Int,
        def: Int
    ): LinearLayout {
        val box = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL }
        box.setPadding(0, 24, 0, 0)
        box.addView(TextView(this).apply { text = label })
        box.addView(SeekBar(this).apply {
            this.max = max - min
            progress = prefs.getFloat(key, def.toFloat()).toInt() - min
            setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
                override fun onProgressChanged(sb: SeekBar?, p: Int, fromUser: Boolean) {
                    if (fromUser) prefs.edit().putFloat(key, (p + min).toFloat()).apply()
                }
                override fun onStartTrackingTouch(sb: SeekBar?) {}
                override fun onStopTrackingTouch(sb: SeekBar?) {}
            })
        })
        return box
    }
}
```

## app/src/main/res/values/strings.xml

```xml
<resources>
    <string name="service_desc">Island ওভারলে সবসময় স্ক্রিনে (লক স্ক্রিনসহ) দেখানোর জন্য এই সার্ভিস দরকার।</string>
</resources>
```

## app/src/main/res/xml/accessibility_config.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeWindowStateChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:accessibilityFlags="flagDefault"
    android:canRetrieveWindowContent="false"
    android:description="@string/service_desc"
    android:notificationTimeout="100" />
```

## build.gradle.kts

```kotlin
plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}
```

## gradle.properties

```
org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
```

## settings.gradle.kts

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "Island"
include(":app")
```

