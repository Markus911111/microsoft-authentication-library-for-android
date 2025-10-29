# Getting Started with Azure Authentication for Android

This guide provides the **fastest path** to adding Azure/Microsoft authentication to your Android app.

## What You'll Build

A simple Android app that allows users to:
- Sign in with their Microsoft account (work, school, or personal)
- View their profile information
- Sign out

**Time to complete:** ~15 minutes

## Before You Start

You need:
- [ ] Android Studio installed
- [ ] An Azure account ([create free](https://azure.microsoft.com/free/))
- [ ] A new or existing Android project (minSdk 24+)

## 3-Minute Setup

### 1. Add MSAL to Your Project

**In your app's `build.gradle`:**

```gradle
dependencies {
    implementation 'com.microsoft.identity.client:msal:7.+'
}
```

**In your project's `build.gradle`, add the repository:**

```gradle
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { 
            url 'https://pkgs.dev.azure.com/MicrosoftDeviceSDK/DuoSDK-Public/_packaging/Duo-SDK-Feed/maven/v1' 
        }
    }
}
```

**In `gradle.properties`:**

```properties
android.useAndroidX=true
android.enableJetifier=true
```

### 2. Register Your App in Azure

1. Go to [Azure Portal - App Registrations](https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade)
2. Click **+ New registration**
3. Name: `MyAndroidApp` (or any name you prefer)
4. Supported account types: **Accounts in any organizational directory and personal Microsoft accounts**
5. Click **Register**
6. **Copy the Application (client) ID** - you'll need it in step 3

### 3. Configure Your App

**Create `app/src/main/res/raw/auth_config.json`:**

```json
{
  "client_id": "PASTE_YOUR_CLIENT_ID_HERE",
  "redirect_uri": "msauth://YOUR_PACKAGE_NAME/YOUR_SIGNATURE_HASH",
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

Replace:
- `PASTE_YOUR_CLIENT_ID_HERE` with the Application ID you copied
- `YOUR_PACKAGE_NAME` with your app's package (e.g., `com.example.myapp`)
- `YOUR_SIGNATURE_HASH` - see below ⬇️

**Get Your Signature Hash:**

Run this in your terminal:

```bash
# On Mac/Linux:
keytool -exportcert -alias androiddebugkey -keystore ~/.android/debug.keystore | openssl dgst -sha1 -binary | openssl base64

# On Windows (PowerShell):
keytool -exportcert -alias androiddebugkey -keystore %HOMEPATH%\.android\debug.keystore | openssl dgst -sha1 -binary | openssl base64
```

Password is: `android`

**Important:** In `auth_config.json`, URL-encode the hash:
- Replace `+` with `%2B`
- Replace `/` with `%2F`
- Replace `=` with `%3D`

Example: `Xy/Ab+Cd=` becomes `Xy%2FAb%2BCd%3D`

**Update `AndroidManifest.xml`:**

Add permissions before the `<application>` tag:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

Add inside the `<application>` tag:

```xml
<activity
    android:name="com.microsoft.identity.client.BrowserTabActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="msauth"
            android:host="YOUR_PACKAGE_NAME"
            android:path="/YOUR_SIGNATURE_HASH_NOT_ENCODED" />
    </intent-filter>
</activity>
```

Replace:
- `YOUR_PACKAGE_NAME` with your package
- `YOUR_SIGNATURE_HASH_NOT_ENCODED` with your hash (NOT URL encoded, keep `/`, `+`, `=` as-is)

### 4. Add Authentication Code

**Create a simple activity:**

```java
package com.example.myapp; // Change to your package

import android.os.Bundle;
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
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        Button signInButton = findViewById(R.id.sign_in_button);
        TextView statusText = findViewById(R.id.status_text);
        
        // Initialize MSAL
        PublicClientApplication.createMultipleAccountPublicClientApplication(
            getApplicationContext(),
            R.raw.auth_config,
            new IPublicClientApplication.IMultipleAccountApplicationCreatedListener() {
                @Override
                public void onCreated(IMultipleAccountPublicClientApplication app) {
                    mPCA = app;
                    statusText.setText("Ready to sign in");
                }
                
                @Override
                public void onError(MsalException exception) {
                    statusText.setText("Error: " + exception.getMessage());
                }
            }
        );
        
        // Sign in button click
        signInButton.setOnClickListener(v -> {
            if (mPCA == null) return;
            
            AcquireTokenParameters params = new AcquireTokenParameters.Builder()
                .startAuthorizationFromActivity(MainActivity.this)
                .withScopes(SCOPES)
                .withCallback(new AuthenticationCallback() {
                    @Override
                    public void onSuccess(IAuthenticationResult result) {
                        runOnUiThread(() -> {
                            String username = result.getAccount().getUsername();
                            statusText.setText("Signed in as: " + username);
                            Toast.makeText(MainActivity.this, 
                                "Welcome, " + username, 
                                Toast.LENGTH_LONG).show();
                        });
                    }
                    
                    @Override
                    public void onError(MsalException exception) {
                        runOnUiThread(() -> 
                            statusText.setText("Sign in failed: " + exception.getMessage())
                        );
                    }
                    
                    @Override
                    public void onCancel() {
                        runOnUiThread(() -> statusText.setText("Sign in cancelled"));
                    }
                })
                .build();
            
            mPCA.acquireToken(params);
        });
    }
}
```

**Create `res/layout/activity_main.xml`:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="24dp">
    
    <TextView
        android:id="@+id/status_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Initializing..."
        android:textSize="18sp"
        android:layout_marginBottom="32dp"/>
    
    <Button
        android:id="@+id/sign_in_button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Sign In with Microsoft"/>
</LinearLayout>
```

### 5. Complete Azure Setup

Back in Azure Portal:

1. Click **Authentication** in your app registration
2. Click **+ Add a platform**
3. Select **Android**
4. Enter your **Package name** (e.g., `com.example.myapp`)
5. Enter your **Signature hash** (the NON-URL-encoded version)
6. Click **Configure**

## Run Your App!

1. Sync Gradle files
2. Run on emulator or device
3. Click "Sign In with Microsoft"
4. Sign in with any Microsoft account
5. See your username displayed!

## What's Next?

### Call Microsoft Graph API

Once signed in, use the access token to call Microsoft Graph:

```java
@Override
public void onSuccess(IAuthenticationResult result) {
    String accessToken = result.getAccessToken();
    
    // Call Graph API to get user info
    new Thread(() -> {
        try {
            URL url = new URL("https://graph.microsoft.com/v1.0/me");
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setRequestProperty("Authorization", "Bearer " + accessToken);
            
            // Read response...
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(conn.getInputStream()));
            StringBuilder response = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                response.append(line);
            }
            
            runOnUiThread(() -> {
                // Parse JSON and display user info
                Toast.makeText(this, response.toString(), Toast.LENGTH_LONG).show();
            });
        } catch (Exception e) {
            e.printStackTrace();
        }
    }).start();
}
```

### Add Sign Out

```java
// Get signed-in accounts
mPCA.getAccounts(new IPublicClientApplication.LoadAccountsCallback() {
    @Override
    public void onTaskCompleted(List<IAccount> accounts) {
        if (!accounts.isEmpty()) {
            // Sign out first account
            mPCA.removeAccount(accounts.get(0), new IMultipleAccountPublicClientApplication.RemoveAccountCallback() {
                @Override
                public void onRemoved() {
                    runOnUiThread(() -> 
                        Toast.makeText(MainActivity.this, "Signed out", Toast.LENGTH_SHORT).show()
                    );
                }
                
                @Override
                public void onError(MsalException exception) {
                    // Handle error
                }
            });
        }
    }
    
    @Override
    public void onError(MsalException exception) {
        // Handle error
    }
});
```

### Explore Complete Examples

For production-ready code:
- **Multiple accounts:** `examples/hello-msal-multiple-account/`
- **Single account:** `examples/hello-msal-single-account/`

## Troubleshooting

**"Redirect URI mismatch"**
- Check that signature hash in Azure Portal matches your debug keystore
- Verify URL encoding in `auth_config.json`
- Ensure package name is correct everywhere

**"Failed to initialize"**
- Check `auth_config.json` is in `res/raw/` directory
- Verify JSON syntax (no trailing commas)
- Ensure client_id is correct

**Build errors**
- Make sure `gradle.properties` has AndroidX enabled
- Sync Gradle files
- Clean and rebuild project

**Authentication doesn't start**
- Verify BrowserTabActivity is in AndroidManifest.xml
- Check internet permissions are granted
- Ensure MSAL initialization completed (check `onCreated` was called)

## Learn More

- 📖 [Full Documentation](README.md)
- 🚀 [Detailed Quick Start](QUICKSTART.md)
- 💡 [Code Snippets](snippets/)
- 🔧 [Configuration Reference](auth_config.template.json)
- 📚 [Microsoft Identity Platform Docs](https://learn.microsoft.com/azure/active-directory/develop/)

---

**You're all set!** You've successfully integrated Azure authentication into your Android app. Your users can now sign in with their Microsoft accounts and you can access Microsoft services on their behalf.
