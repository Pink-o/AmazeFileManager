# AmazeFileManager Build Guide

This guide explains how to build AmazeFileManager from source on Linux.

## Prerequisites

### 1. Java Development Kit (JDK 17+)

```bash
# Install OpenJDK 21 (full JDK, not just JRE)
sudo apt-get install openjdk-21-jdk-headless
```

Verify installation:
```bash
java -version
```

### 2. Android SDK

Download and set up the Android SDK:

```bash
# Create SDK directory
mkdir -p /path/to/android/sdk
cd /path/to/android/sdk

# Download command-line tools (get latest from https://developer.android.com/studio#command-tools)
wget https://dl.google.com/android/repository/commandlinetools-linux-13114758_latest.zip -O cmdline-tools.zip
unzip cmdline-tools.zip
mkdir -p cmdline-tools/latest
mv cmdline-tools/bin cmdline-tools/lib cmdline-tools/NOTICE.txt cmdline-tools/source.properties cmdline-tools/latest/
rm cmdline-tools.zip

# Accept licenses
yes | ./cmdline-tools/latest/bin/sdkmanager --sdk_root=/path/to/android/sdk --licenses

# Install required SDK components
./cmdline-tools/latest/bin/sdkmanager --sdk_root=/path/to/android/sdk \
    "platforms;android-34" \
    "build-tools;34.0.0" \
    "platform-tools"
```

### 3. Android NDK

The project requires NDK version **28.2.13676358** (r28c):

```bash
./cmdline-tools/latest/bin/sdkmanager --sdk_root=/path/to/android/sdk "ndk;28.2.13676358"
```

### 4. Rust (for native code compilation)

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Add Android targets
rustup target add armv7-linux-androideabi aarch64-linux-android i686-linux-android x86_64-linux-android
```

## Project Setup

### 1. Clone the repository

```bash
git clone https://github.com/TeamAmaze/AmazeFileManager.git
cd AmazeFileManager
```

### 2. Configure local.properties

Create `local.properties` in the project root:

```properties
sdk.dir=/path/to/android/sdk
ndk.dir=/path/to/android/sdk/ndk/28.2.13676358
```

### 3. Set environment variables

```bash
export ANDROID_HOME=/path/to/android/sdk
export ANDROID_NDK_HOME=/path/to/android/sdk/ndk/28.2.13676358
```

## Building

### Debug Build (auto-signed, for testing)

```bash
./gradlew assembleFdroidDebug
```

Output: `app/build/outputs/apk/fdroid/debug/app-fdroid-debug.apk`

### Release Build (unsigned)

```bash
./gradlew assembleFdroidRelease
```

Output: `app/build/outputs/apk/fdroid/release/app-fdroid-release-unsigned.apk`

### Build Variants

The project has two flavors:
- **fdroid** - Open source version without Google Play dependencies
- **play** - Version with Google Play billing support (requires additional setup)

Build commands:
- `assembleFdroidDebug` / `assembleFdroidRelease`
- `assemblePlayDebug` / `assemblePlayRelease`

## Signing the APK

Release builds need to be signed before installation.

### Option 1: Create a debug keystore

```bash
cd app/build/outputs/apk/fdroid/release

# Generate keystore
keytool -genkey -v -keystore debug.keystore \
    -storepass android -alias androiddebugkey -keypass android \
    -keyalg RSA -keysize 2048 -validity 10000 \
    -dname "CN=Debug, OU=Debug, O=Debug, L=Debug, ST=Debug, C=US"

# Align the APK
zipalign -v -p 4 app-fdroid-release-unsigned.apk app-fdroid-release-aligned.apk

# Sign the APK
apksigner sign --ks debug.keystore --ks-pass pass:android \
    --key-pass pass:android --out app-fdroid-release-signed.apk \
    app-fdroid-release-aligned.apk
```

### Option 2: Use signing.properties (for production)

Create `signing.properties` in the project root:

```properties
STORE_FILE=/path/to/your/keystore.jks
STORE_PASSWORD=your_store_password
KEY_ALIAS=your_key_alias
KEY_PASSWORD=your_key_password
```

Then build normally - the release APK will be signed automatically.

## Installing

```bash
# Via ADB
adb install app-fdroid-release-signed.apk

# Or transfer to device and install manually
```

## Troubleshooting

### NDK not found
- Ensure `ndk.dir` is set in `local.properties`
- Verify the NDK version matches what's in `gradle/libs.versions.toml`

### Rust compilation errors
- Make sure all Android targets are installed: `rustup target list --installed`
- Verify Rust is in PATH: `which cargo`

### jlink not found
- Install full JDK, not just JRE: `sudo apt-get install openjdk-21-jdk-headless`

### License not accepted
- Run: `./cmdline-tools/latest/bin/sdkmanager --licenses`

## Clean Build

```bash
./gradlew clean
./gradlew assembleFdroidRelease
```

## Useful Gradle Tasks

```bash
# List all tasks
./gradlew tasks

# Build all variants
./gradlew assemble

# Run tests
./gradlew test

# Check code style
./gradlew spotlessCheck
```

