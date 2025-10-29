# MSAL for Android - Example Applications

This directory contains example applications demonstrating how to use MSAL for Android to integrate Azure authentication.

## Quick Navigation

### For Beginners

**Start here:** [Minimal Example](minimal-example/) 
- Absolute minimum code needed (~150 lines total)
- Perfect for learning the basics
- Single activity, simple UI
- Great starting point

### For Production Apps

Choose based on your use case:

#### [Multiple Account Mode](hello-msal-multiple-account/)
**Use this if:**
- Users may need to switch between accounts
- Supporting enterprise scenarios
- Building multi-user apps
- Need maximum flexibility

**Features:**
- Support multiple signed-in accounts
- Account switching
- Account management UI
- Microsoft Graph API integration
- Complete error handling
- Production-ready code

#### [Single Account Mode](hello-msal-single-account/)
**Use this if:**
- Users only need one account at a time
- Simpler user experience desired
- Personal/consumer apps
- One-user-per-device scenario

**Features:**
- Simplified account management
- Automatic account handling
- Microsoft Graph API integration
- Clean, simple UX
- Production-ready code

## Example Comparison

| Feature | Minimal Example | Multiple Account | Single Account |
|---------|----------------|------------------|----------------|
| **Complexity** | Very Simple | Advanced | Moderate |
| **Lines of Code** | ~150 | ~2000 | ~1800 |
| **Account Support** | Single | Multiple | Single |
| **UI Completeness** | Basic | Full | Full |
| **Error Handling** | Basic | Comprehensive | Comprehensive |
| **Graph API Calls** | No | Yes | Yes |
| **Production Ready** | No | Yes | Yes |
| **Best For** | Learning | Enterprise apps | Consumer apps |

## What Each Example Includes

### Minimal Example
```
minimal-example/
└── README.md          # Complete code snippets and setup guide
    ├── MainActivity.java (inline)
    ├── Layout XML (inline)
    ├── Configuration files (inline)
    └── Setup instructions
```

### Multiple Account Example
```
hello-msal-multiple-account/
├── app/
│   ├── src/main/java/
│   │   └── com/example/msalmultipleaccount/
│   │       ├── MainActivity.java        # Main activity with account spinner
│   │       ├── AuthHelper.java          # MSAL wrapper and token management
│   │       └── GraphHelper.java         # Microsoft Graph API client
│   ├── src/main/res/
│   │   ├── layout/
│   │   │   └── activity_main.xml        # UI with account selection
│   │   └── raw/
│   │       └── auth_config.json         # MSAL configuration
│   └── build.gradle                     # App dependencies
├── build.gradle                         # Project configuration
└── gradle.properties                    # Gradle settings
```

### Single Account Example
```
hello-msal-single-account/
├── app/
│   ├── src/main/java/
│   │   └── com/example/msalsingleaccount/
│   │       ├── MainActivity.java        # Main activity
│   │       ├── AuthHelper.java          # MSAL wrapper
│   │       └── GraphHelper.java         # Graph API client
│   ├── src/main/res/
│   │   ├── layout/
│   │   │   └── activity_main.xml        # Simple UI
│   │   └── raw/
│   │       └── auth_config.json         # MSAL configuration
│   └── build.gradle
├── build.gradle
└── gradle.properties
```

## How to Use These Examples

### Option 1: Learning (Minimal Example)
1. Read the [Minimal Example README](minimal-example/README.md)
2. Copy the code snippets into your own project
3. Follow the setup instructions
4. Understand the basics before moving to full examples

### Option 2: Clone and Modify (Full Examples)
1. Copy the entire example folder to your workspace
2. Open in Android Studio
3. Update `auth_config.json` with your Azure app details
4. Update package name in build.gradle and manifest
5. Update AndroidManifest.xml with your signature hash
6. Run and test
7. Customize for your needs

### Option 3: Reference (All Examples)
1. Keep these examples as reference
2. Copy specific code patterns as needed
3. Refer to AuthHelper.java for MSAL integration patterns
4. Refer to GraphHelper.java for API call patterns

## Before Running Any Example

You need:

1. **Azure App Registration**
   - Go to [Azure Portal](https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade)
   - Create a new app registration
   - Note the Client ID
   - Add Android platform
   - Configure redirect URI

2. **Signature Hash**
   ```bash
   keytool -exportcert -alias androiddebugkey -keystore ~/.android/debug.keystore | openssl dgst -sha1 -binary | openssl base64
   ```
   Password: `android`

3. **Update Configuration**
   - Edit `auth_config.json` with your Client ID
   - Update redirect_uri with your package and hash (URL encoded)
   - Update AndroidManifest.xml with your package and hash (NOT URL encoded)

## Common Tasks

### How to Sign In
See: `AuthHelper.java` → `acquireTokenInteractive()` (Multiple Account) or `signIn()` (Single Account)

### How to Get Token Silently
See: `AuthHelper.java` → `acquireTokenSilent()`

### How to Call Microsoft Graph
See: `GraphHelper.java` → `callGraphAPI()` or `getUserInfo()`

### How to Sign Out
See: `AuthHelper.java` → `removeAccount()` (Multiple Account) or `signOut()` (Single Account)

### How to Handle Errors
See: `MainActivity.java` → Authentication callbacks

## Additional Resources

### Documentation
- [Getting Started Guide](../GETTING_STARTED.md) - 15-minute quick start
- [Quick Start Guide](../QUICKSTART.md) - Detailed walkthrough
- [Azure Android Tutorial](../AZURE_ANDROID_TUTORIAL.md) - Complete tutorial
- [Main README](../README.md) - Full documentation

### Code Snippets
The [snippets](../snippets/) directory contains isolated code examples:
- Java and Kotlin versions
- Each MSAL operation in a separate file
- Easy to copy and paste
- Well-commented

### Configuration
- [Configuration Template](../auth_config.template.json) - All available options
- Documentation for each configuration setting
- Default values and recommendations

## Need Help?

1. **First time?** Start with [Getting Started Guide](../GETTING_STARTED.md)
2. **Issues?** Check the [README Troubleshooting](../README.md#common-issues-and-solutions) section
3. **Questions?** See [Microsoft Q&A](https://learn.microsoft.com/answers/topics/azure-active-directory.html)
4. **Bugs?** Open an [issue](https://github.com/AzureAD/microsoft-authentication-library-for-android/issues)

## Example Selection Guide

**Choose Minimal Example if:**
- ✅ You're new to MSAL
- ✅ Want to understand the basics first
- ✅ Need a simple reference
- ✅ Want minimum code to start

**Choose Multiple Account Example if:**
- ✅ Building enterprise apps
- ✅ Users need multiple work accounts
- ✅ Implementing account switching
- ✅ Need production-ready code
- ✅ Want comprehensive error handling

**Choose Single Account Example if:**
- ✅ Building consumer apps
- ✅ One user per device
- ✅ Want simpler account management
- ✅ Don't need account switching
- ✅ Need production-ready code

---

Happy coding! 🚀 These examples will get you started with Azure authentication in your Android app quickly and correctly.
