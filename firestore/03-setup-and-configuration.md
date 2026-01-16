# 3. SETUP & CONFIGURATION

## 1. What This Chapter Covers

This chapter walks you through setting up Firestore for a production Notes application. You'll learn:

- How to create a Firebase project and configure it in the console
- How to set up Firebase CLI for local development and deployment
- How to integrate Firestore into a Flutter/Dart application
- How to manage multiple environments (development, staging, production)

Proper setup is critical because misconfiguration leads to security vulnerabilities, deployment failures, and development workflow issues. This chapter ensures you start with a solid foundation.

## 2. Core Concepts (Simple & Clear)

### 2.1 Firebase Console Setup

The Firebase Console is the web interface where you manage your Firebase projects, configure services, and view analytics.

#### Creating a Firebase Project

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click "Add project" or "Create a project"
3. Enter project name (e.g., "notes-app")
4. Choose whether to enable Google Analytics (recommended for production)
5. Select Analytics account (or create new)
6. Click "Create project"

**Project naming:** Use descriptive names like `notes-app-prod`, `notes-app-dev`. Avoid special characters and spaces.

#### Adding Your App to Firebase

After creating a project, add your app platforms:

**For Flutter (Android + iOS):**
1. Click the Android icon (or iOS icon)
2. Enter package name (Android) or bundle ID (iOS)
   - Android: `com.example.notesapp`
   - iOS: `com.example.notesapp`
3. Enter app nickname (optional)
4. Download configuration files:
   - Android: `google-services.json` → `android/app/`
   - iOS: `GoogleService-Info.plist` → `ios/Runner/`

**For Web:**
1. Click the web icon (`</>`)
2. Register app with nickname
3. Copy the Firebase configuration object (you'll add this to your code)

#### Enabling Firestore

1. In Firebase Console, go to "Build" → "Firestore Database"
2. Click "Create database"
3. Choose security rules mode:
   - **Start in test mode:** Allows read/write for 30 days (development only)
   - **Start in production mode:** Requires security rules immediately
4. Select location (choose closest to your users)
5. Click "Enable"

**Location selection:** Once set, location cannot be changed. Choose based on where most users are located. Multi-region is available for global apps but costs more.

### 2.2 Firebase CLI Setup

Firebase CLI lets you manage Firebase projects from the command line, deploy security rules, and run local emulators.

#### Installing Firebase CLI

**macOS/Linux:**
```bash
npm install -g firebase-tools
```

**Windows:**
```bash
npm install -g firebase-tools
```

**Verify installation:**
```bash
firebase --version
```

#### Login and Project Initialization

**Login to Firebase:**
```bash
firebase login
```

This opens your browser to authenticate. After login, you can access all your Firebase projects.

**Initialize Firebase in your project:**
```bash
cd /path/to/your/flutter/project
firebase init
```

**During initialization, select:**
- Firestore (for database rules)
- Emulators (for local development)
- Storage (if using Firebase Storage)

**Project selection:**
- Choose existing project or create new one
- Select the project you created in the console

This creates:
- `firebase.json` - Firebase configuration
- `.firebaserc` - Project aliases
- `firestore.rules` - Security rules file
- `firestore.indexes.json` - Index configuration

#### Emulator Suite

Firebase Emulators let you develop and test locally without using your production database.

**Start emulators:**
```bash
firebase emulators:start
```

**Start specific emulators:**
```bash
firebase emulators:start --only firestore,auth
```

**Emulator UI:** Access at `http://localhost:4000`

**Benefits:**
- No billing costs during development
- Faster iteration (no network latency)
- Safe testing (can't break production)
- Offline development

**Important:** Emulators use separate data from production. Data in emulators doesn't sync to production.

### 2.3 Flutter/Dart Setup

Flutter apps need Firebase dependencies and platform-specific configuration.

#### Adding Dependencies

**In `pubspec.yaml`:**
```yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^2.24.2
  cloud_firestore: ^4.13.6
```

**Install dependencies:**
```bash
flutter pub get
```

**Version management:** Use `^` for version ranges. Check [pub.dev](https://pub.dev) for latest stable versions.

#### Platform-Specific Setup

**Android Setup:**

1. Add `google-services.json` to `android/app/`
2. Update `android/build.gradle`:
```gradle
buildscript {
    dependencies {
        classpath 'com.google.gms:google-services:4.4.0'
    }
}
```

3. Update `android/app/build.gradle`:
```gradle
apply plugin: 'com.google.gms.google-services'

dependencies {
    implementation platform('com.google.firebase:firebase-bom:32.7.0')
}
```

**iOS Setup:**

1. Add `GoogleService-Info.plist` to `ios/Runner/`
2. Open `ios/Runner.xcworkspace` in Xcode
3. Ensure `GoogleService-Info.plist` is added to the Runner target
4. Update `ios/Podfile` (usually auto-handled):
```ruby
platform :ios, '12.0'
```

5. Install pods:
```bash
cd ios
pod install
cd ..
```

**Web Setup:**

1. Create `lib/firebase_options.dart` (or use FlutterFire CLI)
2. Add Firebase config:
```dart
import 'package:firebase_core/firebase_core.dart';

void main() async {
  await Firebase.initializeApp(
    options: FirebaseOptions(
      apiKey: "your-api-key",
      appId: "your-app-id",
      messagingSenderId: "your-sender-id",
      projectId: "your-project-id",
    ),
  );
  runApp(MyApp());
}
```

**Recommended:** Use FlutterFire CLI to auto-generate configuration:
```bash
flutterfire configure
```

This generates `lib/firebase_options.dart` for all platforms automatically.

#### Firebase App Initialization

**In `main.dart`:**
```dart
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  runApp(MyApp());
}
```

**Error handling:**
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  try {
    await Firebase.initializeApp(
      options: DefaultFirebaseOptions.currentPlatform,
    );
  } catch (e) {
    print('Firebase initialization error: $e');
    // Handle error (show user message, use fallback, etc.)
  }
  
  runApp(MyApp());
}
```

### 2.4 Environment Setup

Managing multiple environments (dev, staging, prod) prevents accidents and organizes development.

#### Development vs Production

**Development Environment:**
- Use Firebase Emulators for local testing
- Separate Firebase project for team development
- Relaxed security rules for testing
- Test data that can be deleted

**Production Environment:**
- Real Firebase project with billing enabled
- Strict security rules
- Real user data
- Monitoring and analytics enabled

**Best practice:** Use separate Firebase projects:
- `notes-app-dev` - Development
- `notes-app-staging` - Pre-production testing
- `notes-app-prod` - Production

#### Multiple Firebase Projects

**Using project aliases:**

In `.firebaserc`:
```json
{
  "projects": {
    "default": "notes-app-dev",
    "dev": "notes-app-dev",
    "staging": "notes-app-staging",
    "prod": "notes-app-prod"
  }
}
```

**Switch projects:**
```bash
firebase use dev
firebase use staging
firebase use prod
```

**Deploy to specific project:**
```bash
firebase deploy --project prod
```

#### Environment Variables

**Flutter environment configuration:**

Create configuration files:
- `lib/config/dev_config.dart`
- `lib/config/prod_config.dart`

**Example `dev_config.dart`:**
```dart
class AppConfig {
  static const String firebaseProjectId = 'notes-app-dev';
  static const bool useEmulators = true;
  static const String apiBaseUrl = 'http://localhost:8080';
}
```

**Example `prod_config.dart`:**
```dart
class AppConfig {
  static const String firebaseProjectId = 'notes-app-prod';
  static const bool useEmulators = false;
  static const String apiBaseUrl = 'https://api.example.com';
}
```

**Using flavors (advanced):**

In `android/app/build.gradle`:
```gradle
android {
    flavorDimensions "environment"
    productFlavors {
        dev {
            dimension "environment"
            applicationIdSuffix ".dev"
        }
        prod {
            dimension "environment"
        }
    }
}
```

Run with flavor:
```bash
flutter run --flavor dev
```

## 3. Practical Examples

### Complete Setup Workflow for Notes App

**Step 1: Create Firebase Project**
```
1. Go to Firebase Console
2. Create project: "notes-app-prod"
3. Enable Firestore (production mode)
4. Select region: us-central1
```

**Step 2: Add Flutter App**
```
1. Add Android app:
   - Package: com.example.notesapp
   - Download google-services.json → android/app/
   
2. Add iOS app:
   - Bundle ID: com.example.notesapp
   - Download GoogleService-Info.plist → ios/Runner/
   
3. Add Web app:
   - Register app
   - Note config values
```

**Step 3: Initialize Firebase CLI**
```bash
cd ~/projects/notes-app
firebase login
firebase init

# Select:
# - Firestore
# - Emulators: Firestore, Auth
# - Project: notes-app-prod
```

**Step 4: Configure Flutter**
```yaml
# pubspec.yaml
dependencies:
  firebase_core: ^2.24.2
  cloud_firestore: ^4.13.6
```

```bash
flutter pub get
flutterfire configure
```

**Step 5: Initialize in Code**
```dart
// main.dart
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  runApp(NotesApp());
}
```

### Example Firebase Configuration Files

**`firebase.json`:**
```json
{
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  },
  "emulators": {
    "auth": {
      "port": 9099
    },
    "firestore": {
      "port": 8080
    },
    "ui": {
      "enabled": true,
      "port": 4000
    }
  }
}
```

**`.firebaserc`:**
```json
{
  "projects": {
    "default": "notes-app-dev",
    "dev": "notes-app-dev",
    "prod": "notes-app-prod"
  }
}
```

**`firestore.rules` (initial):**
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if false; // Start restrictive
    }
  }
}
```

### Connecting to Emulators in Flutter

**Development setup:**
```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  // Connect to emulators in debug mode
  if (kDebugMode) {
    FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080);
  }
  
  runApp(NotesApp());
}
```

**Conditional emulator connection:**
```dart
void setupFirebaseEmulators() {
  if (kDebugMode && !kIsWeb) {
    FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080);
    FirebaseAuth.instance.useAuthEmulator('localhost', 9099);
  }
}
```

## 4. Common Design Decisions

### Why Choose This Approach

**Separate projects for dev/staging/prod:**
- **Pros:** Complete isolation, safe testing, clear separation of concerns
- **Cons:** More projects to manage, need to sync configuration
- **Alternative:** Single project with environment prefixes in collection names
- **Recommendation:** Use separate projects for production apps. Single project is fine for learning.

**Using FlutterFire CLI vs manual setup:**
- **FlutterFire CLI:** Automatic, generates all platform configs, less error-prone
- **Manual setup:** More control, understand each step, works if CLI has issues
- **Recommendation:** Use FlutterFire CLI for new projects. Understand manual setup for troubleshooting.

**Emulators vs direct Firebase connection:**
- **Emulators:** Free, fast, offline, safe for testing
- **Direct connection:** Real data, tests actual Firebase behavior, requires internet
- **Recommendation:** Use emulators for daily development. Use direct connection for integration testing before deployment.

### Environment Management Strategy

**Option 1: Build flavors (Android/iOS specific)**
- Different app IDs per environment
- Can install dev and prod apps simultaneously
- More complex setup
- Best for: Teams needing separate installable apps

**Option 2: Runtime configuration**
- Single app, config loaded at runtime
- Simpler setup
- Can't install multiple versions
- Best for: Solo developers, simpler apps

**Option 3: Firebase project switching**
- Same codebase, switch Firebase projects
- Simple, works across platforms
- Manual project switching
- Best for: Small teams, web-focused apps

**Recommendation:** Start with Option 3 (project switching). Move to flavors if you need separate installable apps per environment.

## 5. Common Mistakes to Avoid

### Beginner Mistakes

**1. Forgetting to add configuration files**
```bash
# BAD: Missing google-services.json
# App crashes on Firebase.initializeApp()

# GOOD: Verify files exist
ls android/app/google-services.json
ls ios/Runner/GoogleService-Info.plist
```

**2. Wrong package name/bundle ID**
```dart
// BAD: Package name mismatch
// Android: com.example.notesapp
// Firebase: com.example.notes  // Mismatch!

// GOOD: Exact match required
// Both must be: com.example.notesapp
```

**3. Not initializing Firebase before use**
```dart
// BAD: Using Firestore before initialization
void main() {
  runApp(MyApp()); // Firebase not initialized!
}

class MyApp extends StatelessWidget {
  Widget build(BuildContext context) {
    FirebaseFirestore.instance.collection('notes').get(); // Crashes!
  }
}

// GOOD: Initialize first
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(...);
  runApp(MyApp());
}
```

**4. Using production Firebase in development**
```dart
// BAD: Always connecting to production
// Wastes quota, risks data corruption

// GOOD: Use emulators in development
if (kDebugMode) {
  FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080);
}
```

### Configuration Pitfalls

**1. Hardcoding API keys in source code**
```dart
// BAD: Exposed in version control
const apiKey = "AIzaSyC-example-key-here";

// GOOD: Use firebase_options.dart (generated, gitignored secrets)
import 'firebase_options.dart';
await Firebase.initializeApp(
  options: DefaultFirebaseOptions.currentPlatform,
);
```

**2. Not updating dependencies regularly**
```yaml
# BAD: Old versions, security issues
firebase_core: ^1.0.0

# GOOD: Keep updated (check pub.dev)
firebase_core: ^2.24.2
```

**3. Missing platform-specific setup**
```gradle
// BAD: Android build.gradle missing Google Services plugin
// App builds but Firebase doesn't work

// GOOD: Include in android/app/build.gradle
apply plugin: 'com.google.gms.google-services'
```

**4. Incorrect emulator connection**
```dart
// BAD: Wrong port or host
FirebaseFirestore.instance.useFirestoreEmulator('127.0.0.1', 8080);
// Should be 'localhost' for some setups

// GOOD: Match firebase.json emulator config
FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080);
```

### Security Warnings

**1. Test mode security rules in production**
```javascript
// BAD: Allows anyone to read/write
match /{document=**} {
  allow read, write: if true;
}

// GOOD: Restrictive rules
match /notes/{noteId} {
  allow read, write: if request.auth != null 
    && request.auth.uid == resource.data.userId;
}
```

**2. Exposing project configuration**
```dart
// BAD: Committing sensitive config
// firebase_options.dart with API keys in public repo

// GOOD: Use .gitignore
# .gitignore
firebase_options.dart
google-services.json
GoogleService-Info.plist
```

**3. Using same project for dev and prod**
```bash
# BAD: Risk of deleting production data during development
firebase use notes-app-prod  # Using prod for dev work

# GOOD: Separate projects
firebase use dev    # Development
firebase use prod   # Production only
```

## 6. Summary & Checklist

### Summary

- **Firebase Console** is where you create projects, enable Firestore, and download configuration files
- **Firebase CLI** enables command-line management, local emulators, and deployment
- **Flutter setup** requires adding dependencies, platform-specific configuration files, and initializing Firebase in code
- **Environment management** (dev/staging/prod) prevents accidents and organizes development workflows
- **Emulators** provide free, fast local development without affecting production

### Checklist: You Are Ready When You Can...

- [ ] Create a Firebase project in the console
- [ ] Add Android, iOS, and Web apps to your Firebase project
- [ ] Download and place configuration files in correct locations
- [ ] Install and authenticate with Firebase CLI
- [ ] Initialize Firebase in your Flutter project using `firebase init`
- [ ] Add `firebase_core` and `cloud_firestore` to `pubspec.yaml`
- [ ] Configure Android and iOS platforms with Google Services
- [ ] Initialize Firebase in your `main.dart` before using Firestore
- [ ] Start and connect to Firebase Emulators for local development
- [ ] Set up separate Firebase projects for dev and production
- [ ] Switch between Firebase projects using `firebase use`
- [ ] Verify your app connects to Firestore without errors
- [ ] Understand the difference between emulator and production connections

### Verification Steps

**Test your setup:**

1. **Verify Firebase initialization:**
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  // Test connection
  try {
    await FirebaseFirestore.instance
        .collection('test')
        .doc('connection')
        .set({'status': 'connected'});
    print('Firestore connected successfully!');
  } catch (e) {
    print('Firestore connection error: $e');
  }
  
  runApp(MyApp());
}
```

2. **Verify emulator connection:**
```bash
# Terminal 1: Start emulators
firebase emulators:start

# Terminal 2: Run Flutter app
flutter run

# Check emulator UI: http://localhost:4000
# Should see test document created
```

3. **Verify production connection:**
```bash
# Switch to production project
firebase use prod

# Deploy rules
firebase deploy --only firestore:rules

# Run app (without emulator connection)
# Should connect to production Firestore
```

### Next Steps

Once setup is complete, you're ready to:
- Read data from Firestore (Chapter 4)
- Write data to Firestore (Chapter 5)
- Query and filter your data (Chapter 6)
