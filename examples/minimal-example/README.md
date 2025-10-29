# Minimal Azure Authentication Example

This is the **absolute minimum** code needed to add Azure authentication to an Android app using MSAL.

## What This Example Shows

- Minimal viable implementation (less than 100 lines)
- Single Activity app with sign-in functionality
- No complex UI, just essential functionality
- Perfect starting point for understanding MSAL basics

## Files You Need

### 1. MainActivity.java (The only Java file needed)

```java
package com.example.minimal;

import android.os.Bundle;
import android.util.Log;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;

import com.microsoft.identity.client.*;

import java.util.Collections;
import java.util.List;

public class MainActivity extends AppCompatActivity {
    private IMultipleAccountPublicClientApplication mPCA;
    private static final List<String> SCOPES = Collections.singletonList("User.Read");
    private static final String TAG = "MinimalMSAL";
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        TextView statusText = findViewById(R.id.status);
        Button signInBtn = findViewById(R.id.sign_in);
        
        // Initialize MSAL
        PublicClientApplication.createMultipleAccountPublicClientApplication(
            getApplicationContext(),
            R.raw.auth_config,
            new IPublicClientApplication.IMultipleAccountApplicationCreatedListener() {
                @Override
                public void onCreated(IMultipleAccountPublicClientApplication app) {
                    mPCA = app;
                    runOnUiThread(() -> statusText.setText("Ready"));
                    Log.d(TAG, "MSAL initialized");
                }
                
                @Override
                public void onError(MsalException exception) {
                    runOnUiThread(() -> statusText.setText("Init failed: " + exception.getMessage()));
                    Log.e(TAG, "Init failed", exception);
                }
            }
        );
        
        // Sign in when button clicked
        signInBtn.setOnClickListener(v -> {
            if (mPCA == null) return;
            
            AcquireTokenParameters params = new AcquireTokenParameters.Builder()
                .startAuthorizationFromActivity(MainActivity.this)
                .withScopes(SCOPES)
                .withCallback(new AuthenticationCallback() {
                    @Override
                    public void onSuccess(IAuthenticationResult result) {
                        String user = result.getAccount().getUsername();
                        runOnUiThread(() -> {
                            statusText.setText("Signed in: " + user);
                            Toast.makeText(MainActivity.this, "Welcome!", Toast.LENGTH_SHORT).show();
                        });
                        Log.d(TAG, "Sign in successful: " + user);
                    }
                    
                    @Override
                    public void onError(MsalException exception) {
                        runOnUiThread(() -> statusText.setText("Sign in failed"));
                        Log.e(TAG, "Sign in failed", exception);
                    }
                    
                    @Override
                    public void onCancel() {
                        runOnUiThread(() -> statusText.setText("Sign in cancelled"));
                        Log.d(TAG, "Sign in cancelled");
                    }
                })
                .build();
            
            mPCA.acquireToken(params);
        });
    }
}
```

### 2. activity_main.xml (Simple layout)

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="32dp">
    
    <TextView
        android:id="@+id/status"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Initializing..."
        android:textSize="16sp"
        android:layout_marginBottom="24dp"/>
    
    <Button
        android:id="@+id/sign_in"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Sign In"/>
</LinearLayout>
```

### 3. auth_config.json (res/raw/auth_config.json)

```json
{
  "client_id": "YOUR_CLIENT_ID",
  "redirect_uri": "msauth://com.example.minimal/YOUR_HASH_URL_ENCODED",
  "broker_redirect_uri_registered": true,
  "authorities": [
    {
      "type": "AAD",
      "audience": {
        "type": "AzureADandPersonalMicrosoftAccount",
        "tenant_id": "common"
      }
    }
  ]
}
```

### 4. AndroidManifest.xml (Add these parts)

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.minimal">

    <!-- Add these permissions -->
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>

    <application
        android:label="Minimal MSAL"
        android:theme="@style/Theme.AppCompat.Light">
        
        <activity android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        
        <!-- Add BrowserTabActivity for MSAL -->
        <activity
            android:name="com.microsoft.identity.client.BrowserTabActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data
                    android:scheme="msauth"
                    android:host="com.example.minimal"
                    android:path="/YOUR_HASH_NOT_ENCODED"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

### 5. build.gradle (app level)

```gradle
android {
    compileSdk 35
    
    defaultConfig {
        applicationId "com.example.minimal"
        minSdk 24
        targetSdk 35
        versionCode 1
        versionName "1.0"
    }
}

dependencies {
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.microsoft.identity.client:msal:7.+'
}
```

### 6. gradle.properties

```properties
android.useAndroidX=true
android.enableJetifier=true
```

## Setup Steps

1. **Create Azure App Registration:**
   - Go to [Azure Portal](https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade)
   - Click "New registration"
   - Name it and select account types
   - Note the Client ID

2. **Get Your Signature Hash:**
   ```bash
   keytool -exportcert -alias androiddebugkey -keystore ~/.android/debug.keystore | openssl dgst -sha1 -binary | openssl base64
   ```
   Password: `android`

3. **Update Configuration:**
   - In `auth_config.json`: Use URL-encoded hash (`/` → `%2F`, `+` → `%2B`, `=` → `%3D`)
   - In `AndroidManifest.xml`: Use plain hash (not encoded)

4. **Configure Azure Portal:**
   - Add Android platform in your app registration
   - Enter package name: `com.example.minimal`
   - Enter signature hash (plain, not encoded)

5. **Run the App!**

## That's It!

This is literally all the code you need for Azure authentication in Android:
- 1 Activity (about 70 lines)
- 1 Layout XML
- 1 Config JSON
- Manifest updates
- Gradle dependencies

Total: ~150 lines of configuration and code.

## What This Example Does

1. **Initializes MSAL** when app starts
2. **Signs in user** when button clicked
3. **Displays username** after successful sign-in
4. **Handles errors** gracefully

## What's Missing (Intentionally)

For a complete app, you'd also need:
- Token caching and refresh
- Sign out functionality
- API calls with access token
- Multiple account handling
- Better error handling
- Loading states

See the [full examples](../) for production-ready implementations.

## Next Steps

Once you understand this minimal example:
- Add sign out: See [remove_account.java](../../snippets/remove_account.java)
- Call Microsoft Graph: See [hello-msal-multiple-account](../hello-msal-multiple-account/)
- Handle multiple accounts: See the full examples
- Silent token acquisition: See [acquire_token_silent.java](../../snippets/acquire_token_silent.java)

## Troubleshooting

**"Redirect URI mismatch"**
- Signature hash must match between Azure Portal and your keystore
- Check URL encoding (encoded in auth_config.json, plain in AndroidManifest.xml)

**"MSAL not initialized"**
- Check auth_config.json is in `res/raw/` folder
- Verify client_id is correct

**Build errors**
- Sync Gradle
- Check AndroidX is enabled in gradle.properties
- Ensure MSAL dependency is added

---

This minimal example is designed to teach the basics. For production apps, use the complete examples with proper error handling, token management, and UI.
