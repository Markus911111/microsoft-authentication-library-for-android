# Azure for Android: Complete Integration Tutorial

This tutorial demonstrates how to build Android applications that integrate with Azure services using the Microsoft Authentication Library (MSAL) for Android.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Common Scenarios](#common-scenarios)
4. [Implementation Guide](#implementation-guide)
5. [Best Practices](#best-practices)
6. [Production Checklist](#production-checklist)

## Overview

MSAL for Android is Microsoft's official library for authenticating users and accessing Azure services from Android applications. It enables:

- **User Authentication**: Sign in with Microsoft work/school accounts or personal Microsoft accounts
- **Secure Token Management**: Automatic token caching and refresh
- **SSO (Single Sign-On)**: Seamless authentication across apps using Microsoft Authenticator
- **Azure Service Access**: Call Microsoft Graph, Azure APIs, and custom APIs
- **Conditional Access**: Support for MFA, device compliance, and other policies

### What is Azure for Android?

Azure for Android refers to integrating Azure Active Directory (AAD) authentication and Azure services into Android applications. With MSAL, you can:

- Authenticate users against Azure AD
- Access Microsoft 365 data (emails, calendars, files)
- Call Microsoft Graph APIs
- Integrate with Azure App Services
- Use Azure Functions from mobile apps
- Access custom APIs protected by Azure AD

## Architecture

```
┌─────────────────┐
│  Android App    │
│  (Your Code)    │
└────────┬────────┘
         │
         │ MSAL Library
         ▼
┌─────────────────┐
│  MSAL for       │◄──────┐
│  Android        │       │ Broker
└────────┬────────┘       │ (Optional)
         │         ┌──────┴────────┐
         │         │ MS Authenticator│
         │         │ Company Portal  │
         │         └─────────────────┘
         │
         │ OAuth 2.0 / OpenID Connect
         ▼
┌─────────────────┐
│  Microsoft      │
│  Identity       │
│  Platform       │
└────────┬────────┘
         │
         │ Tokens
         ▼
┌─────────────────┐
│  Azure Services │
│  - Graph API    │
│  - Your APIs    │
│  - Azure Apps   │
└─────────────────┘
```

## Common Scenarios

### Scenario 1: Employee Portal App

**Use Case**: Corporate app for employees to access company resources

**Features**:
- Sign in with work account only
- Access company SharePoint, OneDrive
- View organizational chart
- Check company calendar

**Configuration**:
```json
{
  "client_id": "YOUR_APP_ID",
  "redirect_uri": "msauth://com.company.portal/HASH",
  "authorities": [
    {
      "type": "AAD",
      "audience": {
        "type": "AzureADMyOrg",
        "tenant_id": "YOUR_TENANT_ID"
      }
    }
  ],
  "account_mode": "SINGLE"
}
```

### Scenario 2: Consumer App with Microsoft Sign-In

**Use Case**: Consumer app offering "Sign in with Microsoft" option

**Features**:
- Sign in with personal Microsoft account
- Access OneDrive for file storage
- Sync across devices

**Configuration**:
```json
{
  "client_id": "YOUR_APP_ID",
  "redirect_uri": "msauth://com.myapp.consumer/HASH",
  "authorities": [
    {
      "type": "AAD",
      "audience": {
        "type": "PersonalMicrosoftAccount",
        "tenant_id": "consumers"
      }
    }
  ],
  "account_mode": "SINGLE"
}
```

### Scenario 3: Multi-Tenant SaaS Application

**Use Case**: Business app supporting multiple organizations

**Features**:
- Users from any organization can sign in
- Support both work and personal accounts
- Multiple account switching

**Configuration**:
```json
{
  "client_id": "YOUR_APP_ID",
  "redirect_uri": "msauth://com.saas.app/HASH",
  "authorities": [
    {
      "type": "AAD",
      "audience": {
        "type": "AzureADandPersonalMicrosoftAccount",
        "tenant_id": "common"
      }
    }
  ],
  "account_mode": "MULTIPLE",
  "broker_redirect_uri_registered": true
}
```

## Implementation Guide

### Step 1: Plan Your Integration

**Decide on Account Mode:**

| Mode | Use When | Benefits |
|------|----------|----------|
| **SINGLE** | Users only need one account at a time | Simpler UX, automatic account management |
| **MULTIPLE** | Users may need to switch between accounts | Flexibility, enterprise scenarios |

**Determine Required Scopes:**

Common Microsoft Graph scopes:
- `User.Read` - Read user profile
- `Mail.Read` - Read user's email
- `Calendars.Read` - Read user's calendar
- `Files.Read` - Read user's OneDrive files
- `offline_access` - Get refresh tokens

### Step 2: Implement Authentication Flow

**Initialize MSAL (Application Class):**

```java
public class MyApplication extends Application {
    private static IMultipleAccountPublicClientApplication sMSAL;
    
    @Override
    public void onCreate() {
        super.onCreate();
        initializeMSAL();
    }
    
    private void initializeMSAL() {
        PublicClientApplication.createMultipleAccountPublicClientApplication(
            getApplicationContext(),
            R.raw.auth_config,
            new IPublicClientApplication.IMultipleAccountApplicationCreatedListener() {
                @Override
                public void onCreated(IMultipleAccountPublicClientApplication application) {
                    sMSAL = application;
                    Log.d("MSAL", "Initialized successfully");
                }
                
                @Override
                public void onError(MsalException exception) {
                    Log.e("MSAL", "Failed to initialize", exception);
                }
            }
        );
    }
    
    public static IMultipleAccountPublicClientApplication getMSAL() {
        return sMSAL;
    }
}
```

**Sign In Flow:**

```java
public class AuthManager {
    private IMultipleAccountPublicClientApplication mPCA;
    private static final List<String> SCOPES = Arrays.asList(
        "User.Read",
        "Mail.Read",
        "Calendars.Read"
    );
    
    public void signIn(Activity activity, AuthCallback callback) {
        if (mPCA == null) {
            callback.onError(new Exception("MSAL not initialized"));
            return;
        }
        
        AcquireTokenParameters parameters = new AcquireTokenParameters.Builder()
            .startAuthorizationFromActivity(activity)
            .withScopes(SCOPES)
            .withCallback(new AuthenticationCallback() {
                @Override
                public void onSuccess(IAuthenticationResult result) {
                    // Save account for future use
                    IAccount account = result.getAccount();
                    String accessToken = result.getAccessToken();
                    
                    // Store account identifier
                    SharedPreferences prefs = activity.getSharedPreferences("auth", MODE_PRIVATE);
                    prefs.edit()
                        .putString("account_id", account.getId())
                        .apply();
                    
                    callback.onSuccess(account, accessToken);
                }
                
                @Override
                public void onError(MsalException exception) {
                    callback.onError(exception);
                }
                
                @Override
                public void onCancel() {
                    callback.onCancel();
                }
            })
            .build();
        
        mPCA.acquireToken(parameters);
    }
    
    public interface AuthCallback {
        void onSuccess(IAccount account, String accessToken);
        void onError(Exception exception);
        void onCancel();
    }
}
```

**Silent Token Acquisition:**

```java
public void getTokenSilently(IAccount account, TokenCallback callback) {
    if (mPCA == null || account == null) {
        callback.onError(new Exception("Invalid parameters"));
        return;
    }
    
    AcquireTokenSilentParameters parameters = new AcquireTokenSilentParameters.Builder()
        .forAccount(account)
        .fromAuthority(account.getAuthority())
        .withScopes(SCOPES)
        .forceRefresh(false) // Use cached token if valid
        .build();
    
    // Run asynchronously
    new Thread(() -> {
        try {
            IAuthenticationResult result = mPCA.acquireTokenSilent(parameters);
            callback.onSuccess(result.getAccessToken());
        } catch (MsalException e) {
            callback.onError(e);
        } catch (InterruptedException e) {
            callback.onError(e);
        }
    }).start();
}

public interface TokenCallback {
    void onSuccess(String accessToken);
    void onError(Exception exception);
}
```

### Step 3: Integrate with Azure Services

**Call Microsoft Graph API:**

```java
public class GraphService {
    private static final String GRAPH_ENDPOINT = "https://graph.microsoft.com/v1.0";
    
    public void getUserProfile(String accessToken, GraphCallback callback) {
        new Thread(() -> {
            try {
                URL url = new URL(GRAPH_ENDPOINT + "/me");
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                conn.setRequestMethod("GET");
                conn.setRequestProperty("Authorization", "Bearer " + accessToken);
                conn.setRequestProperty("Accept", "application/json");
                
                int responseCode = conn.getResponseCode();
                if (responseCode == 200) {
                    String response = readStream(conn.getInputStream());
                    JSONObject user = new JSONObject(response);
                    
                    String displayName = user.getString("displayName");
                    String email = user.getString("userPrincipalName");
                    
                    callback.onSuccess(displayName, email);
                } else {
                    callback.onError(new Exception("HTTP " + responseCode));
                }
            } catch (Exception e) {
                callback.onError(e);
            }
        }).start();
    }
    
    public void getEmails(String accessToken, int count, EmailCallback callback) {
        new Thread(() -> {
            try {
                URL url = new URL(GRAPH_ENDPOINT + "/me/messages?$top=" + count);
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                conn.setRequestProperty("Authorization", "Bearer " + accessToken);
                
                String response = readStream(conn.getInputStream());
                JSONObject result = new JSONObject(response);
                JSONArray messages = result.getJSONArray("value");
                
                List<Email> emails = new ArrayList<>();
                for (int i = 0; i < messages.length(); i++) {
                    JSONObject msg = messages.getJSONObject(i);
                    emails.add(new Email(
                        msg.getString("subject"),
                        msg.getJSONObject("from").getJSONObject("emailAddress").getString("name"),
                        msg.getString("receivedDateTime")
                    ));
                }
                
                callback.onSuccess(emails);
            } catch (Exception e) {
                callback.onError(e);
            }
        }).start();
    }
    
    private String readStream(InputStream stream) throws IOException {
        BufferedReader reader = new BufferedReader(new InputStreamReader(stream));
        StringBuilder sb = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            sb.append(line);
        }
        return sb.toString();
    }
    
    public interface GraphCallback {
        void onSuccess(String displayName, String email);
        void onError(Exception exception);
    }
    
    public interface EmailCallback {
        void onSuccess(List<Email> emails);
        void onError(Exception exception);
    }
}
```

## Best Practices

### 1. Token Management

**DO:**
- ✅ Always try silent token acquisition first
- ✅ Cache tokens using MSAL's built-in cache
- ✅ Handle token expiration gracefully
- ✅ Request minimum required scopes

**DON'T:**
- ❌ Store tokens in SharedPreferences manually
- ❌ Request excessive scopes upfront
- ❌ Ignore MsalUiRequiredException

### 2. Error Handling

```java
@Override
public void onError(MsalException exception) {
    if (exception instanceof MsalClientException) {
        // Client-side error (no network, etc.)
        showError("Connection error. Please try again.");
    } else if (exception instanceof MsalServiceException) {
        // Server-side error
        MsalServiceException serviceEx = (MsalServiceException) exception;
        if (serviceEx.getErrorCode().equals("invalid_grant")) {
            // User needs to sign in again
            promptSignIn();
        }
    } else if (exception instanceof MsalUiRequiredException) {
        // Need interactive authentication
        signInInteractively();
    }
}
```

### 3. Security

- ✅ Always use HTTPS for API calls
- ✅ Enable broker authentication for SSO
- ✅ Validate tokens on your backend
- ✅ Use certificate pinning for sensitive apps
- ✅ Clear tokens on sign out
- ❌ Never log access tokens in production
- ❌ Don't disable certificate validation

### 4. Performance

- ✅ Initialize MSAL in Application class
- ✅ Use background threads for network calls
- ✅ Implement proper loading states
- ✅ Cache user data appropriately
- ❌ Don't block UI thread for token operations

## Production Checklist

Before releasing your app:

### Azure Portal Configuration
- [ ] Production app registration created
- [ ] Correct redirect URI configured
- [ ] Release keystore signature hash added
- [ ] Required API permissions granted
- [ ] Admin consent obtained (if needed)

### App Configuration
- [ ] Using production client ID
- [ ] URL-encoded signature hash in auth_config.json
- [ ] Non-encoded signature hash in AndroidManifest.xml
- [ ] Broker integration enabled
- [ ] ProGuard rules added (if using minification)

### Code Quality
- [ ] Error handling implemented
- [ ] Loading states for all async operations
- [ ] Token refresh logic tested
- [ ] Sign out clears all cached data
- [ ] No sensitive data in logs

### Testing
- [ ] Tested with multiple account types
- [ ] Tested offline scenarios
- [ ] Tested token expiration
- [ ] Tested on different Android versions
- [ ] Tested with Microsoft Authenticator installed

### Security
- [ ] HTTPS only for API calls
- [ ] No hardcoded secrets
- [ ] Proper keystore management
- [ ] Certificate pinning (if required)
- [ ] Compliance with data protection regulations

## Additional Resources

### Documentation
- [Microsoft Identity Platform](https://learn.microsoft.com/azure/active-directory/develop/)
- [MSAL Android Wiki](https://github.com/AzureAD/microsoft-authentication-library-for-android/wiki)
- [Microsoft Graph Documentation](https://learn.microsoft.com/graph/)

### Sample Code
- [Complete Examples](examples/)
- [Code Snippets](snippets/)
- [Configuration Template](auth_config.template.json)

### Tools
- [Azure Portal](https://portal.azure.com)
- [Graph Explorer](https://developer.microsoft.com/graph/graph-explorer)
- [JWT Decoder](https://jwt.ms)

### Support
- [Stack Overflow](https://stackoverflow.com/questions/tagged/msal) - Tag: `msal` or `azure-ad-msal`
- [GitHub Issues](https://github.com/AzureAD/microsoft-authentication-library-for-android/issues)
- [Microsoft Q&A](https://learn.microsoft.com/answers/topics/azure-active-directory.html)

---

This tutorial covered the essential concepts and implementation patterns for integrating Azure services into Android applications using MSAL. For detailed API documentation and advanced scenarios, refer to the resources listed above.
