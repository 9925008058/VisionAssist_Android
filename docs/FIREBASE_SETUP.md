# Firebase Setup Guide for VisionAssist Android

This guide walks you through setting up Firebase Authentication for VisionAssist.

## Prerequisites

- A Google Account
- Android Studio with the project cloned
- Java 11+

## Step 1: Create a Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click "Create a project"
3. Enter project name: `VisionAssist`
4. Accept the terms → **Continue**
5. (Optional) Enable Google Analytics → **Create project**
6. Wait for provisioning → **Continue**

## Step 2: Register Your Android App

1. In Firebase Console, click the Android icon (or "Add app")
2. Enter the following details:
   - **Package name:** `com.visionassist`
   - **App nickname:** VisionAssist (optional)
   - **SHA-1 certificate fingerprint:** [See instructions below]
3. Click **Register app**
4. Download `google-services.json`
5. Place it in the `app/` directory of your Android project:
   ```
   VisionAssist_Android/
   ├── app/
   │   ├── google-services.json  ← Place here
   │   ├── build.gradle
   │   ├── src/
   │   └── ...
   ```
6. Click **Next** twice, then **Continue to console**

### Getting Your SHA-1 Certificate Fingerprint

**Option A: From Android Studio**
1. Open your project in Android Studio
2. View → Tool Windows → Gradle (or double-click "gradle" from sidebar)
3. Expand: android → Tasks → android
4. Double-click `signingReport`
5. Copy the **SHA-1** value shown in the Gradle console

**Option B: From Command Line**
```bash
# For debug key
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android

# For release key
keytool -list -v -keystore /path/to/your.keystore -alias your_alias
```

Paste the SHA-1 into the Firebase Console field.

## Step 3: Enable Authentication

1. In Firebase Console, go to **Authentication** (left sidebar)
2. Click **Get started** or **Sign-in method**
3. Enable **Email/Password:**
   - Click "Email/Password"
   - Toggle "Enable"
   - Save
4. (Optional) Enable **Google Sign-In:**
   - Click "Google"
   - Toggle "Enable"
   - Select your project from the dropdown
   - Save

## Step 4: Update Your Android Project

### Add Firebase Dependencies

Ensure your `app/build.gradle` includes:

```gradle
dependencies {
    // Firebase
    implementation platform('com.google.firebase:firebase-bom:32.7.0')
    implementation 'com.google.firebase:firebase-auth-ktx'
    implementation 'com.google.firebase:firebase-analytics-ktx'
    
    // Kotlin Coroutines
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.7.3'
    
    // ... other dependencies
}

plugins {
    id 'com.google.gms.google-services' version '4.4.0' apply false
}
```

Also add at the **end of the file**:
```gradle
apply plugin: 'com.google.gms.google-services'
```

And update project-level `build.gradle`:
```gradle
plugins {
    id 'com.google.gms.google-services' version '4.4.0' apply false
}
```

### Sync and Build

1. Android Studio → **File** → **Sync Now**
2. Let Gradle download dependencies
3. Build the project: **Build** → **Make Project**

## Step 5: Test Authentication

### Unit Test Example

Create `app/src/test/java/com/visionassist/auth/AuthManagerTest.kt`:

```kotlin
package com.visionassist.auth

import com.google.firebase.auth.FirebaseAuth
import kotlinx.coroutines.runBlocking
import org.junit.Before
import org.junit.Test
import org.mockito.Mock
import org.mockito.MockitoAnnotations

class AuthManagerTest {
    @Mock
    private lateinit var mockAuth: FirebaseAuth

    private lateinit var authManager: AuthManager

    @Before
    fun setUp() {
        MockitoAnnotations.openMocks(this)
        authManager = AuthManager()
    }

    @Test
    fun testSignUpWithValidEmail() = runBlocking {
        val result = authManager.signUp("test@example.com", "SecurePass123")
        assert(result.isSuccess)
    }

    @Test
    fun testLoginWithValidCredentials() = runBlocking {
        val result = authManager.login("test@example.com", "SecurePass123")
        assert(result.isSuccess)
    }
}
```

Run: **Run** → **Run 'AuthManagerTest'**

## Step 6: Deploy to Emulator/Device

1. Build and install:
   ```bash
   ./gradlew installDebug
   ```

2. Launch the app and test:
   - Click "Sign Up"
   - Enter email and password
   - Verify success message
   - Log out and log back in

## Troubleshooting

### "google-services.json not found"
- Ensure the file is in `app/` directory, not `app/src/`
- Refresh Gradle: **File** → **Sync Now**

### "Could not resolve com.google.firebase:firebase-auth"
- Update `com.google.gms:google-services` plugin to latest
- Clear Gradle cache: `./gradlew clean`

### "Authentication fails with "Error: [auth/invalid-email]""
- Ensure email format is valid: `user@example.com`
- Check Firebase Console → Authentication → Users for any typos

### "Password too short" error
- Firebase enforces minimum 6 characters
- VisionAssist enforces 8+ chars with mixed case

## Next Steps

1. ✅ Configure password policies (done in AuthManager)
2. ✅ Test signup/login flows
3. Integrate Google Sign-In (optional)
4. Set up Firestore for user profile data
5. Enable Crashlytics for error tracking

## Security Best Practices

✅ **Enabled:**
- Password validation (8+ chars, mixed case, digits)
- HTTPS-only network communication
- Firebase Auth session management

⚠️ **To Enable Later:**
- Email verification before account activation
- 2FA (Two-Factor Authentication)
- Password reset via email
- Account lockout after failed attempts

## Resources

- [Firebase Authentication Docs](https://firebase.google.com/docs/auth)
- [Firebase Console](https://console.firebase.google.com)
- [Google Play Services Setup](https://developers.google.com/android/setup)
- [Android Security Best Practices](https://developer.android.com/training/articles/security-tips)
