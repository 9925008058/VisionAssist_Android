# VisionAssist Android Architecture

## High-Level Design

```
User Input
   │
   ├─ Voice → SpeechRecognizer → NavigationFSM → Action
   ├─ Tap    → TapListener      → NavigationFSM → Action
   └─ Gesture→ GestureDetector  → NavigationFSM → Action

   ↓

Application Logic
   ├─ AuthManager     (Firebase login/signup)
   ├─ MLKitModelManager (Model loading & caching)
   └─ NavigationFSM   (State machine for app flow)

   ↓

ML Processing
   ├─ CameraX         (Frame capture)
   ├─ ML Kit          (Object detection, classification)
   └─ Image Processor (Resizing, normalization)

   ↓

Output Feedback
   ├─ TextToSpeech     (Audio announcements)
   ├─ BluetoothHaptic  (Vibration patterns)
   └─ Logger (Timber)  (Debug output)
```

## Module Details

### 1. MainActivity (Entry Point)

**File:** `app/src/main/java/com/visionassist/MainActivity.kt`

**Responsibilities:**
- Request camera & microphone permissions
- Initialize CameraX pipeline
- Orchestrate speech recognition lifecycle
- Handle voice input and route to FSM

**Key Methods:**
```kotlin
private fun checkPermissions()
private fun launchVoice(prompt: String)
private fun handleVoiceInput(heard: String)
private fun openCamera()
private fun greetUser()
```

**Lifecycle:**
```
onCreate() → checkPermissions() → greetUser()
   ↓
onStart() → Initialize CameraX
   ↓
User Voice Input → launchVoice() → SpeechRecognizer → handleVoiceInput()
   ↓
navigationFSM.transition() → Navigate to next state
```

---

### 2. AuthManager (Security Layer)

**File:** `app/src/main/java/com/visionassist/auth/AuthManager.kt`

**Responsibilities:**
- Firebase signup with email/password
- Secure login/logout
- User session management

**Key Methods:**
```kotlin
suspend fun signUp(email: String, password: String): Result<String>
suspend fun login(email: String, password: String): Result<String>
fun logout()
fun isAuthenticated(): Boolean
fun getCurrentUserId(): String?
```

**Security Features:**
- ✅ Firebase handles password hashing (bcrypt-like)
- ✅ No plaintext credentials stored locally
- ✅ Enforces HTTPS for all Firebase calls
- ✅ Session tokens auto-refreshed

**Usage Example:**
```kotlin
val authManager = AuthManager()
val result = authManager.signUp("user@example.com", "SecurePass123")

if (result.isSuccess) {
    val userId = result.getOrNull()
    // Proceed to camera
} else {
    val error = result.exceptionOrNull()
    showError(error.message)
}
```

---

### 3. PasswordValidator (Input Validation)

**File:** `app/src/main/java/com/visionassist/auth/PasswordValidator.kt`

**Enforces:**
- Minimum 8 characters
- At least one uppercase letter (A-Z)
- At least one lowercase letter (a-z)
- At least one digit (0-9)
- Optionally: special character

**Usage:**
```kotlin
val result = PasswordValidator.validate("MyPassword123")
when (result) {
    is PasswordValidationResult.Success -> proceed()
    is PasswordValidationResult.Failure -> showErrors(result.errors)
}
```

---

### 4. NavigationFSM (State Machine)

**File:** `app/src/main/java/com/visionassist/navigation/NavigationFSM.kt`

**States:**
```kotlin
sealed class AppState {
    object Welcome : AppState()
    object AskingUsername : AppState()
    object AskingPassword : AppState()
    object CameraActive : AppState()
    object DetectingObject : AppState()
    object ReadingText : AppState()
    object Error : AppState()
}
```

**State Transitions:**
```
Welcome
  ├─ "Sign Up" → AskingUsername (signup branch)
  └─ "Login" → AskingUsername (login branch)

AskingUsername
  └─ Valid input → AskingPassword

AskingPassword
  ├─ Valid password → CameraActive
  └─ Invalid → Error

CameraActive
  ├─ "Detect" → DetectingObject
  └─ "Read" → ReadingText

DetectingObject / ReadingText
  └─ "Back" → CameraActive
```

**Implementation:**
```kotlin
data class NavigationState(
    val currentState: AppState,
    val previousState: AppState? = null,
    val context: Map<String, String> = emptyMap()
)

class NavigationFSM {
    private val _state = MutableStateFlow<NavigationState>(...)
    val state: StateFlow<NavigationState> = _state.asStateFlow()
    
    fun transition(input: String) {
        val nextState = when (currentState) {
            Welcome → handleWelcome(input)
            AskingUsername → handleUsername(input)
            // ...
        }
        _state.value = nextState
    }
}
```

---

### 5. MLKitModelManager (ML Pipeline)

**File:** `app/src/main/java/com/visionassist/ml/MLKitModelManager.kt`

**Responsibilities:**
- Load ML Kit models on startup
- Cache models in app memory
- Run inference on camera frames
- Post-process results

**Models Used:**
- **Object Detection:** `com.google.mlkit.vision.objects.ObjectDetection`
- **Text Recognition (OCR):** `com.google.mlkit.vision.text.TextRecognition`
- **Face Detection:** `com.google.mlkit.vision.face.FaceDetection`

**Methods:**
```kotlin
suspend fun loadModels(): Result<Unit>
suspend fun detectObjects(image: InputImage): Result<List<DetectedObject>>
suspend fun recognizeText(image: InputImage): Result<Text>
fun clearCache()
```

**Performance Optimizations:**
- Models loaded once and cached
- Inference runs on background thread
- Frame skipping: process every Nth frame (adaptive)

---

### 6. BluetoothHapticManager (Haptic Feedback)

**File:** `app/src/main/java/com/visionassist/bluetooth/BluetoothHapticManager.kt`

**Haptic Patterns:**
- `onSuccess()` → Short vibration (100ms)
- `onError()` → Long vibration (300ms, pulse)
- `onNavigation(direction)` → Directional vibration

**Bluetooth Integration:**
- Connects to compatible haptic devices (vests, bracelets)
- Fallback to phone vibration if device unavailable

---

## Data Flow Examples

### Example 1: Voice Recognition → Object Detection

```
1. User says "detect"
   └─ launchVoice("What would you like to do?")

2. SpeechRecognizer captures audio
   └─ onSuccess(heard="detect")

3. MainActivity calls handleVoiceInput("detect")
   └─ NavigationFSM.transition("detect")

4. FSM updates state to DetectingObject
   └─ MainActivity observes state change

5. CameraX frame available
   └─ MLKitModelManager.detectObjects(frame)

6. Results returned
   └─ TTS speaks: "Person detected at 2 o'clock"
   └─ Haptic vibrates in direction

7. User says "back"
   └─ FSM transitions to CameraActive
```

### Example 2: Authentication Flow

```
1. User says "sign up"
   └─ FSM → AskingUsername

2. User provides email via voice
   └─ FSM → AskingPassword

3. User provides password
   └─ AuthManager.signUp(email, password)

4. Firebase creates user
   └─ TTS: "Account created successfully"
   └─ FSM → CameraActive

5. On next launch:
   └─ User says "login"
   └─ AuthManager.login(email, password)
   └─ Firebase authenticates
   └─ FSM → CameraActive
```

## Concurrency Model

### Coroutines + StateFlow

```kotlin
// MainActivity
launchWhenStarted {
    navigationFSM.state.collect { state ->
        when (state.currentState) {
            CameraActive → openCamera()
            DetectingObject → detectObjects()
            ReadingText → recognizeText()
            // ...
        }
    }
}

// Background inference
viewModelScope.launch(Dispatchers.Default) {
    mlKitManager.detectObjects(frame)
}.invokeOnCompletion { throwable ->
    if (throwable != null) Timber.e(throwable)
}
```

## Testing Strategy

### Unit Tests
- `AuthManagerTest` → Firebase signup/login
- `NavigationFSMTest` → State transitions
- `PasswordValidatorTest` → Input validation
- `MLKitModelManagerTest` → Model caching

### Instrumented Tests
- `CameraXIntegrationTest` → Frame capture
- `VoiceRecognitionTest` → Speech recognition lifecycle
- `MainActivityTest` → Permission requests & navigation

### Manual Testing
- TalkBack screen reader compatibility
- Voice commands in noisy environment
- Real device with varying camera quality
- Low-end device (API 26) performance

## Security Considerations

✅ **Implemented:**
- Firebase Auth for user credentials
- HTTPS-only network calls
- No plaintext storage
- Code obfuscation (ProGuard/R8)

⚠️ **Future Enhancements:**
- Encrypt local cache
- Rate limit authentication attempts
- Hardware-backed keystore
- Biometric authentication option

## Performance Metrics

| Metric | Target | Current |
|--------|--------|----------|
| App Startup | <2s | ~1.5s |
| Camera Open | <1s | ~0.8s |
| Object Detection | <500ms | ~300ms |
| TTS Latency | <500ms | ~200ms |
| Memory Usage | <150MB | ~120MB |
| Battery Drain | <5% per hour | ~3% per hour |

## Future Architecture Changes

1. **ViewModel Integration** → More robust configuration handling
2. **Dependency Injection** → Dagger/Hilt for cleaner module coupling
3. **Repository Pattern** → Abstract Firebase data access
4. **Room Database** → Local persistence for offline profiles
5. **Workers API** → Background model caching updates
