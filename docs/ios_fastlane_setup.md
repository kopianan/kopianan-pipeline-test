# iOS Fastlane Setup — Firebase App Distribution

Panduan lengkap setup Fastlane untuk build dan distribusi iOS ke Firebase App Distribution menggunakan GitHub Actions dengan manual code signing.

---

## Daftar Isi

1. [Prasyarat](#1-prasyarat)
2. [Membuat Certificate Signing Request (CSR)](#2-membuat-certificate-signing-request-csr)
3. [Membuat Distribution Certificate di Apple Developer Portal](#3-membuat-distribution-certificate-di-apple-developer-portal)
4. [Export Certificate ke .p12](#4-export-certificate-ke-p12)
5. [Membuat Provisioning Profile (Ad Hoc)](#5-membuat-provisioning-profile-ad-hoc)
6. [ExportOptions.plist](#6-exportoptionsplist)
7. [Setup GitHub Secrets](#7-setup-github-secrets)
8. [Fastfile — iOS Lane](#8-fastfile--ios-lane)
9. [GitHub Actions Workflow](#9-github-actions-workflow)
10. [Alur Kerja Keseluruhan](#10-alur-kerja-keseluruhan)

---

## 1. Prasyarat

Sebelum memulai, pastikan hal berikut sudah siap:

| Kebutuhan | Keterangan |
|---|---|
| Xcode | Versi terbaru dari App Store |
| Apple Developer Account | Akun berbayar ($99/tahun) |
| Bundle ID terdaftar | Sudah dibuat di Apple Developer Portal |
| Firebase Project | Sudah ada app iOS di Firebase Console |
| Repository GitHub | Tempat menyimpan kode dan menjalankan CI/CD |

---

## 2. Membuat Certificate Signing Request (CSR)

**Apa itu CSR?**
CSR adalah file permintaan yang kamu kirim ke Apple, berisi informasi identitas kamu. Apple akan memverifikasi dan mengeluarkan sertifikat resmi berdasarkan CSR ini. Analoginya seperti formulir pendaftaran yang kamu isi sebelum mendapat KTP.

**Langkah-langkah:**

1. Buka **Keychain Access** (cari via Spotlight: `Cmd + Space` → ketik "Keychain Access")
2. Di menu bar atas → **Keychain Access** → **Certificate Assistant** → **Request a Certificate From a Certificate Authority**
3. Isi form:
   - **User Email Address** → email Apple ID kamu
   - **Common Name** → nama bebas, contoh: `MyApp Distribution`
   - **CA Email Address** → **kosongkan**
   - Pilih **Saved to disk**
4. Klik **Continue** → pilih lokasi simpan → **Save**
5. File `CertificateSigningRequest.certSigningRequest` tersimpan

---

## 3. Membuat Distribution Certificate di Apple Developer Portal

**Apa itu Distribution Certificate?**
Sertifikat ini adalah tanda tangan digital resmi dari Apple yang membuktikan bahwa IPA yang kamu build berasal dari developer yang sah. Setiap IPA yang ingin diinstall di device **wajib** ditandatangani dengan sertifikat ini. Tanpa sertifikat, Apple menolak IPA tersebut.

**Langkah-langkah:**

1. Login ke [developer.apple.com](https://developer.apple.com)
2. Masuk ke **Certificates, Identifiers & Profiles**
3. Tab **Certificates** → klik tombol **+** (tambah)
4. Pilih **Apple Distribution** → klik **Continue**

   > Pilih **Apple Distribution** karena berlaku untuk Ad Hoc dan App Store sekaligus.

5. Klik **Choose File** → upload file `CertificateSigningRequest.certSigningRequest` tadi
6. Klik **Continue** → klik **Download**
7. File `distribution.cer` tersimpan ke komputer kamu

**Install `.cer` ke Keychain:**

1. Double-click file `distribution.cer`
2. Otomatis terbuka Keychain Access dan sertifikat terinstall
3. Buka Keychain Access → tab **My Certificates**
4. Pastikan muncul entry **Apple Distribution: Nama Kamu (TEAM_ID)**

---

## 4. Export Certificate ke `.p12`

**Apa itu file `.p12`?**
File `.p12` adalah bundle terenkripsi yang berisi sertifikat + private key secara bersamaan. File inilah yang nantinya diupload ke GitHub Secrets dan digunakan oleh CI/CD untuk menandatangani IPA. Format `.p12` dipilih karena bisa diproteksi dengan password dan mudah dipindahkan antar mesin.

**Langkah-langkah:**

1. Buka **Keychain Access** → tab **My Certificates**
2. Temukan **Apple Distribution: Nama Kamu**
3. Klik kanan → **Export "Apple Distribution: ..."**
4. Pilih format **Personal Information Exchange (.p12)**
5. Beri nama file, contoh: `distribution.p12` → klik **Save**
6. **Set password** untuk file ini → catat passwordnya, akan dipakai di GitHub Secrets
7. Masukkan **login password Mac** jika diminta untuk konfirmasi

> ⚠️ Jaga baik-baik file `.p12` dan passwordnya. Siapapun yang punya keduanya bisa menandatangani app atas nama kamu.

---

## 5. Membuat Provisioning Profile (Ad Hoc)

**Apa itu Provisioning Profile?**
Provisioning Profile adalah file izin dari Apple yang mengikat tiga hal sekaligus:
- **Bundle ID** app kamu
- **Distribution Certificate** yang dipakai
- **Daftar device** yang boleh menginstall app (khusus Ad Hoc)

Tanpa file ini, iOS menolak untuk menginstall IPA meskipun sertifikatnya valid. Analoginya seperti STNK kendaraan — membuktikan bahwa kendaraan (app) ini legal dan boleh dipakai di jalan (device) tertentu.

**Kenapa Ad Hoc?**
Ad Hoc adalah metode distribusi yang memungkinkan IPA diinstall di device-device terdaftar tanpa melalui App Store. Cocok untuk distribusi ke tester via Firebase App Distribution.

**Langkah-langkah:**

1. Login ke [developer.apple.com](https://developer.apple.com) → **Certificates, Identifiers & Profiles**
2. Tab **Profiles** → klik tombol **+**
3. Di bagian **Distribution** → pilih **Ad Hoc** → klik **Continue**
4. Pilih **App ID** sesuai Bundle ID app kamu → klik **Continue**
5. Pilih **Distribution Certificate** yang baru dibuat → klik **Continue**
6. Tambahkan **device UDID** tester yang akan menerima build
   > UDID device bisa didapat via Xcode atau [udid.io](https://udid.io)
7. Beri nama profile, contoh: `MyApp AdHoc` → klik **Generate**
8. Klik **Download** → file `AdHoc.mobileprovision` tersimpan

---

## 6. ExportOptions.plist

**Apa itu ExportOptions.plist?**
Ketika `flutter build ipa` dijalankan, Flutter memanggil perintah Xcode di balik layar:

```
xcodebuild -exportArchive -exportOptionsPlist ExportOptions.plist
```

File ini adalah instruksi eksplisit untuk Xcode tentang **bagaimana** cara mengemas IPA:
- Mau didistribusi via metode apa (Ad Hoc, App Store, dll)
- Provisioning profile mana yang dipakai
- Apakah signing dilakukan manual atau otomatis

Tanpa file ini, `flutter build ipa` akan gagal karena Xcode tidak tahu instruksi packaging-nya.

**Isi file `ios/ExportOptions.plist`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>ad-hoc</string>

    <key>provisioningProfiles</key>
    <dict>
        <key>com.kopianan.Pipeline</key>
        <string>AdHoc</string>
    </dict>

    <key>signingStyle</key>
    <string>manual</string>

    <key>teamID</key>
    <string>$(IOS_TEAM_ID)</string>

    <key>stripSwiftSymbols</key>
    <true/>

    <key>uploadBitcode</key>
    <false/>

    <key>uploadSymbols</key>
    <true/>
</dict>
</plist>
```

**Penjelasan tiap key:**

| Key | Value | Penjelasan |
|---|---|---|
| `method` | `ad-hoc` | Jenis distribusi. `ad-hoc` untuk Firebase App Distribution |
| `provisioningProfiles` | Bundle ID → Nama Profile | Mengikat Bundle ID ke provisioning profile yang dipakai |
| `signingStyle` | `manual` | Xcode tidak mencari sertifikat otomatis, pakai yang sudah disiapkan |
| `teamID` | Team ID Apple | Identifikasi Apple Developer Team kamu |
| `stripSwiftSymbols` | `true` | Kurangi ukuran IPA dengan menghapus Swift debug symbols |
| `uploadBitcode` | `false` | Bitcode tidak diperlukan untuk Ad Hoc |
| `uploadSymbols` | `true` | Upload crash symbols untuk debugging di Firebase Crashlytics |

---

## 7. Setup GitHub Secrets

**Encode file ke Base64:**

Jalankan di Terminal dari folder tempat file `.p12` dan `.mobileprovision` disimpan:

```bash
# Encode .p12 dan copy ke clipboard
base64 -i distribution.p12 | pbcopy
# Paste hasilnya sebagai secret IOS_P12_BASE64

# Encode provisioning profile dan copy ke clipboard
base64 -i AdHoc.mobileprovision | pbcopy
# Paste hasilnya sebagai secret IOS_PROVISIONING_PROFILE_BASE64
```

**Tambahkan secrets di GitHub:**

Buka repo GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret Name | Cara Mendapatkan |
|---|---|
| `IOS_P12_BASE64` | Hasil encode `distribution.p12` |
| `IOS_P12_PASSWORD` | Password yang diset saat export `.p12` |
| `IOS_PROVISIONING_PROFILE_BASE64` | Hasil encode `AdHoc.mobileprovision` |
| `IOS_TEAM_ID` | [developer.apple.com](https://developer.apple.com) → Account → Membership → Team ID |
| `FIREBASE_APP_ID_IOS` | Firebase Console → Project Settings → pilih app iOS → App ID |
| `FIREBASE_TESTERS` | Email tester, pisahkan dengan koma |
| `FIREBASE_SERVICE_ACCOUNT_JSON` | Google Cloud → Service Account JSON (sudah ada dari setup Android) |

---

## 8. Fastfile — iOS Lane

Tambahkan platform iOS ke `fastlane/Fastfile`:

```ruby
platform :ios do
  desc "Build IPA and distribute to Firebase App Distribution"
  lane :build_and_distribute_firebase do
    # Setup keychain sementara di environment CI
    setup_ci if ENV["CI"]

    # Import sertifikat ke keychain
    import_certificate(
      certificate_path: "distribution.p12",
      certificate_password: ENV["IOS_P12_PASSWORD"],
      keychain_name: ENV["MATCH_KEYCHAIN_NAME"],
      keychain_password: ENV["MATCH_KEYCHAIN_PASSWORD"]
    )

    # Install provisioning profile
    install_provisioning_profile(
      path: "AdHoc.mobileprovision"
    )

    # Build IPA dengan Flutter
    sh(
      command: "flutter build ipa --release --build-number=#{ENV['BUILD_NUMBER']} --export-options-plist=../ios/ExportOptions.plist",
      log: true
    )

    ipa_path = "../build/ios/ipa/*.ipa"

    # Upload ke Firebase App Distribution
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID_IOS"],
      ipa_path: ipa_path,
      testers: ENV["FIREBASE_TESTERS"],
      service_credentials_json_data: ENV["FIREBASE_SERVICE_ACCOUNT_JSON"]
    )

    UI.success("✅ iOS build completed and distributed to Firebase!")
  end
end
```

**Penjelasan tiap step di lane:**

| Step | Penjelasan |
|---|---|
| `setup_ci` | Membuat temporary keychain di CI agar sertifikat bisa diinstall tanpa interaksi user |
| `import_certificate` | Memasukkan `.p12` ke keychain CI supaya Xcode bisa menggunakannya untuk signing |
| `install_provisioning_profile` | Menempatkan `.mobileprovision` di lokasi yang dikenali Xcode |
| `flutter build ipa` | Build app dan kemas menjadi file IPA sesuai instruksi di `ExportOptions.plist` |
| `firebase_app_distribution` | Upload IPA ke Firebase dan kirim notifikasi ke tester |

---

## 9. GitHub Actions Workflow

File: `.github/workflows/ios.yml`

```yaml
name: iOS Build & Distribute

on:
  push:
    branches:
      - main
      - develop
  workflow_dispatch:
    inputs:
      build_number:
        description: "Build Number"
        required: false
        default: "1"

jobs:
  build-and-distribute:
    runs-on: macos-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: "3.x"
          channel: "stable"

      - name: Install Flutter dependencies
        run: flutter pub get

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.2"
          bundler-cache: true

      - name: Decode & install certificate
        env:
          IOS_P12_BASE64: ${{ secrets.IOS_P12_BASE64 }}
        run: |
          echo $IOS_P12_BASE64 | base64 --decode > distribution.p12

      - name: Decode & install provisioning profile
        env:
          IOS_PROVISIONING_PROFILE_BASE64: ${{ secrets.IOS_PROVISIONING_PROFILE_BASE64 }}
        run: |
          echo $IOS_PROVISIONING_PROFILE_BASE64 | base64 --decode > AdHoc.mobileprovision

      - name: Run Fastlane iOS
        env:
          BUILD_NUMBER: ${{ github.run_number }}
          IOS_P12_PASSWORD: ${{ secrets.IOS_P12_PASSWORD }}
          IOS_TEAM_ID: ${{ secrets.IOS_TEAM_ID }}
          FIREBASE_APP_ID_IOS: ${{ secrets.FIREBASE_APP_ID_IOS }}
          FIREBASE_TESTERS: ${{ secrets.FIREBASE_TESTERS }}
          FIREBASE_SERVICE_ACCOUNT_JSON: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          MATCH_KEYCHAIN_NAME: "build.keychain"
          MATCH_KEYCHAIN_PASSWORD: "temp_keychain_password"
        run: bundle exec fastlane ios build_and_distribute_firebase
```

**Kenapa `runs-on: macos-latest`?**
Build iOS **wajib** dijalankan di macOS karena Xcode hanya tersedia di sistem operasi Apple. GitHub Actions menyediakan runner macOS untuk kebutuhan ini.

---

## 10. Alur Kerja Keseluruhan

```
Push ke branch main/develop
        │
        ▼
GitHub Actions trigger
        │
        ▼
Setup Flutter & Ruby
        │
        ▼
Decode secrets → distribution.p12 + AdHoc.mobileprovision
        │
        ▼
Fastlane: import_certificate + install_provisioning_profile
        │
        ▼
flutter build ipa (Xcode baca ExportOptions.plist)
        │
        ▼
Fastlane: firebase_app_distribution
        │
        ▼
Tester dapat notifikasi email & bisa download IPA
```

---

## Troubleshooting

| Error | Penyebab | Solusi |
|---|---|---|
| `No signing certificate found` | `.p12` tidak terimport dengan benar | Cek `IOS_P12_BASE64` dan `IOS_P12_PASSWORD` di secrets |
| `No provisioning profile found` | Profile tidak terinstall | Cek `IOS_PROVISIONING_PROFILE_BASE64` di secrets |
| `exportArchive failed` | `ExportOptions.plist` salah konfigurasi | Pastikan nama profile di plist sama persis dengan nama di Developer Portal |
| `firebase_app_distribution failed` | Service account tidak valid | Cek `FIREBASE_SERVICE_ACCOUNT_JSON` dan pastikan app ID benar |
| `Team ID not found` | `IOS_TEAM_ID` kosong atau salah | Cek di developer.apple.com → Account → Membership |
