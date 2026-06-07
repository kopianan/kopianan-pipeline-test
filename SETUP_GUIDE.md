# Flutter iOS + Android CI/CD Setup Guide

Setup flavor (dev, stage, prod) untuk Flutter project dengan Firebase App Distribution via GitHub Actions + Fastlane.

---

## Daftar Isi

1. [Persiapan Awal](#1-persiapan-awal)
2. [Setup Flutter Flavors](#2-setup-flutter-flavors)
3. [Setup Entry Point per Flavor](#3-setup-entry-point-per-flavor)
4. [Setup VS Code Launch Config](#4-setup-vs-code-launch-config)
5. [Jalankan flutter_flavorizr](#5-jalankan-flutter_flavorizr)
6. [Untrack Firebase Config dari Git](#6-untrack-firebase-config-dari-git)
7. [Setup Firebase Secrets di GitHub](#7-setup-firebase-secrets-di-github)
8. [Setup Fastlane](#8-setup-fastlane)
9. [Setup GitHub Actions Pipeline](#9-setup-github-actions-pipeline)

---

## 1. Persiapan Awal

### Yang dibutuhkan
- Flutter SDK
- Xcode (untuk iOS)
- Apple Developer Account (Team ID: `J3RQ6NYUVH`)
- 3 Firebase Project (dev, stage, prod) — masing-masing punya iOS & Android app
- GitHub repository

### Bundle ID per flavor
| Flavor | Android applicationId | iOS Bundle ID |
|--------|-----------------------|---------------|
| dev    | `com.kopianan.Pipeline.dev` | `com.kopianan.Pipeline.dev` |
| stage  | `com.kopianan.Pipeline.stage` | `com.kopianan.Pipeline.stage` |
| prod   | `com.kopianan.Pipeline` | `com.kopianan.Pipeline` |

---

## 2. Setup Flutter Flavors

### 2.1 Tambah `flutter_flavorizr` ke `pubspec.yaml`

```yaml
dev_dependencies:
  flutter_flavorizr: ^2.2.4
```

### 2.2 Buat `flavorizr.yaml` di root project

```yaml
app:
  android:
    flavorDimensions: "flavor-type"
  ios:
    xcodeproj: Runner.xcodeproj

flavors:
  dev:
    app:
      name: "Pipeline Dev"
    android:
      applicationId: "com.kopianan.Pipeline.dev"
      firebase:
        config: "android/app/src/dev/google-services.json"
    ios:
      bundleId: "com.kopianan.Pipeline.dev"
      firebase:
        config: "ios/Runner/dev/GoogleService-Info.plist"

  stage:
    app:
      name: "Pipeline Stage"
    android:
      applicationId: "com.kopianan.Pipeline.stage"
      firebase:
        config: "android/app/src/stage/google-services.json"
    ios:
      bundleId: "com.kopianan.Pipeline.stage"
      firebase:
        config: "ios/Runner/stage/GoogleService-Info.plist"

  prod:
    app:
      name: "Pipeline"
    android:
      applicationId: "com.kopianan.Pipeline"
      firebase:
        config: "android/app/src/prod/google-services.json"
    ios:
      bundleId: "com.kopianan.Pipeline"
      firebase:
        config: "ios/Runner/prod/GoogleService-Info.plist"
```

---

## 3. Setup Entry Point per Flavor

### 3.1 Buat `lib/main_dev.dart`

```dart
import 'package:flutter/material.dart';
import 'app.dart';
import 'flavors.dart';

void main() {
  F.appFlavor = Flavor.dev;
  runApp(const App());
}
```

### 3.2 Buat `lib/main_stage.dart`

```dart
import 'package:flutter/material.dart';
import 'app.dart';
import 'flavors.dart';

void main() {
  F.appFlavor = Flavor.stage;
  runApp(const App());
}
```

### 3.3 Buat `lib/main_prod.dart`

```dart
import 'package:flutter/material.dart';
import 'app.dart';
import 'flavors.dart';

void main() {
  F.appFlavor = Flavor.prod;
  runApp(const App());
}
```

> **Catatan:** `app.dart` dan `flavors.dart` akan di-generate otomatis oleh `flutter_flavorizr` di Step 5. Buat dulu file entry point ini, nanti import-nya akan resolved setelah flavorizr dijalankan.

---

## 4. Setup VS Code Launch Config

Buat `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Dev",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_dev.dart",
      "args": ["--flavor", "dev"]
    },
    {
      "name": "Stage",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_stage.dart",
      "args": ["--flavor", "stage"]
    },
    {
      "name": "Prod",
      "request": "launch",
      "type": "dart",
      "program": "lib/main_prod.dart",
      "args": ["--flavor", "prod"]
    }
  ]
}
```

### Cara run di VS Code
- Buka panel **Run and Debug** (`Cmd+Shift+D`)
- Pilih flavor dari dropdown: `Dev`, `Stage`, atau `Prod`
- Tekan `F5`

### Cara run via terminal
```bash
flutter run --flavor dev -t lib/main_dev.dart
flutter run --flavor stage -t lib/main_stage.dart
flutter run --flavor prod -t lib/main_prod.dart
```

---

## 5. Jalankan flutter_flavorizr

### 5.1 Buat dummy Firebase config files

Flavorizr butuh file ini untuk bisa jalan. Isi dummy dulu, nanti akan di-untrack dari git di step berikutnya.

```bash
# Buat folder
mkdir -p android/app/src/dev android/app/src/stage android/app/src/prod
mkdir -p ios/Runner/dev ios/Runner/stage ios/Runner/prod

# Dummy Android
cat > android/app/src/dev/google-services.json << 'EOF'
{"project_info":{"project_id":"placeholder-dev"},"client":[{"client_info":{"mobilesdk_app_id":"1:000000000000:android:0000000000000000"},"api_key":[{"current_key":"placeholder"}]}]}
EOF
cp android/app/src/dev/google-services.json android/app/src/stage/google-services.json
cp android/app/src/dev/google-services.json android/app/src/prod/google-services.json

# Dummy iOS
cat > ios/Runner/dev/GoogleService-Info.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>API_KEY</key><string>placeholder</string>
  <key>BUNDLE_ID</key><string>com.kopianan.Pipeline.dev</string>
  <key>CLIENT_ID</key><string>placeholder</string>
  <key>GOOGLE_APP_ID</key><string>1:000000000000:ios:0000000000000000</string>
  <key>GCM_SENDER_ID</key><string>000000000000</string>
  <key>PROJECT_ID</key><string>placeholder-dev</string>
  <key>REVERSED_CLIENT_ID</key><string>placeholder</string>
  <key>STORAGE_BUCKET</key><string>placeholder-dev.appspot.com</string>
</dict>
</plist>
EOF
sed 's/Pipeline.dev/Pipeline.stage/g; s/placeholder-dev/placeholder-stage/g' \
  ios/Runner/dev/GoogleService-Info.plist > ios/Runner/stage/GoogleService-Info.plist
sed 's/Pipeline.dev/Pipeline/g; s/placeholder-dev/placeholder-prod/g' \
  ios/Runner/dev/GoogleService-Info.plist > ios/Runner/prod/GoogleService-Info.plist
```

### 5.2 Jalankan flavorizr

```bash
flutter pub get
dart run flutter_flavorizr
```

Flavorizr akan generate:
- `lib/flavors.dart` — enum Flavor + class F
- `lib/app.dart` — root widget dengan flavor banner (banner muncul di debug mode)
- `ios/Flutter/dev*.xcconfig`, `stage*.xcconfig`, `prod*.xcconfig` — Xcode build config per flavor
- `ios/Runner.xcodeproj/xcshareddata/xcschemes/dev.xcscheme` dll — Xcode schemes
- `android/app/flavorizr.gradle.kts` — Android build variants

### 5.3 Update entry point files

Setelah flavorizr jalan, pastikan `main_dev.dart`, `main_stage.dart`, `main_prod.dart` menggunakan pattern dari flavorizr (pakai `F.appFlavor`, bukan `AppConfig`):

```dart
// Contoh main_dev.dart — sama polanya untuk stage dan prod
import 'package:flutter/material.dart';
import 'app.dart';
import 'flavors.dart';

void main() {
  F.appFlavor = Flavor.dev;
  runApp(const App());
}
```

---

## 6. Untrack Firebase Config dari Git

### 6.1 Tambah ke `.gitignore`

```
# Firebase config files — injected at CI/CD from GitHub Secrets, never committed
android/app/src/dev/google-services.json
android/app/src/stage/google-services.json
android/app/src/prod/google-services.json
ios/Runner/dev/GoogleService-Info.plist
ios/Runner/stage/GoogleService-Info.plist
ios/Runner/prod/GoogleService-Info.plist
```

### 6.2 Untrack dummy files yang sudah terlanjur di-stage

```bash
git rm --cached android/app/src/dev/google-services.json \
                android/app/src/stage/google-services.json \
                android/app/src/prod/google-services.json \
                ios/Runner/dev/GoogleService-Info.plist \
                ios/Runner/stage/GoogleService-Info.plist \
                ios/Runner/prod/GoogleService-Info.plist

git commit -m "chore: untrack firebase config files from git"
```

> File dummy tetap ada di local disk untuk keperluan build lokal. Hanya tidak di-track git.

### 6.3 Replace dummy dengan file asli di local

Download dari Firebase Console lalu replace:
- `android/app/src/dev/google-services.json`
- `android/app/src/stage/google-services.json`
- `android/app/src/prod/google-services.json`
- `ios/Runner/dev/GoogleService-Info.plist`
- `ios/Runner/stage/GoogleService-Info.plist`
- `ios/Runner/prod/GoogleService-Info.plist`

File asli ini tidak akan ke-commit karena sudah ada di `.gitignore`.

---

## 7. Setup Firebase Secrets di GitHub

### 7.1 Encode file ke base64

```bash
# Android
base64 -i android/app/src/dev/google-services.json | pbcopy    # → FIREBASE_ANDROID_DEV
base64 -i android/app/src/stage/google-services.json | pbcopy  # → FIREBASE_ANDROID_STAGE
base64 -i android/app/src/prod/google-services.json | pbcopy   # → FIREBASE_ANDROID_PROD

# iOS
base64 -i ios/Runner/dev/GoogleService-Info.plist | pbcopy      # → FIREBASE_IOS_DEV
base64 -i ios/Runner/stage/GoogleService-Info.plist | pbcopy    # → FIREBASE_IOS_STAGE
base64 -i ios/Runner/prod/GoogleService-Info.plist | pbcopy     # → FIREBASE_IOS_PROD
```

### 7.2 Tambahkan ke GitHub Secrets

Buka repo GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret | Keterangan |
|--------|-----------|
| `FIREBASE_ANDROID_DEV` | base64 dari `google-services.json` dev |
| `FIREBASE_ANDROID_STAGE` | base64 dari `google-services.json` stage |
| `FIREBASE_ANDROID_PROD` | base64 dari `google-services.json` prod |
| `FIREBASE_IOS_DEV` | base64 dari `GoogleService-Info.plist` dev |
| `FIREBASE_IOS_STAGE` | base64 dari `GoogleService-Info.plist` stage |
| `FIREBASE_IOS_PROD` | base64 dari `GoogleService-Info.plist` prod |
| `FIREBASE_APP_ID_ANDROID_DEV` | Firebase Android App ID dev (format: `1:xxx:android:xxx`) |
| `FIREBASE_APP_ID_ANDROID_STAGE` | Firebase Android App ID stage |
| `FIREBASE_APP_ID_ANDROID_PROD` | Firebase Android App ID prod |
| `FIREBASE_APP_ID_IOS_DEV` | Firebase iOS App ID dev (format: `1:xxx:ios:xxx`) |
| `FIREBASE_APP_ID_IOS_STAGE` | Firebase iOS App ID stage |
| `FIREBASE_APP_ID_IOS_PROD` | Firebase iOS App ID prod |
| `FIREBASE_SERVICE_ACCOUNT_JSON` | Service account JSON untuk upload ke Firebase |
| `FIREBASE_TESTERS` | Email tester, contoh: `a@test.com, b@test.com` |

---

## 8. Setup Fastlane

Update `fastlane/Fastfile`:

```ruby
default_platform(:android)

platform :android do
  desc "Build & distribute Android ke Firebase per flavor"
  lane :build_and_distribute_firebase do |options|
    flavor = options[:flavor] || "dev"

    sh(
      command: "flutter build apk --debug --flavor #{flavor} -t lib/main_#{flavor}.dart --build-number=#{ENV['BUILD_NUMBER']}",
      log: true
    )

    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID_ANDROID_#{flavor.upcase}"],
      android_artifact_path: "build/app/outputs/flutter-apk/app-#{flavor}-debug.apk",
      android_artifact_type: "APK",
      testers: ENV["FIREBASE_TESTERS"],
      service_credentials_json_data: ENV["FIREBASE_SERVICE_ACCOUNT_JSON"]
    )

    UI.success("✅ Android #{flavor} distributed to Firebase!")
  end
end

platform :ios do
  desc "Build & distribute iOS ke Firebase per flavor"
  lane :build_and_distribute_firebase do |options|
    flavor = options[:flavor] || "dev"

    match(
      type: "adhoc",
      app_identifier: "com.kopianan.Pipeline#{flavor == 'prod' ? '' : ".#{flavor}"}",
      git_url: ENV["MATCH_GIT_URL"],
      readonly: true
    )

    sh(
      command: "flutter build ipa --release --flavor #{flavor} -t lib/main_#{flavor}.dart --export-method=ad-hoc --build-number=#{ENV['BUILD_NUMBER']}",
      log: true
    )

    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID_IOS_#{flavor.upcase}"],
      ipa_path: "build/ios/ipa/Runner.ipa",
      testers: ENV["FIREBASE_TESTERS"],
      service_credentials_json_data: ENV["FIREBASE_SERVICE_ACCOUNT_JSON"]
    )

    UI.success("✅ iOS #{flavor} distributed to Firebase!")
  end
end
```

---

## 9. Setup GitHub Actions Pipeline

### Branch → Flavor mapping

| Branch push | Flavor yang di-build |
|-------------|---------------------|
| `dev`       | dev |
| `stage`     | stage |
| `main`      | prod |

### `.github/workflows/main.yml`

```yaml
name: Build & Distribute

on:
  push:
    branches:
      - dev
      - stage
      - main

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      flavor: ${{ steps.set-flavor.outputs.flavor }}
    steps:
      - name: Set flavor from branch
        id: set-flavor
        run: |
          if [ "${{ github.ref_name }}" = "main" ]; then
            echo "flavor=prod" >> $GITHUB_OUTPUT
          else
            echo "flavor=${{ github.ref_name }}" >> $GITHUB_OUTPUT
          fi

  build_android:
    needs: setup
    runs-on: ubuntu-latest
    env:
      FLAVOR: ${{ needs.setup.outputs.flavor }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: 3.44.0
          cache: true

      - uses: actions/cache@v4
        with:
          path: ~/.gradle/caches
          key: ${{ runner.os }}-gradle-${{ hashFiles('android/gradle/wrapper/gradle-wrapper.properties') }}
          restore-keys: ${{ runner.os }}-gradle-

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
          bundler-cache: true

      - name: Flutter pub get
        run: flutter pub get

      - name: Inject Firebase Android config
        run: |
          echo "${{ secrets[format('FIREBASE_ANDROID_{0}', env.FLAVOR_UPPER)] }}" \
            | base64 --decode > android/app/src/${{ env.FLAVOR }}/google-services.json
        env:
          FLAVOR_UPPER: ${{ needs.setup.outputs.flavor == 'dev' && 'DEV' || needs.setup.outputs.flavor == 'stage' && 'STAGE' || 'PROD' }}

      - name: Build & Distribute Android
        run: bundle exec fastlane android build_and_distribute_firebase flavor:${{ env.FLAVOR }}
        env:
          FIREBASE_APP_ID_ANDROID_DEV: ${{ secrets.FIREBASE_APP_ID_ANDROID_DEV }}
          FIREBASE_APP_ID_ANDROID_STAGE: ${{ secrets.FIREBASE_APP_ID_ANDROID_STAGE }}
          FIREBASE_APP_ID_ANDROID_PROD: ${{ secrets.FIREBASE_APP_ID_ANDROID_PROD }}
          FIREBASE_TESTERS: ${{ secrets.FIREBASE_TESTERS }}
          FIREBASE_SERVICE_ACCOUNT_JSON: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          BUILD_NUMBER: ${{ github.run_number }}

  build_ios:
    needs: setup
    runs-on: macos-latest
    env:
      FLAVOR: ${{ needs.setup.outputs.flavor }}
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: 3.44.0
          cache: true

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
          bundler-cache: true

      - name: Flutter pub get
        run: flutter pub get

      - name: Inject Firebase iOS config
        run: |
          echo "${{ secrets[format('FIREBASE_IOS_{0}', env.FLAVOR_UPPER)] }}" \
            | base64 --decode > ios/Runner/${{ env.FLAVOR }}/GoogleService-Info.plist
        env:
          FLAVOR_UPPER: ${{ needs.setup.outputs.flavor == 'dev' && 'DEV' || needs.setup.outputs.flavor == 'stage' && 'STAGE' || 'PROD' }}

      - name: Build & Distribute iOS
        run: bundle exec fastlane ios build_and_distribute_firebase flavor:${{ env.FLAVOR }}
        env:
          FIREBASE_APP_ID_IOS_DEV: ${{ secrets.FIREBASE_APP_ID_IOS_DEV }}
          FIREBASE_APP_ID_IOS_STAGE: ${{ secrets.FIREBASE_APP_ID_IOS_STAGE }}
          FIREBASE_APP_ID_IOS_PROD: ${{ secrets.FIREBASE_APP_ID_IOS_PROD }}
          FIREBASE_TESTERS: ${{ secrets.FIREBASE_TESTERS }}
          FIREBASE_SERVICE_ACCOUNT_JSON: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
          BUILD_NUMBER: ${{ github.run_number }}
```

---

## Ringkasan Flow

```
Developer push ke branch dev/stage/main
          ↓
GitHub Actions trigger
  ├── Job: setup → tentukan flavor dari nama branch
  ├── Job: build_android (ubuntu-latest) — paralel
  │     ├── Decode secret → android/app/src/{flavor}/google-services.json
  │     ├── flutter pub get
  │     ├── fastlane android build_and_distribute_firebase flavor:{flavor}
  │     └── Upload APK ke Firebase App Distribution
  └── Job: build_ios (macos-latest) — paralel
        ├── Decode secret → ios/Runner/{flavor}/GoogleService-Info.plist
        ├── flutter pub get
        ├── fastlane ios build_and_distribute_firebase flavor:{flavor}
        └── Upload IPA ke Firebase App Distribution
          ↓
Tester terima notifikasi email → download build
```

---

## Yang di-commit ke git ✅ vs ❌

### Commit ✅
| File/Folder | Keterangan |
|-------------|-----------|
| `flavorizr.yaml` | Config flavorizr |
| `lib/main_dev.dart`, `main_stage.dart`, `main_prod.dart` | Entry point per flavor |
| `lib/app.dart`, `lib/flavors.dart` | Di-generate flavorizr |
| `ios/Flutter/*.xcconfig` | Xcode build config per flavor |
| `ios/Runner.xcodeproj/xcshareddata/xcschemes/*.xcscheme` | Xcode schemes |
| `android/app/flavorizr.gradle.kts` | Android build variants |
| `android/app/src/{flavor}/res/` | Icon per flavor |
| `.vscode/launch.json` | VS Code run config |
| `.github/workflows/main.yml` | CI/CD pipeline |
| `fastlane/Fastfile` | Fastlane lanes |

### Jangan commit ❌
| File/Folder | Keterangan |
|-------------|-----------|
| `android/app/src/*/google-services.json` | Firebase Android config |
| `ios/Runner/*/GoogleService-Info.plist` | Firebase iOS config |
