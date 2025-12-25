# MINI-PROJECT
package com.finansmart.app;

```
import static android.content.ContentValues.TAG;
import androidx.activity.EdgeToEdge;
import androidx.appcompat.app.AppCompatActivity;

import android.Manifest;
import android.app.Activity;
import android.app.AlarmManager;
import android.app.Application;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.app.PendingIntent;
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.os.Bundle;
import android.os.PowerManager;
import android.provider.Settings;
import android.provider.Telephony;
import android.text.TextUtils;
import android.util.Log;
import android.view.MenuItem;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.core.app.ActivityCompat;
import androidx.core.app.NotificationManagerCompat;
import androidx.core.content.ContextCompat;

import androidx.work.OneTimeWorkRequest;
import androidx.work.PeriodicWorkRequest;
import androidx.work.WorkManager;

import java.util.Calendar;
import java.util.concurrent.TimeUnit;

import com.google.android.material.bottomnavigation.BottomNavigationView;
import com.google.android.material.navigation.NavigationBarView;

public class MainActivity extends AppCompatActivity {
    DatabaseHelper myDB;
    BottomNavigationView bottomNavigationView;
    HomeFragment homeFragment = new HomeFragment();
    AnalysisFragment analysisFragment = new AnalysisFragment();
    ExportFragment exportFragment = new ExportFragment();
    AssistantFragment assistantFragment = new AssistantFragment();
    private final int RC_Notification = 99;

    private static final int SMS_PERMISSION_CODE = 101;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState); // ✅ Always first
        EdgeToEdge.enable(this);
        setContentView(R.layout.activity_main);

        AppState.isAppInBackground = false;

        System.out.println("Hello Are you Working");

        myDB = new DatabaseHelper(this);
        bottomNavigationView = findViewById(R.id.bottom_navigation);

        // Start foreground service
        startSmsService();

        requestSmsPermission();

        createNotificationChannel();
        scheduleWeeklyAndMonthlyAlarms();

      //  askForNotification();

        // Ask user to grant notification listener permission
        if (!isNotificationServiceEnabled(this)) {
            Intent intent = new Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS);
            startActivity(intent);
        }

        // Ask to become default SMS app (optional, if you're replacing SMS UI)
        if (!getPackageName().equals(Telephony.Sms.getDefaultSmsPackage(this))) {
            Intent smsIntent = new Intent(Telephony.Sms.Intents.ACTION_CHANGE_DEFAULT);
            smsIntent.putExtra(Telephony.Sms.Intents.EXTRA_PACKAGE_NAME, getPackageName());
            startActivity(smsIntent);
        }
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.RECEIVE_SMS)
                != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this,
                    new String[]{Manifest.permission.RECEIVE_SMS, Manifest.permission.READ_SMS},
                    1);
        }
        ComponentName componentName = new ComponentName(this, SmsCheckingService.class);
        getPackageManager().setComponentEnabledSetting(componentName,
                PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
                PackageManager.DONT_KILL_APP);
        getPackageManager().setComponentEnabledSetting(componentName,
                PackageManager.COMPONENT_ENABLED_STATE_ENABLED,
                PackageManager.DONT_KILL_APP);

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            Intent intent = new Intent();
            String packageName = getPackageName();
            PowerManager pm = (PowerManager) getSystemService(POWER_SERVICE);
            if (!pm.isIgnoringBatteryOptimizations(packageName)) {
                intent.setAction(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS);
                intent.setData(Uri.parse("package:" + packageName));
                startActivity(intent);
            }
        }
        ignoreBatteryOptimizations();


        getSupportFragmentManager().beginTransaction().replace(R.id.container, homeFragment).commit();
        bottomNavigationView.setOnItemSelectedListener(new NavigationBarView.OnItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(MenuItem item) {
                int itemId = item.getItemId();
                if (itemId == R.id.home) {
                    getSupportFragmentManager().beginTransaction().replace(R.id.container, homeFragment).commit();
                    return true;
                } else if (itemId == R.id.analysis) {
                    getSupportFragmentManager().beginTransaction().replace(R.id.container, analysisFragment).commit();
                    return true;
                } else if (itemId == R.id.export) {
                    getSupportFragmentManager().beginTransaction().replace(R.id.container, exportFragment).commit();
                    return true;
                } else if (itemId == R.id.assistant){
                    getSupportFragmentManager().beginTransaction().replace(R.id.container, assistantFragment).commit();
                    return true;
                }
                return false;
            }
        });

        // Register activity lifecycle callbacks
        getApplication().registerActivityLifecycleCallbacks(new Application.ActivityLifecycleCallbacks() {
            @Override
            public void onActivityResumed(Activity activity) {
                // Log when the app comes to the foreground
//                startService();
            }

            @Override
            public void onActivityPaused(Activity activity) {
                // Log when the app goes to the background
                AppState.isAppInBackground = true;
            }

            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {}

            @Override
            public void onActivityStarted(Activity activity) {}

            @Override
            public void onActivityStopped(Activity activity) {
                AppState.isAppInBackground = true;
            }

            @Override
            public void onActivitySaveInstanceState(Activity activity, Bundle outState) {}

            @Override
            public void onActivityDestroyed(Activity activity) {}
        });



    }

    private void askForNotification(){
        if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU){
            requestPermissions(new String[]{Manifest.permission.POST_NOTIFICATIONS},RC_Notification);
        }
    }



    public void startSmsService() {
        Intent serviceIntent = new Intent(this, SmsCheckingService.class);
        ContextCompat.startForegroundService(this, serviceIntent);
    }

    private void requestSmsPermission() {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.RECEIVE_SMS) != PackageManager.PERMISSION_GRANTED ||
                ContextCompat.checkSelfPermission(this, Manifest.permission.READ_SMS) != PackageManager.PERMISSION_GRANTED) {

            ActivityCompat.requestPermissions(this,
                    new String[]{Manifest.permission.RECEIVE_SMS, Manifest.permission.READ_SMS},
                    SMS_PERMISSION_CODE);
        }
    }

    private void createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(
                    SmsCheckingService.SMS_CHANNEL_ID,
                    "SMS Transaction Alerts",
                    NotificationManager.IMPORTANCE_HIGH
            );
            channel.setDescription("Handles transaction notification replies");
            NotificationManager manager = getSystemService(NotificationManager.class);
            if (manager != null) {
                manager.createNotificationChannel(channel);
            }
        }
    }

    private boolean isNotificationServiceEnabled(Context context) {
        String pkgName = context.getPackageName();
        final String flat = Settings.Secure.getString(context.getContentResolver(),
                "enabled_notification_listeners");
        if (!TextUtils.isEmpty(flat)) {
            String[] names = flat.split(":");
            for (String name : names) {
                ComponentName cn = ComponentName.unflattenFromString(name);
                if (cn != null && TextUtils.equals(pkgName, cn.getPackageName())) {
                    return true;
                }
            }
        }
        return false;
    }
    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == 1 && grantResults.length > 0
                && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            Toast.makeText(this, "SMS Permission Granted", Toast.LENGTH_SHORT).show();
        }
    }
    private void ignoreBatteryOptimizations() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            PowerManager pm = (PowerManager) getSystemService(Context.POWER_SERVICE);
            if (pm != null && !pm.isIgnoringBatteryOptimizations(getPackageName())) {
                Intent intent = new Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS);
                intent.setData(Uri.parse("package:" + getPackageName()));
                startActivity(intent);
            }
        }
    }

    private void scheduleWeeklyAndMonthlyAlarms() {
        AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);

        // Weekly Alarm
        Intent weeklyIntent = new Intent(this, WeeklyExpenseReceiver.class);
        PendingIntent weeklyPendingIntent = PendingIntent.getBroadcast(this, 1, weeklyIntent, PendingIntent.FLAG_IMMUTABLE);

        Calendar calendarWeekly = Calendar.getInstance();
        calendarWeekly.set(Calendar.DAY_OF_WEEK, Calendar.SUNDAY);
        calendarWeekly.set(Calendar.HOUR_OF_DAY, 20);  // 8 PM
        calendarWeekly.set(Calendar.MINUTE, 0);
        calendarWeekly.set(Calendar.SECOND, 0);
        if (calendarWeekly.before(Calendar.getInstance())) {
            calendarWeekly.add(Calendar.WEEK_OF_YEAR, 1);
        }

        alarmManager.setRepeating(AlarmManager.RTC_WAKEUP,
                calendarWeekly.getTimeInMillis(),
                AlarmManager.INTERVAL_DAY * 7,
                weeklyPendingIntent);

        // Monthly Alarm
        Intent monthlyIntent = new Intent(this, MonthlyExpenseReceiver.class);
        PendingIntent monthlyPendingIntent = PendingIntent.getBroadcast(this, 2, monthlyIntent, PendingIntent.FLAG_IMMUTABLE);

        Calendar calendarMonthly = Calendar.getInstance();
        calendarMonthly.set(Calendar.DAY_OF_MONTH, 1);
        calendarMonthly.set(Calendar.HOUR_OF_DAY, 10);  // 10 AM
        calendarMonthly.set(Calendar.MINUTE, 0);
        calendarMonthly.set(Calendar.SECOND, 0);
        if (calendarMonthly.before(Calendar.getInstance())) {
            calendarMonthly.add(Calendar.MONTH, 1);
        }

        alarmManager.setRepeating(AlarmManager.RTC_WAKEUP,
                calendarMonthly.getTimeInMillis(),
                AlarmManager.INTERVAL_DAY * 30,
                monthlyPendingIntent);
    }

}
```
