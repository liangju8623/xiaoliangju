name: Build APK

on:
  push:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.6'
          cache-disabled: true

      - name: Generate Project & Compile APK
        run: |
          mkdir -p app/src/main/java/com/example/touchburst
          mkdir -p app/src/main/assets

          cat << 'EOF' > settings.gradle
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
                  maven { url "https://api.xposed.info/" }
              }
          }
          rootProject.name = "TouchBurst"
          include ':app'
          EOF

          cat << 'EOF' > build.gradle
          plugins {
              id 'com.android.application' version '8.2.2' apply false
          }
          EOF

          cat << 'EOF' > app/build.gradle
          plugins {
              id 'com.android.application'
          }
          android {
              namespace 'com.example.touchburst'
              compileSdk 34
              defaultConfig {
                  applicationId "com.example.touchburst"
                  minSdk 24
                  targetSdk 34
                  versionCode 2
                  versionName "2.0"
              }
              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_17
                  targetCompatibility JavaVersion.VERSION_17
              }
          }
          dependencies {
              compileOnly 'de.robv.android.xposed:api:82'
          }
          EOF

          cat << 'EOF' > app/src/main/AndroidManifest.xml
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <application
                  android:label="按住连点器"
                  android:theme="@android:style/Theme.Material.Light.NoActionBar">

                  <meta-data android:name="xposedmodule" android:value="true" />
                  <meta-data android:name="xposeddescription" android:value="按住屏幕快速连点+随机微偏移，松手即停，带桌面测试台与悬浮调参" />
                  <meta-data android:name="xposedminversion" android:value="82" />

                  <activity
                      android:name=".MainActivity"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                          <category android:name="de.robv.android.xposed.category.MODULE_SETTINGS" />
                      </intent-filter>
                  </activity>
              </application>
          </manifest>
          EOF

          echo "com.example.touchburst.MainHook" > app/src/main/assets/xposed_init

          cat << 'EOF' > app/src/main/java/com/example/touchburst/TouchBurstEngine.java
          package com.example.touchburst;

          import android.app.Activity;
          import android.app.AlertDialog;
          import android.content.Context;
          import android.content.DialogInterface;
          import android.content.SharedPreferences;
          import android.graphics.Color;
          import android.graphics.drawable.GradientDrawable;
          import android.os.Handler;
          import android.os.Looper;
          import android.os.SystemClock;
          import android.view.Gravity;
          import android.view.InputDevice;
          import android.view.MotionEvent;
          import android.view.View;
          import android.view.ViewGroup;
          import android.view.WindowManager;
          import android.widget.FrameLayout;
          import android.widget.LinearLayout;
          import android.widget.SeekBar;
          import android.widget.TextView;
          import java.util.Random;

          public class TouchBurstEngine {
              private static final int TAG_ATTACHED = 0x7E001001;
              private final View rootView;
              private final Context context;
              private final SharedPreferences prefs;
              private final Handler mainHandler = new Handler(Looper.getMainLooper());
              private final Random random = new Random();

              private boolean isEnabled = true;
              private int holdTriggerMs = 110;
              private int minIntervalMs = 35;
              private int maxIntervalMs = 65;
              private int offsetRangePx = 8;

              private boolean isDispatchingSynthetic = false;
              private boolean ignoreCurrentGesture = false;
              private boolean isBursting = false;
              private boolean isSwiping = false;
              private boolean isSyntheticDown = false;

              private float downX = 0f, downY = 0f;
              private float currentX = 0f, currentY = 0f;
              private float lastSyntheticX = 0f, lastSyntheticY = 0f;
              private long physicalDownTime = 0L;
              private long syntheticDownTime = 0L;
              private int lastDeviceId = 0;

              private LinearLayout floatingCapsule;

              public TouchBurstEngine(View rootView) {
                  this.rootView = rootView;
                  this.context = rootView.getContext();
                  this.prefs = context.getSharedPreferences("touch_burst_config", Context.MODE_PRIVATE);
                  loadPrefs();
              }

              private void loadPrefs() {
                  holdTriggerMs = prefs.getInt("hold_ms", 110);
                  minIntervalMs = prefs.getInt("min_interval_ms", 35);
                  maxIntervalMs = Math.max(minIntervalMs + 10, prefs.getInt("max_interval_ms", 65));
                  offsetRangePx = prefs.getInt("offset_px", 8);
              }

              private void savePrefs() {
                  prefs.edit()
                      .putInt("hold_ms", holdTriggerMs)
                      .putInt("min_interval_ms", minIntervalMs)
                      .putInt("max_interval_ms", maxIntervalMs)
                      .putInt("offset_px", offsetRangePx)
                      .apply();
              }

              private final Runnable startBurstRunnable = new Runnable() {
                  @Override
                  public void run() {
                      if (isSwiping || !isEnabled || ignoreCurrentGesture) return;
                      isBursting = true;
                      sendSyntheticEvent(MotionEvent.ACTION_UP, downX, downY, physicalDownTime);
                      mainHandler.postDelayed(burstLoopRunnable, 18);
                  }
              };

              private final Runnable burstLoopRunnable = new Runnable() {
                  @Override
                  public void run() {
                      if (!isBursting || !isEnabled) return;

                      float ox = (random.nextFloat() * 2f - 1f) * offsetRangePx;
                      float oy = (random.nextFloat() * 2f - 1f) * offsetRangePx;
                      lastSyntheticX = Math.max(1f, currentX + ox);
                      lastSyntheticY = Math.max(1f, currentY + oy);

                      syntheticDownTime = SystemClock.uptimeMillis();
                      isSyntheticDown = true;
                      sendSyntheticEvent(MotionEvent.ACTION_DOWN, lastSyntheticX, lastSyntheticY, syntheticDownTime);

                      int pressDuration = 12 + random.nextInt(12);
                      mainHandler.postDelayed(upRunnable, pressDuration);

                      int nextDelay = minIntervalMs + random.nextInt(Math.max(1, maxIntervalMs - minIntervalMs + 1));
                      mainHandler.postDelayed(this, pressDuration + nextDelay);
                  }
              };

              private final Runnable upRunnable = new Runnable() {
                  @Override
                  public void run() {
                      if (isSyntheticDown) {
                          sendSyntheticEvent(MotionEvent.ACTION_UP, lastSyntheticX, lastSyntheticY, syntheticDownTime);
                          isSyntheticDown = false;
                      }
                  }
              };

              public boolean onRootTouchEvent(MotionEvent event) {
                  if (isDispatchingSynthetic) return false;

                  ViewGroup.LayoutParams rlp = rootView.getLayoutParams();
                  if (rlp instanceof WindowManager.LayoutParams) {
                      int type = ((WindowManager.LayoutParams) rlp).type;
                      if (type != WindowManager.LayoutParams.TYPE_BASE_APPLICATION && type != WindowManager.LayoutParams.TYPE_APPLICATION) {
                          return false;
                      }
                      if (type == WindowManager.LayoutParams.TYPE_APPLICATION && !(rootView.getContext() instanceof Activity)) {
                          return false;
                      }
                  }

                  ensureFloatingCapsule();

                  int action = event.getActionMasked();
                  if (action == MotionEvent.ACTION_DOWN) {
                      if (isPointInsideFloatingCapsule(event.getRawX(), event.getRawY())) {
                          ignoreCurrentGesture = true;
                          return false;
                      }
                      ignoreCurrentGesture = false;
                  }

                  if (ignoreCurrentGesture) {
                      if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
                          ignoreCurrentGesture = false;
                      }
                      return false;
                  }

                  if (!isEnabled) return false;

                  if (event.getPointerCount() > 1) {
                      stopBurstCleanly();
                      isSwiping = true;
                      return false;
                  }

                  if (action == MotionEvent.ACTION_DOWN) {
                      downX = event.getX();
                      downY = event.getY();
                      currentX = downX;
                      currentY = downY;
                      physicalDownTime = event.getDownTime();
                      lastDeviceId = event.getDeviceId();
                      isSwiping = false;
                      isBursting = false;
                      mainHandler.removeCallbacks(startBurstRunnable);
                      mainHandler.postDelayed(startBurstRunnable, holdTriggerMs);
                      return false;
                  } else if (action == MotionEvent.ACTION_MOVE) {
                      currentX = event.getX();
                      currentY = event.getY();
                      if (isBursting) {
                          return true;
                      } else if (!isSwiping) {
                          double dist = Math.hypot(currentX - downX, currentY - downY);
                          if (dist > 18f) {
                              isSwiping = true;
                              mainHandler.removeCallbacks(startBurstRunnable);
                          }
                      }
                      return false;
                  } else if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
                      mainHandler.removeCallbacks(startBurstRunnable);
                      if (isBursting) {
                          stopBurstCleanly();
                          return true;
                      }
                      isSwiping = false;
                      return false;
                  }
                  return false;
              }

              private void stopBurstCleanly() {
                  mainHandler.removeCallbacks(startBurstRunnable);
                  mainHandler.removeCallbacks(burstLoopRunnable);
                  mainHandler.removeCallbacks(upRunnable);
                  if (isSyntheticDown) {
                      sendSyntheticEvent(MotionEvent.ACTION_UP, lastSyntheticX, lastSyntheticY, syntheticDownTime);
                      isSyntheticDown = false;
                  }
                  isBursting = false;
              }

              private void sendSyntheticEvent(int action, float x, float y, long downTime) {
                  long now = SystemClock.uptimeMillis();
                  MotionEvent.PointerProperties[] props = new MotionEvent.PointerProperties[1];
                  props[0] = new MotionEvent.PointerProperties();
                  props[0].id = 0;
                  props[0].toolType = MotionEvent.TOOL_TYPE_FINGER;

                  MotionEvent.PointerCoords[] coords = new MotionEvent.PointerCoords[1];
                  coords[0] = new MotionEvent.PointerCoords();
                  coords[0].x = x;
                  coords[0].y = y;
                  coords[0].pressure = action == MotionEvent.ACTION_DOWN ? (0.45f + random.nextFloat() * 0.2f) : 0f;
                  coords[0].size = 0.06f + random.nextFloat() * 0.03f;

                  MotionEvent synthetic = MotionEvent.obtain(
                      downTime, now, action, 1, props, coords,
                      0, 0, 1.0f, 1.0f, lastDeviceId, 0,
                      InputDevice.SOURCE_TOUCHSCREEN, 0
                  );
                  try {
                      isDispatchingSynthetic = true;
                      rootView.dispatchTouchEvent(synthetic);
                  } finally {
                      isDispatchingSynthetic = false;
                      synthetic.recycle();
                  }
              }

              private boolean isPointInsideFloatingCapsule(float rawX, float rawY) {
                  if (floatingCapsule == null || floatingCapsule.getVisibility() != View.VISIBLE) return false;
                  int[] loc = new int[2];
                  floatingCapsule.getLocationOnScreen(loc);
                  return rawX >= loc[0] - 10 && rawX <= loc[0] + floatingCapsule.getWidth() + 10
                      && rawY >= loc[1] - 10 && rawY <= loc[1] + floatingCapsule.getHeight() + 10;
              }

              public void ensureFloatingCapsule() {
                  if (!(rootView instanceof FrameLayout)) return;
                  if (rootView.getTag(TAG_ATTACHED) != null) return;
                  rootView.setTag(TAG_ATTACHED, Boolean.TRUE);

                  rootView.post(new Runnable() {
                      @Override
                      public void run() {
                          attachCapsuleInternal((FrameLayout) rootView);
                      }
                  });
              }

              private void attachCapsuleInternal(FrameLayout decor) {
                  floatingCapsule = new LinearLayout(context);
                  floatingCapsule.setOrientation(LinearLayout.HORIZONTAL);
                  floatingCapsule.setGravity(Gravity.CENTER_VERTICAL);
                  floatingCapsule.setElevation(1000f);
                  floatingCapsule.setTranslationZ(1000f);

                  final GradientDrawable bg = new GradientDrawable();
                  bg.setCornerRadius(60f);
                  bg.setColor(Color.parseColor("#D91E88E5"));
                  bg.setStroke(2, Color.parseColor("#66FFFFFF"));
                  floatingCapsule.setBackground(bg);

                  final TextView toggleBtn = new TextView(context);
                  toggleBtn.setText("连点:开");
                  toggleBtn.setTextColor(Color.WHITE);
                  toggleBtn.setTextSize(12f);
                  toggleBtn.setPadding(28, 14, 16, 14);

                  TextView cfgBtn = new TextView(context);
                  cfgBtn.setText("⚙");
                  cfgBtn.setTextColor(Color.WHITE);
                  cfgBtn.setTextSize(13f);
                  cfgBtn.setPadding(16, 14, 26, 14);

                  floatingCapsule.addView(toggleBtn);
                  floatingCapsule.addView(cfgBtn);

                  toggleBtn.setOnTouchListener(new View.OnTouchListener() {
                      float dX, dY, startRawX, startRawY;
                      boolean moved = false;
                      @Override
                      public boolean onTouch(View v, MotionEvent event) {
                          switch (event.getActionMasked()) {
                              case MotionEvent.ACTION_DOWN:
                                  startRawX = event.getRawX();
                                  startRawY = event.getRawY();
                                  dX = floatingCapsule.getX() - startRawX;
                                  dY = floatingCapsule.getY() - startRawY;
                                  moved = false;
                                  return true;
                              case MotionEvent.ACTION_MOVE:
                                  if (Math.hypot(event.getRawX() - startRawX, event.getRawY() - startRawY) > 12f) {
                                      moved = true;
                                      floatingCapsule.setX(event.getRawX() + dX);
                                      floatingCapsule.setY(event.getRawY() + dY);
                                  }
                                  return true;
                              case MotionEvent.ACTION_UP:
                                  if (!moved) {
                                      isEnabled = !isEnabled;
                                      toggleBtn.setText(isEnabled ? "连点:开" : "连点:关");
                                      bg.setColor(Color.parseColor(isEnabled ? "#D91E88E5" : "#D9616161"));
                                  }
                                  return true;
                          }
                          return true;
                      }
                  });

                  cfgBtn.setOnClickListener(new View.OnClickListener() {
                      @Override
                      public void onClick(View v) {
                          showConfigDialog();
                      }
                  });

                  FrameLayout.LayoutParams lp = new FrameLayout.LayoutParams(
                      FrameLayout.LayoutParams.WRAP_CONTENT,
                      FrameLayout.LayoutParams.WRAP_CONTENT
                  );
                  lp.gravity = Gravity.TOP | Gravity.END;
                  lp.topMargin = 180;
                  lp.rightMargin = 36;
                  decor.addView(floatingCapsule, lp);
              }

              public void showConfigDialog() {
                  loadPrefs();
                  LinearLayout layout = new LinearLayout(context);
                  layout.setOrientation(LinearLayout.VERTICAL);
                  layout.setPadding(56, 40, 56, 20);

                  final TextView tvHold = new TextView(context);
                  tvHold.setText("按住触发时间: " + holdTriggerMs + " 毫秒 (低于此时间为普通点击/划屏)");
                  SeekBar sbHold = new SeekBar(context);
                  sbHold.setMax(350);
                  sbHold.setProgress(Math.max(0, holdTriggerMs - 40));
                  sbHold.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
                      @Override public void onProgressChanged(SeekBar s, int p, boolean b) {
                          holdTriggerMs = 40 + p;
                          tvHold.setText("按住触发时间: " + holdTriggerMs + " 毫秒 (低于此时间为普通点击/划屏)");
                      }
                      @Override public void onStartTrackingTouch(SeekBar s) {}
                      @Override public void onStopTrackingTouch(SeekBar s) {}
                  });

                  final TextView tvSpeed = new TextView(context);
                  tvSpeed.setText("连点最小间隔: " + minIntervalMs + " ~ " + maxIntervalMs + " 毫秒 (随机波动)");
                  SeekBar sbSpeed = new SeekBar(context);
                  sbSpeed.setMax(150);
                  sbSpeed.setProgress(Math.max(0, minIntervalMs - 10));
                  sbSpeed.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
                      @Override public void onProgressChanged(SeekBar s, int p, boolean b) {
                          minIntervalMs = 10 + p;
                          maxIntervalMs = minIntervalMs + 25;
                          tvSpeed.setText("连点最小间隔: " + minIntervalMs + " ~ " + maxIntervalMs + " 毫秒 (随机波动)");
                      }
                      @Override public void onStartTrackingTouch(SeekBar s) {}
                      @Override public void onStopTrackingTouch(SeekBar s) {}
                  });

                  final TextView tvOffset = new TextView(context);
                  tvOffset.setText("随机坐标偏移半径: ±" + offsetRangePx + " 像素");
                  SeekBar sbOffset = new SeekBar(context);
                  sbOffset.setMax(40);
                  sbOffset.setProgress(offsetRangePx);
                  sbOffset.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
                      @Override public void onProgressChanged(SeekBar s, int p, boolean b) {
                          offsetRangePx = p;
                          tvOffset.setText("随机坐标偏移半径: ±" + offsetRangePx + " 像素");
                      }
                      @Override public void onStartTrackingTouch(SeekBar s) {}
                      @Override public void onStopTrackingTouch(SeekBar s) {}
                  });

                  layout.addView(tvHold);
                  layout.addView(sbHold);
                  layout.addView(tvSpeed);
                  layout.addView(sbSpeed);
                  layout.addView(tvOffset);
                  layout.addView(sbOffset);

                  new AlertDialog.Builder(context)
                      .setTitle("连点器参数调节")
                      .setView(layout)
                      .setPositiveButton("保存生效", new DialogInterface.OnClickListener() {
                          @Override
                          public void onClick(DialogInterface dialog, int which) {
                              savePrefs();
                          }
                      })
                      .show();
              }
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/example/touchburst/MainHook.java
          package com.example.touchburst;

          import android.app.Activity;
          import android.view.InputDevice;
          import android.view.MotionEvent;
          import android.view.View;
          import java.util.WeakHashMap;
          import de.robv.android.xposed.IXposedHookLoadPackage;
          import de.robv.android.xposed.XC_MethodHook;
          import de.robv.android.xposed.XC_MethodReplacement;
          import de.robv.android.xposed.XposedHelpers;
          import de.robv.android.xposed.callbacks.XC_LoadPackage;

          public class MainHook implements IXposedHookLoadPackage {
              private static final WeakHashMap<View, TouchBurstEngine> engines = new WeakHashMap<>();

              private static synchronized TouchBurstEngine getEngine(View rootView) {
                  TouchBurstEngine engine = engines.get(rootView);
                  if (engine == null) {
                      engine = new TouchBurstEngine(rootView);
                      engines.put(rootView, engine);
                  }
                  return engine;
              }

              @Override
              public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
                  if ("android".equals(lpparam.packageName) || lpparam.packageName.startsWith("com.android.systemui")) {
                      return;
                  }

                  if ("com.example.touchburst".equals(lpparam.packageName)) {
                      XposedHelpers.findAndHookMethod(
                          "com.example.touchburst.MainActivity",
                          lpparam.classLoader,
                          "isLSPosedModuleActive",
                          XC_MethodReplacement.returnConstant(true)
                      );
                  }

                  XposedHelpers.findAndHookMethod(Activity.class, "onResume", new XC_MethodHook() {
                      @Override
                      protected void afterHookedMethod(MethodHookParam param) {
                          Activity act = (Activity) param.thisObject;
                          if (act.getWindow() != null && act.getWindow().getDecorView() != null) {
                              View root = act.getWindow().getDecorView().getRootView();
                              getEngine(root).ensureFloatingCapsule();
                          }
                      }
                  });

                  XposedHelpers.findAndHookMethod(View.class, "dispatchTouchEvent", MotionEvent.class, new XC_MethodHook() {
                      @Override
                      protected void beforeHookedMethod(MethodHookParam param) {
                          View view = (View) param.thisObject;
                          if (view != view.getRootView()) return;
                          MotionEvent event = (MotionEvent) param.args[0];
                          if ((event.getSource() & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                              boolean consumed = getEngine(view).onRootTouchEvent(event);
                              if (consumed) {
                                  param.setResult(true);
                              }
                          }
                      }
                  });
              }
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/example/touchburst/MainActivity.java
          package com.example.touchburst;

          import android.app.Activity;
          import android.graphics.Canvas;
          import android.graphics.Color;
          import android.graphics.Paint;
          import android.graphics.PointF;
          import android.graphics.drawable.GradientDrawable;
          import android.os.Bundle;
          import android.view.Gravity;
          import android.view.MotionEvent;
          import android.view.View;
          import android.widget.Button;
          import android.widget.LinearLayout;
          import android.widget.TextView;
          import java.util.LinkedList;

          public class MainActivity extends Activity {
              private TouchBurstEngine localEngine;
              private int clickCount = 0;
              private final LinkedList<Long> clickTimestamps = new LinkedList<>();
              private final LinkedList<PointF> clickPoints = new LinkedList<>();

              public static boolean isLSPosedModuleActive() {
                  return false;
              }

              @Override
              protected void onCreate(Bundle savedInstanceState) {
                  super.onCreate(savedInstanceState);

                  final View decor = getWindow().getDecorView();
                  localEngine = new TouchBurstEngine(decor);
                  localEngine.ensureFloatingCapsule();

                  LinearLayout root = new LinearLayout(this);
                  root.setOrientation(LinearLayout.VERTICAL);
                  root.setBackgroundColor(Color.parseColor("#F5F7FA"));
                  root.setPadding(40, 90, 40, 40);

                  TextView statusCard = new TextView(this);
                  boolean active = isLSPosedModuleActive();
                  statusCard.setText(active
                      ? "✅ LSPosed 模块已激活！\n去 LSPosed 勾选目标应用/游戏并重启它，右上角即会出现「连点:开 | ⚙」悬浮球。"
                      : "⚠️ 当前处于「应用内本地预览模式」\n若要在其他软件/游戏中使用：请打开 LSPosed -> 点击「按住连点器」-> 开启模块并勾选目标软件 -> 强行停止并重开目标软件。");
                  statusCard.setTextColor(Color.WHITE);
                  statusCard.setTextSize(13f);
                  statusCard.setPadding(36, 32, 36, 32);
                  GradientDrawable cardBg = new GradientDrawable();
                  cardBg.setCornerRadius(28f);
                  cardBg.setColor(Color.parseColor(active ? "#2E7D32" : "#EF6C00"));
                  statusCard.setBackground(cardBg);
                  root.addView(statusCard);

                  Button cfgBtn = new Button(this);
                  cfgBtn.setText("⚙ 打开连点速度与随机偏移设置");
                  cfgBtn.setOnClickListener(new View.OnClickListener() {
                      @Override
                      public void onClick(View v) {
                          localEngine.showConfigDialog();
                      }
                  });
                  LinearLayout.LayoutParams btnLp = new LinearLayout.LayoutParams(
                      LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT);
                  btnLp.topMargin = 28;
                  root.addView(cfgBtn, btnLp);

                  final TextView statsView = new TextView(this);
                  statsView.setText("累计触发点击: 0 次   |   实时频率: 0 CPS");
                  statsView.setTextSize(15f);
                  statsView.setTextColor(Color.parseColor("#1A237E"));
                  statsView.setGravity(Gravity.CENTER);
                  statsView.setPadding(0, 28, 0, 20);
                  root.addView(statsView);

                  final View testPad = new View(this) {
                      private final Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
                      @Override
                      protected void onDraw(Canvas canvas) {
                          super.onDraw(canvas);
                          paint.setColor(Color.parseColor("#E3F2FD"));
                          canvas.drawRoundRect(0, 0, getWidth(), getHeight(), 36f, 36f, paint);

                          paint.setColor(Color.parseColor("#1565C0"));
                          paint.setTextSize(36f);
                          paint.setTextAlign(Paint.Align.CENTER);
                          canvas.drawText("👇 按住此区域不放测试连点与随机微偏移 👇", getWidth() / 2f, 90f, paint);
                          paint.setTextSize(26f);
                          paint.setColor(Color.parseColor("#546E7A"));
                          canvas.drawText("（松手即停，红点代表每次随机偏移的实际落点）", getWidth() / 2f, 140f, paint);

                          for (int i = 0; i < clickPoints.size(); i++) {
                              PointF p = clickPoints.get(i);
                              paint.setColor(i == clickPoints.size() - 1 ? Color.RED : Color.parseColor("#66E53935"));
                              canvas.drawCircle(p.x, p.y, i == clickPoints.size() - 1 ? 14f : 8f, paint);
                          }
                      }
                  };

                  testPad.setOnTouchListener(new View.OnTouchListener() {
                      @Override
                      public boolean onTouch(View v, MotionEvent event) {
                          if (event.getActionMasked() == MotionEvent.ACTION_UP) {
                              clickCount++;
                              long now = System.currentTimeMillis();
                              clickTimestamps.addLast(now);
                              while (!clickTimestamps.isEmpty() && now - clickTimestamps.getFirst() > 1000) {
                                  clickTimestamps.removeFirst();
                              }
                              clickPoints.addLast(new PointF(event.getX(), event.getY()));
                              if (clickPoints.size() > 35) clickPoints.removeFirst();

                              statsView.setText("累计触发点击: " + clickCount + " 次   |   实时频率: " + clickTimestamps.size() + " CPS"
                                  + "\n最新随机坐标: (" + Math.round(event.getX()) + ", " + Math.round(event.getY()) + ")");
                              testPad.invalidate();
                          }
                          return true;
                      }
                  });

                  LinearLayout.LayoutParams padLp = new LinearLayout.LayoutParams(
                      LinearLayout.LayoutParams.MATCH_PARENT, 0, 1f);
                  root.addView(testPad, padLp);

                  setContentView(root);
              }

              @Override
              public boolean dispatchTouchEvent(MotionEvent ev) {
                  if (!isLSPosedModuleActive() && localEngine != null) {
                      if (localEngine.onRootTouchEvent(ev)) {
                          return true;
                      }
                  }
                  return super.dispatchTouchEvent(ev);
              }
          }
          EOF

          gradle assembleDebug --no-daemon

      - name: Upload Compiled APK
        uses: actions/upload-artifact@v4
        with:
          name: TouchBurst-v2-APK
          path: app/build/outputs/apk/debug/app-debug.apk
