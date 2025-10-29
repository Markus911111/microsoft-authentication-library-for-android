# Quick Start: Using Azure Authentication in Android Apps

This guide will help you quickly integrate Azure authentication into your Android application using the Microsoft Authentication Library (MSAL) for Android.

## Prerequisites

- Android Studio (latest version recommended)
- Android SDK with minimum API level 24
- An Azure account (create one for free at [azure.microsoft.com](https://azure.microsoft.com))
- Basic knowledge of Android development

## Step 1: Register Your App in Azure Portal

1. Go to the [Azure Portal App Registrations](https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade)
2. Click **New registration**
3. Enter a name for your application
4. Select **Accounts in any organizational directory and personal Microsoft accounts**
5. Click **Register**
6. Note down the **Application (client) ID** - you'll need this later
7. Click **Add a platform** under **Platform configurations**
8. Select **Android**
9. Enter your package name (e.g., `com.example.myapp`)
10. Generate and enter your signature hash (see below)
11. Click **Configure**

### Generating Your Signature Hash

Use the following command in your terminal (requires Java keytool):

```bash
# For debug keystore (development)
keytool -exportcert -alias androiddebugkey -keystore ~/.android/debug.keystore | openssl dgst -sha1 -binary | openssl base64

# Password for debug keystore is typically: android
```

For release builds, use your release keystore:
```bash
keytool -exportcert -alias YOUR_RELEASE_KEY_ALIAS -keystore YOUR_RELEASE_KEYSTORE_PATH | openssl dgst -sha1 -binary | openssl base64
```

## Step 2: Set Up Your Android Project

### Add Dependencies

Add to your **project-level** `build.gradle`:

```gradle
buildscript {
    repositories {
        google()
        mavenCentral()
        maven { 
            url 'https://pkgs.dev.azure.com/MicrosoftDeviceSDK/DuoSDK-Public/_packaging/Duo-SDK-Feed/maven/v1' 
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.1.0' // or newer
    }
}
```

Add to your **app-level** `build.gradle`:

```gradle
android {
    compileSdk 35
    
    defaultConfig {
        minSdk 24
        targetSdk 35
        // ... other config
    }
}

dependencies {
    implementation 'com.microsoft.identity.client:msal:7.+'
    // ... other dependencies
}
```

Add to your `gradle.properties`:

```properties
# Required for MSAL
android.useAndroidX=true
android.enableJetifier=true
```

## Step 3: Configure MSAL

### Create Configuration File

Create `app/src/main/res/raw/auth_config.json`:

```json
{
  "client_id": "YOUR_CLIENT_ID_FROM_AZURE",
  "redirect_uri": "msauth://YOUR_PACKAGE_NAME/YOUR_SIGNATURE_HASH_URL_ENCODED",
  "broker_redirect_uri_registered": true,
  "account_mode": "MULTIPLE",
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

**Important Notes:**
- Replace `YOUR_CLIENT_ID_FROM_AZURE` with your Application ID from Azure Portal
- Replace `YOUR_PACKAGE_NAME` with your app's package name (e.g., `com.example.myapp`)
- Replace `YOUR_SIGNATURE_HASH_URL_ENCODED` with your URL-encoded signature hash
  - URL encode special characters: `+` becomes `%2B`, `=` becomes `%3D`, `/` becomes `%2F`
  - Example: If your hash is `Ab/Cd+Ef=`, it becomes `Ab%2FCd%2BEf%3D`

### Update AndroidManifest.xml

Add permissions:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

Add BrowserTabActivity inside `<application>` tag:

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
            android:path="/YOUR_SIGNATURE_HASH_NOT_URL_ENCODED" />
    </intent-filter>
</activity>
```

**Important:** In AndroidManifest.xml, the signature hash is **NOT** URL encoded (use `/` and `=` directly).

## Step 4: Initialize MSAL in Your Activity

```java
import com.microsoft.identity.client.*;
import java.util.List;
import java.util.Collections;

public class MainActivity extends AppCompatActivity {
    private IMultipleAccountPublicClientApplication mPCA;
    private static final List<String> SCOPES = Collections.singletonList("User.Read");
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        // Initialize MSAL
        PublicClientApplication.createMultipleAccountPublicClientApplication(
            getApplicationContext(),
            R.raw.auth_config,
            new IPublicClientApplication.IMultipleAccountApplicationCreatedListener() {
                @Override
                public void onCreated(IMultipleAccountPublicClientApplication application) {
                    mPCA = application;
                    // MSAL is ready to use
                }
                
                @Override
                public void onError(MsalException exception) {
                    // Handle initialization error
                    Log.e("MSAL", "Failed to initialize", exception);
                }
            }
        );
    }
}
```

## Step 5: Implement Sign-In

```java
public void signIn() {
    if (mPCA == null) {
        return; // MSAL not initialized yet
    }
    
    AcquireTokenParameters parameters = new AcquireTokenParameters.Builder()
        .startAuthorizationFromActivity(this)
        .withScopes(SCOPES)
        .withCallback(new AuthenticationCallback() {
            @Override
            public void onSuccess(IAuthenticationResult result) {
                // User signed in successfully
                String accessToken = result.getAccessToken();
                IAccount account = result.getAccount();
                
                runOnUiThread(() -> {
                    // Update UI
                    Toast.makeText(MainActivity.this, 
                        "Signed in as: " + account.getUsername(), 
                        Toast.LENGTH_SHORT).show();
                });
            }
            
            @Override
            public void onError(MsalException exception) {
                // Handle sign-in error
                Log.e("MSAL", "Sign in failed", exception);
            }
            
            @Override
            public void onCancel() {
                // User canceled sign-in
            }
        })
        .build();
    
    mPCA.acquireToken(parameters);
}
```

## Step 6: Get Access Token Silently

After initial sign-in, get tokens silently (without user interaction):

```java
public void getTokenSilently(IAccount account) {
    if (mPCA == null || account == null) {
        return;
    }
    
    AcquireTokenSilentParameters parameters = new AcquireTokenSilentParameters.Builder()
        .forAccount(account)
        .fromAuthority(account.getAuthority())
        .withScopes(SCOPES)
        .withCallback(new AuthenticationCallback() {
            @Override
            public void onSuccess(IAuthenticationResult result) {
                // Got token successfully
                String accessToken = result.getAccessToken();
                // Use the token to call your API
            }
            
            @Override
            public void onError(MsalException exception) {
                if (exception instanceof MsalUiRequiredException) {
                    // Token expired or consent required, need interactive auth
                    signIn();
                }
            }
            
            @Override
            public void onCancel() {
                // Not called for silent requests
            }
        })
        .build();
    
    mPCA.acquireTokenSilentAsync(parameters);
}
```

## Step 7: Sign Out

```java
public void signOut(IAccount account) {
    if (mPCA == null || account == null) {
        return;
    }
    
    mPCA.removeAccount(account, new IMultipleAccountPublicClientApplication.RemoveAccountCallback() {
        @Override
        public void onRemoved() {
            runOnUiThread(() -> {
                Toast.makeText(MainActivity.this, "Signed out", Toast.LENGTH_SHORT).show();
            });
        }
        
        @Override
        public void onError(MsalException exception) {
            Log.e("MSAL", "Sign out failed", exception);
        }
    });
}
```

## Using the Access Token

Once you have an access token, you can call Azure services or Microsoft Graph:

```java
// Example: Call Microsoft Graph to get user profile
private void callGraphAPI(String accessToken) {
    // Use HTTP client to call Graph API
    // GET https://graph.microsoft.com/v1.0/me
    // Header: Authorization: Bearer {accessToken}
    
    // Example with OkHttp:
    OkHttpClient client = new OkHttpClient();
    Request request = new Request.Builder()
        .url("https://graph.microsoft.com/v1.0/me")
        .addHeader("Authorization", "Bearer " + accessToken)
        .build();
    
    client.newCall(request).enqueue(new Callback() {
        @Override
        public void onResponse(Call call, Response response) throws IOException {
            String jsonData = response.body().string();
            // Parse and use the data
        }
        
        @Override
        public void onFailure(Call call, IOException e) {
            // Handle error
        }
    });
}
```

## Complete Example Apps

For complete, working examples, check out:

- **Multiple Account Mode:** `examples/hello-msal-multiple-account/`
- **Single Account Mode:** `examples/hello-msal-single-account/`

These examples include:
- Complete UI implementation
- Token acquisition and refresh
- Microsoft Graph API calls
- Error handling
- Account management

## Common Issues and Solutions

### Issue: BrowserTabActivity not found
**Solution:** Make sure you've added the MSAL dependency and synced your Gradle files.

### Issue: Redirect URI mismatch
**Solution:** 
- Verify the signature hash is correct
- In `auth_config.json`, the hash must be URL encoded
- In `AndroidManifest.xml`, the hash must NOT be URL encoded
- Ensure package name matches exactly

### Issue: Authentication fails silently
**Solution:** 
- Check Logcat for error messages
- Verify your client_id is correct
- Ensure internet permission is granted
- Check that the redirect URI is registered in Azure Portal

### Issue: Build errors about AndroidX
**Solution:** Add these lines to `gradle.properties`:
```properties
android.useAndroidX=true
android.enableJetifier=true
```

## Next Steps

- **Read the full documentation:** [README.md](README.md)
- **Explore code snippets:** `snippets/` directory contains Java and Kotlin examples
- **Learn about configuration options:** [auth_config.template.json](auth_config.template.json)
- **Understand MSAL concepts:** Check out the [official Microsoft documentation](https://learn.microsoft.com/en-us/azure/active-directory/develop/msal-overview)

## Additional Resources

- [Azure Portal](https://portal.azure.com)
- [Microsoft Identity Platform Documentation](https://learn.microsoft.com/en-us/azure/active-directory/develop/)
- [Microsoft Graph API](https://developer.microsoft.com/en-us/graph)
- [MSAL Android Wiki](https://github.com/AzureAD/microsoft-authentication-library-for-android/wiki)

## Support

If you encounter issues:
1. Check the [FAQ](https://github.com/AzureAD/microsoft-authentication-library-for-android/wiki/MSAL-FAQ)
2. Search existing [GitHub Issues](https://github.com/AzureAD/microsoft-authentication-library-for-android/issues)
3. Review the [troubleshooting guide](https://learn.microsoft.com/en-us/azure/active-directory/develop/msal-error-handling-android)

---

**Congratulations!** You now have Azure authentication integrated into your Android app. Users can sign in with their Microsoft accounts and you can access Azure services securely.
