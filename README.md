name: Build Boker Shoker APK

on:
  push:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.7'

      - name: Generate Android App Sources
        run: |
          mkdir -p app/src/main/java/com/bokershoker/app app/src/main/res/values
          cat << 'EOF' > app/build.gradle.kts
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.bokershoker.app"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.bokershoker.app"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              buildTypes { release { isMinifyEnabled = false } }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          dependencies {
              implementation("androidx.core:core-ktx:1.13.1")
              implementation("androidx.appcompat:appcompat:1.7.0")
              implementation("com.google.android.material:material:1.12.0")
              implementation("com.google.guava:guava:32.1.3-android")
              val cameraxVersion = "1.3.4"
              implementation("androidx.camera:camera-core:$cameraxVersion")
              implementation("androidx.camera:camera-camera2:$cameraxVersion")
              implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
              implementation("androidx.camera:camera-view:$cameraxVersion")
          }
          EOF

          cat << 'EOF' > app/src/main/res/values/strings.xml
          <resources>
              <string name="app_name">בוקר שוקר</string>
          </resources>
          EOF

          cat << 'EOF' > app/src/main/res/values/themes.xml
          <resources>
              <style name="Theme.BokerShoker" parent="Theme.MaterialComponents.DayNight.NoActionBar">
                  <item name="android:windowBackground">#0E0E10</item>
                  <item name="android:statusBarColor">#0E0E10</item>
                  <item name="android:navigationBarColor">#0E0E10</item>
                  <item name="colorPrimary">#E2B75B</item>
                  <item name="colorOnPrimary">#111114</item>
              </style>
          </resources>
          EOF

          cat << 'EOF' > app/src/main/AndroidManifest.xml
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
              <uses-permission android:name="android.permission.USE_EXACT_ALARM" />
              <uses-permission android:name="android.permission.WAKE_LOCK" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
              <uses-permission android:name="android.permission.VIBRATE" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT" />
              <uses-permission android:name="android.permission.CAMERA" />
              <uses-feature android:name="android.hardware.camera.any" android:required="false" />
              <application
                  android:allowBackup="true"
                  android:label="@string/app_name"
                  android:supportsRtl="true"
                  android:theme="@style/Theme.BokerShoker">
                  <activity android:name=".OnboardingActivity" android:exported="true" android:screenOrientation="portrait">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
                  <activity android:name=".MainActivity" android:exported="false" android:screenOrientation="portrait" />
                  <activity android:name=".AlarmRingingActivity" android:exported="false" android:excludeFromRecents="true" android:launchMode="singleInstance" android:showWhenLocked="true" android:turnScreenOn="true" android:screenOrientation="portrait" />
                  <receiver android:name=".AlarmReceiver" android:exported="false" />
                  <service android:name=".AlarmService" android:exported="false" android:foregroundServiceType="mediaPlayback" />
              </application>
          </manifest>
          EOF

          cat << 'EOF' > app/src/main/java/com/bokershoker/app/OnboardingActivity.kt
          package com.bokershoker.app
          import android.Manifest
          import android.animation.ObjectAnimator
          import android.content.Intent
          import android.content.pm.PackageManager
          import android.graphics.Color
          import android.graphics.drawable.GradientDrawable
          import android.os.Bundle
          import android.view.Gravity
          import android.view.View
          import android.widget.*
          import androidx.activity.result.contract.ActivityResultContracts
          import androidx.appcompat.app.AppCompatActivity
          import androidx.core.content.ContextCompat

          class OnboardingActivity : AppCompatActivity() {
              private var step = 0
              private var gender = "male"
              private var morningPerson = "no"
              private var selectedMission = "pushups"
              private lateinit var stepContainer: LinearLayout
              private lateinit var nextBtn: Button
              private lateinit var progressBar: ProgressBar

              private val cameraPermLauncher = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
                  Toast.makeText(this, if (granted) "גישה למצלמה אושרה" else "ניתן לאשר מצלמה גם בהמשך", Toast.LENGTH_SHORT).show()
                  renderStep()
              }

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val prefs = getSharedPreferences("boker_shoker_prefs", MODE_PRIVATE)
                  if (prefs.getBoolean("onboarding_done", false) && !intent.getBooleanExtra("force_onboarding", false)) {
                      startActivity(Intent(this, MainActivity::class.java))
                      finish()
                      return
                  }
                  val root = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setBackgroundColor(Color.parseColor("#0E0E10"))
                      setPadding(48, 84, 48, 56)
                      layoutDirection = View.LAYOUT_DIRECTION_RTL
                  }
                  root.addView(TextView(this).apply {
                      text = "בוקר שוקר"
                      textSize = 30f
                      setTextColor(Color.parseColor("#F5DFB0"))
                      gravity = Gravity.CENTER
                      setPadding(0, 0, 0, 24)
                  })
                  progressBar = ProgressBar(this, null, android.R.attr.progressBarStyleHorizontal).apply { max = 5; progress = 1 }
                  root.addView(progressBar)
                  val scroll = ScrollView(this).apply { layoutParams = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, 0, 1f) }
                  stepContainer = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setPadding(0, 40, 0, 24) }
                  scroll.addView(stepContainer)
                  root.addView(scroll)
                  nextBtn = Button(this).apply {
                      text = "בנו לי תוכנית בוקר"
                      setTextColor(Color.parseColor("#111114"))
                      textSize = 16f
                      background = GradientDrawable().apply { setColor(Color.parseColor("#E8DEC8")); cornerRadius = 60f }
                      setPadding(24, 34, 24, 34)
                      setOnClickListener {
                          if (step < 4) { step++; renderStep() } else {
                              prefs.edit().putBoolean("onboarding_done", true).putString("gender", gender).putString("morning_person", morningPerson).putString("mission", selectedMission).putInt("reps", 10).apply()
                              startActivity(Intent(this@OnboardingActivity, MainActivity::class.java))
                              finish()
                          }
                      }
                  }
                  root.addView(nextBtn)
                  setContentView(root)
                  renderStep()
              }

              private fun renderStep() {
                  stepContainer.removeAllViews()
                  progressBar.progress = step + 1
                  nextBtn.text = when (step) { 0 -> "בנו לי תוכנית בוקר"; 4 -> "כניסה לאפליקציה"; else -> "המשך" }
                  when (step) {
                      0 -> { addTitle("להתעורר מוקדם. הבוקר מתחיל בצלצול הראשון."); addSub("שעון אחד. משימה אחת. בלי נודניק ובלי הנמכת ווליום.") }
                      1 -> { addTitle("מה המגדר שלך?"); addOption("זכר", gender == "male") { gender = "male"; renderStep() }; addOption("נקבה", gender == "female") { gender = "female"; renderStep() }; addOption("מעדיף לא לענות", gender == "other") { gender = "other"; renderStep() } }
                      2 -> { addTitle(if (gender == "female") "את רואה את עצמך אשת של בוקר?" else "אתה רואה את עצמך איש של בוקר?"); addOption("כן", morningPerson == "yes") { morningPerson = "yes"; renderStep() }; addOption("לא", morningPerson == "no") { morningPerson = "no"; renderStep() } }
                      3 -> { addTitle("איזו משימה תוציא אותך מהמיטה?"); listOf("pushups" to "שכיבות סמיכה (מצלמה)", "toothbrush" to "מברשת שיניים", "sky" to "צילום שמיים", "bed" to "סידור המיטה", "math" to "תרגילי חשבון").forEach { (id, lbl) -> addOption(lbl, selectedMission == id) { selectedMission = id; renderStep() } } }
                      4 -> { addTitle("הרשאת מצלמה לאימות ביצוע"); addSub("המצלמה פועלת מקומית בלבד כדי לוודא ביצוע אמיתי."); val ok = ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) == PackageManager.PERMISSION_GRANTED; addOption(if (ok) "גישה למצלמה אושרה ✓" else "לחץ לאישור גישה למצלמה", ok) { if (!ok) cameraPermLauncher.launch(Manifest.permission.CAMERA) } }
                  }
                  stepContainer.alpha = 0f; stepContainer.translationY = 30f
                  ObjectAnimator.ofFloat(stepContainer, "alpha", 0f, 1f).setDuration(280).start()
                  ObjectAnimator.ofFloat(stepContainer, "translationY", 30f, 0f).setDuration(280).start()
              }

              private fun addTitle(t: String) { stepContainer.addView(TextView(this).apply { text = t; textSize = 24f; setTextColor(Color.WHITE); setPadding(0, 0, 0, 24) }) }
              private fun addSub(t: String) { stepContainer.addView(TextView(this).apply { text = t; textSize = 15f; setTextColor(Color.parseColor("#A1A1AA")); setPadding(0, 0, 0, 32) }) }
              private fun addOption(lbl: String, sel: Boolean, cb: () -> Unit) {
                  stepContainer.addView(Button(this).apply {
                      text = lbl; setTextColor(Color.WHITE); textSize = 15f; gravity = Gravity.CENTER_VERTICAL or Gravity.START; setPadding(40, 34, 40, 34)
                      background = GradientDrawable().apply { setColor(Color.parseColor(if (sel) "#211D15" else "#17171A")); setStroke(3, Color.parseColor(if (sel) "#E2B75B" else "#27272D")); cornerRadius = 36f }
                      layoutParams = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT).apply { bottomMargin = 22 }
                      setOnClickListener { cb() }
                  })
              }
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/bokershoker/app/MainActivity.kt
          package com.bokershoker.app
          import android.app.AlarmManager
          import android.app.PendingIntent
          import android.content.Context
          import android.content.Intent
          import android.graphics.Color
          import android.graphics.drawable.GradientDrawable
          import android.media.RingtoneManager
          import android.net.Uri
          import android.os.Build
          import android.os.Bundle
          import android.view.Gravity
          import android.view.View
          import android.widget.*
          import androidx.activity.result.contract.ActivityResultContracts
          import androidx.appcompat.app.AppCompatActivity
          import java.util.Calendar

          class MainActivity : AppCompatActivity() {
              private lateinit var ringtoneLabel: TextView
              private val ringtonePicker = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
                  val uri: Uri? = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                      result.data?.getParcelableExtra(RingtoneManager.EXTRA_RINGTONE_PICKED_URI, Uri::class.java)
                  } else {
                      @Suppress("DEPRECATION") result.data?.getParcelableExtra(RingtoneManager.EXTRA_RINGTONE_PICKED_URI)
                  }
                  if (uri != null) {
                      getSharedPreferences("boker_shoker_prefs", MODE_PRIVATE).edit().putString("alarm_ringtone_uri", uri.toString()).apply()
                      val title = RingtoneManager.getRingtone(this, uri)?.getTitle(this) ?: "צלצול הטלפון"
                      ringtoneLabel.text = "צלצול נבחר: $title"
                  }
              }

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val prefs = getSharedPreferences("boker_shoker_prefs", MODE_PRIVATE)
                  val root = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL; setBackgroundColor(Color.parseColor("#0E0E10")); setPadding(48, 72, 48, 48); layoutDirection = View.LAYOUT_DIRECTION_RTL
                  }
                  root.addView(TextView(this).apply { text = "בוקר שוקר"; textSize = 28f; setTextColor(Color.parseColor("#F5DFB0")); gravity = Gravity.CENTER; setPadding(0, 0, 0, 12) })
                  val timePicker = TimePicker(this).apply { setIs24HourView(true) }
                  root.addView(timePicker)
                  ringtoneLabel = TextView(this).apply {
                      val saved = prefs.getString("alarm_ringtone_uri", null)
                      val t = if (saved != null) RingtoneManager.getRingtone(this@MainActivity, Uri.parse(saved))?.getTitle(this@MainActivity) ?: "צלצול הטלפון" else "צלצול ברירת המחדל של הטלפון"
                      text = "צלצול נבחר: $t"; textSize = 13f; setTextColor(Color.parseColor("#E2B75B")); setPadding(0, 20, 0, 10)
                  }
                  root.addView(ringtoneLabel)
                  root.addView(makeBtn("בחר צלצול מהטלפון", "#17171A", "#F5DFB0") {
                      ringtonePicker.launch(Intent(RingtoneManager.ACTION_RINGTONE_PICKER).apply {
                          putExtra(RingtoneManager.EXTRA_RINGTONE_TYPE, RingtoneManager.TYPE_ALARM)
                          putExtra(RingtoneManager.EXTRA_RINGTONE_SHOW_DEFAULT, true)
                          putExtra(RingtoneManager.EXTRA_RINGTONE_SHOW_SILENT, false)
                      })
                  })
                  root.addView(makeBtn("קבע שעון מעורר", "#E8DEC8", "#111114") { scheduleAlarm(timePicker.hour, timePicker.minute) })
                  root.addView(makeBtn("הפעל אזעקה עכשיו (בדיקה)", "#211D15", "#E2B75B") {
                      startForegroundService(Intent(this, AlarmService::class.java))
                      startActivity(Intent(this, AlarmRingingActivity::class.java))
                  })
                  setContentView(root)
              }

              private fun scheduleAlarm(hour: Int, minute: Int) {
                  val cal = Calendar.getInstance().apply {
                      set(Calendar.HOUR_OF_DAY, hour); set(Calendar.MINUTE, minute); set(Calendar.SECOND, 0)
                      if (timeInMillis <= System.currentTimeMillis()) add(Calendar.DAY_OF_YEAR, 1)
                  }
                  val am = getSystemService(Context.ALARM_SERVICE) as AlarmManager
                  val pending = PendingIntent.getBroadcast(this, 1001, Intent(this, AlarmReceiver::class.java), PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
                  try {
                      am.setAlarmClock(AlarmManager.AlarmClockInfo(cal.timeInMillis, pending), pending)
                      Toast.makeText(this, String.format("שעון בוקר שוקר נקבע לשעה %02d:%02d", hour, minute), Toast.LENGTH_LONG).show()
                  } catch (e: SecurityException) {
                      Toast.makeText(this, "יש לאשר הרשאת שעונים מדויקים בהגדרות הטלפון", Toast.LENGTH_LONG).show()
                  }
              }

              private fun makeBtn(lbl: String, bg: String, fg: String, cb: () -> Unit): Button = Button(this).apply {
                  text = lbl; setTextColor(Color.parseColor(fg)); textSize = 15f
                  background = GradientDrawable().apply { setColor(Color.parseColor(bg)); cornerRadius = 56f }
                  setPadding(24, 30, 24, 30)
                  layoutParams = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT).apply { topMargin = 20 }
                  setOnClickListener { cb() }
              }
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/bokershoker/app/AlarmReceiver.kt
          package com.bokershoker.app
          import android.content.BroadcastReceiver
          import android.content.Context
          import android.content.Intent
          class AlarmReceiver : BroadcastReceiver() {
              override fun onReceive(context: Context, intent: Intent) {
                  context.startForegroundService(Intent(context, AlarmService::class.java))
                  context.startActivity(Intent(context, AlarmRingingActivity::class.java).apply { addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP) })
              }
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/bokershoker/app/AlarmService.kt
          package com.bokershoker.app
          import android.app.*
          import android.content.Context
          import android.content.Intent
          import android.media.AudioAttributes
          import android.media.AudioManager
          import android.media.MediaPlayer
          import android.media.RingtoneManager
          import android.net.Uri
          import android.os.*

          class AlarmService : Service() {
              private var player: MediaPlayer? = null
              private var vibrator: Vibrator? = null
              private val handler = Handler(Looper.getMainLooper())
              private lateinit var audioManager: AudioManager
              private val lockRunnable = object : Runnable {
                  override fun run() {
                      val maxVol = audioManager.getStreamMaxVolume(AudioManager.STREAM_ALARM)
                      if (audioManager.getStreamVolume(AudioManager.STREAM_ALARM) < maxVol) audioManager.setStreamVolume(AudioManager.STREAM_ALARM, maxVol, 0)
                      handler.postDelayed(this, 250)
                  }
              }
              override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
                  audioManager = getSystemService(Context.AUDIO_SERVICE) as AudioManager
                  startForeground(1, buildNotif())
                  val saved = getSharedPreferences("boker_shoker_prefs", MODE_PRIVATE).getString("alarm_ringtone_uri", null)
                  val uri = if (!saved.isNullOrEmpty()) Uri.parse(saved) else RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM) ?: RingtoneManager.getDefaultUri(RingtoneManager.TYPE_RINGTONE)
                  audioManager.setStreamVolume(AudioManager.STREAM_ALARM, audioManager.getStreamMaxVolume(AudioManager.STREAM_ALARM), 0)
                  player?.release()
                  player = MediaPlayer().apply {
                      setDataSource(applicationContext, uri)
                      setAudioAttributes(AudioAttributes.Builder().setUsage(AudioAttributes.USAGE_ALARM).setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION).build())
                      isLooping = true; prepare(); start()
                  }
                  @Suppress("DEPRECATION")
                  vibrator = getSystemService(Context.VIBRATOR_SERVICE) as Vibrator
                  vibrator?.vibrate(VibrationEffect.createWaveform(longArrayOf(0, 600, 400), 0))
                  handler.post(lockRunnable)
                  return START_STICKY
              }
              private fun buildNotif(): Notification {
                  val ch = "boker_shoker_alarm"
                  getSystemService(NotificationManager::class.java).createNotificationChannel(NotificationChannel(ch, "בוקר שוקר", NotificationManager.IMPORTANCE_HIGH))
                  val pi = PendingIntent.getActivity(this, 0, Intent(this, AlarmRingingActivity::class.java), PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
                  return Notification.Builder(this, ch).setContentTitle("בוקר שוקר — קום!").setContentText("השעון לא ייכבה עד שתסיים את המשימה").setSmallIcon(android.R.drawable.ic_lock_idle_alarm).setFullScreenIntent(pi, true).setOngoing(true).build()
              }
              override fun onDestroy() { handler.removeCallbacks(lockRunnable); player?.stop(); player?.release(); vibrator?.cancel(); super.onDestroy() }
              override fun onBind(intent: Intent?): IBinder? = null
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/bokershoker/app/AlarmRingingActivity.kt
          package com.bokershoker.app
          import android.Manifest
          import android.content.Context
          import android.content.Intent
          import android.content.pm.PackageManager
          import android.graphics.Color
          import android.graphics.drawable.GradientDrawable
          import android.hardware.Sensor
          import android.hardware.SensorEvent
          import android.hardware.SensorEventListener
          import android.hardware.SensorManager
          import android.media.AudioManager
          import android.os.Bundle
          import android.view.Gravity
          import android.view.KeyEvent
          import android.view.View
          import android.view.WindowManager
          import android.widget.*
          import androidx.activity.OnBackPressedCallback
          import androidx.appcompat.app.AppCompatActivity
          import androidx.camera.core.CameraSelector
          import androidx.camera.core.ImageAnalysis
          import androidx.camera.core.Preview
          import androidx.camera.lifecycle.ProcessCameraProvider
          import androidx.camera.view.PreviewView
          import androidx.core.content.ContextCompat
          import java.util.concurrent.Executors

          class AlarmRingingActivity : AppCompatActivity(), SensorEventListener {
              private lateinit var audioManager: AudioManager
              private lateinit var sensorManager: SensorManager
              private var prox: Sensor? = null
              private val camExec = Executors.newSingleThreadExecutor()
              private var repsDone = 0
              private var repsTarget = 10
              private var isDown = false
              private var baseLuma = 0.0
              private var warmup = 0
              private var lastRepMs = 0L
              private lateinit var counterView: TextView

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  setShowWhenLocked(true); setTurnScreenOn(true); window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
                  audioManager = getSystemService(Context.AUDIO_SERVICE) as AudioManager
                  sensorManager = getSystemService(Context.SENSOR_SERVICE) as SensorManager
                  prox = sensorManager.getDefaultSensor(Sensor.TYPE_PROXIMITY)
                  onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
                      override fun handleOnBackPressed() { Toast.makeText(this@AlarmRingingActivity, "אין נודניק! סיים את המשימה", Toast.LENGTH_SHORT).show() }
                  })
                  repsTarget = getSharedPreferences("boker_shoker_prefs", MODE_PRIVATE).getInt("reps", 10)
                  val root = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setBackgroundColor(Color.parseColor("#0E0E10")); setPadding(48, 72, 48, 48); gravity = Gravity.CENTER_HORIZONTAL; layoutDirection = View.LAYOUT_DIRECTION_RTL }
                  root.addView(TextView(this).apply { text = "בוקר שוקר"; textSize = 28f; setTextColor(Color.parseColor("#F5DFB0")); gravity = Gravity.CENTER })
                  root.addView(TextView(this).apply { text = "נעילת עוצמה 100% פעילה · בצע שכיבות סמיכה"; textSize = 14f; setTextColor(Color.parseColor("#E2B75B")); gravity = Gravity.CENTER; setPadding(0, 16, 0, 24) })
                  val pv = PreviewView(this).apply { layoutParams = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, 520).apply { bottomMargin = 24 } }
                  root.addView(pv)
                  counterView = TextView(this).apply { text = "0 / $repsTarget"; textSize = 48f; setTextColor(Color.WHITE); gravity = Gravity.CENTER; setPadding(0, 8, 0, 24) }
                  root.addView(counterView)
                  root.addView(Button(this).apply {
                      text = "ספירת שכיבה (מגע אף במסך)"; setTextColor(Color.parseColor("#111114")); textSize = 16f
                      background = GradientDrawable().apply { setColor(Color.parseColor("#E8DEC8")); cornerRadius = 60f }
                      setPadding(24, 34, 24, 34); setOnClickListener { countRep() }
                  })
                  setContentView(root)
                  if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) == PackageManager.PERMISSION_GRANTED) startCam(pv)
              }

              private fun startCam(pv: PreviewView) {
                  val f = ProcessCameraProvider.getInstance(this)
                  f.addListener({
                      val p = f.get()
                      val prev = Preview.Builder().build().also { it.setSurfaceProvider(pv.surfaceProvider) }
                      val an = ImageAnalysis.Builder().setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST).build()
                      an.setAnalyzer(camExec) { img ->
                          val buf = img.planes[0].buffer; val b = ByteArray(buf.remaining()); buf.get(b)
                          var s = 0L; for (x in b) s += (x.toInt() and 0xFF)
                          val avg = s.toDouble() / b.size.coerceAtLeast(1)
                          if (warmup < 8) { baseLuma = (baseLuma * warmup + avg) / (warmup + 1); warmup++ } else {
                              val now = System.currentTimeMillis()
                              if (!isDown && avg < baseLuma * 0.72) isDown = true
                              else if (isDown && avg > baseLuma * 0.85 && now - lastRepMs > 650) { isDown = false; lastRepMs = now; runOnUiThread { countRep() } }
                          }
                          img.close()
                      }
                      p.unbindAll(); p.bindToLifecycle(this, CameraSelector.DEFAULT_FRONT_CAMERA, prev, an)
                  }, ContextCompat.getMainExecutor(this))
              }

              private fun countRep() {
                  repsDone++; counterView.text = "$repsDone / $repsTarget"
                  if (repsDone >= repsTarget) { stopService(Intent(this, AlarmService::class.java)); Toast.makeText(this, "כל הכבוד! השעון כובה", Toast.LENGTH_LONG).show(); finish() }
              }

              override fun dispatchKeyEvent(event: KeyEvent): Boolean = when (event.keyCode) {
                  KeyEvent.KEYCODE_VOLUME_DOWN, KeyEvent.KEYCODE_VOLUME_UP, KeyEvent.KEYCODE_VOLUME_MUTE -> {
                      audioManager.setStreamVolume(AudioManager.STREAM_ALARM, audioManager.getStreamMaxVolume(AudioManager.STREAM_ALARM), 0)
                      Toast.makeText(this, "אי אפשר להנמיך את בוקר שוקר!", Toast.LENGTH_SHORT).show(); true
                  }
                  else -> super.dispatchKeyEvent(event)
              }
              override fun onResume() { super.onResume(); prox?.let { sensorManager.registerListener(this, it, SensorManager.SENSOR_DELAY_NORMAL) } }
              override fun onPause() { super.onPause(); sensorManager.unregisterListener(this) }
              override fun onSensorChanged(e: SensorEvent) { if (e.sensor.type == Sensor.TYPE_PROXIMITY) { if (e.values[0] < e.sensor.maximumRange) isDown = true else if (isDown) { isDown = false; countRep() } } }
              override fun onAccuracyChanged(s: Sensor?, a: Int) {}
              override fun onDestroy() { camExec.shutdown(); super.onDestroy() }
          }
          EOF

      - name: Build Debug APK
        run: gradle assembleDebug --no-daemon

      - name: Upload BokerShoker APK
        uses: actions/upload-artifact@v4
        with:
          name: BokerShoker-APK
          path: app/build/outputs/apk/debug/app-debug.apk

