# Release APK — Hướng dẫn build

## Môi trường

| Thành phần | Phiên bản |
|---|---|
| OS | macOS 26.3 (arm64 / Apple Silicon) |
| Flutter | 3.41.7 (stable) |
| Java | OpenJDK 17.0.13 LTS |

---

## Chuẩn bị một lần (first-time setup)

### 1. Tạo keystore

```bash
keytool -genkey -v \
  -keystore flutter/android/app/rustdesk.jks \
  -alias rustdesk \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -storepass 123456 \
  -keypass 123456 \
  -dname "CN=RustDesk, OU=Dev, O=XSofts, L=HCM, ST=HCM, C=VN"
```

> ⚠️ Backup file `rustdesk.jks` cẩn thận. Mất keystore = không thể update app trên store.

### 2. Tạo file `flutter/android/key.properties`

```properties
storePassword=123456
keyPassword=123456
keyAlias=rustdesk
storeFile=rustdesk.jks
```

> `storeFile` là đường dẫn tương đối từ `android/app/` (không phải từ `android/`).

---

## Build release APK

```bash
cd flutter/
flutter build apk --target-platform android-arm64 --release
```

**Output:** `flutter/build/app/outputs/flutter-apk/app-release.apk` (~53MB)

### Build tách APK theo ABI (nhỏ hơn, khuyến nghị cho distribution)

```bash
flutter build apk --split-per-abi --target-platform android-arm64,android-arm --release
```

---

## Xác minh chữ ký

```bash
~/Library/Android/sdk/build-tools/36.0.0/apksigner verify --verbose \
  flutter/build/app/outputs/flutter-apk/app-release.apk
```

Kết quả mong đợi:
```
Verifies
Verified using v2 scheme (APK Signature Scheme v2): true
```

---

## So sánh debug vs release

| | Debug | Release |
|---|---|---|
| Size | ~122MB | ~53MB |
| Ký | debug key tự động | `rustdesk.jks` |
| Minify | Không | Có (ProGuard) |
| Dùng cho | Phát triển / test | Phân phối |

---

## Lỗi thường gặp

### `Keystore file '...app/app/rustdesk.jks' not found`

`storeFile` trong `key.properties` bị double path. Sửa lại thành `storeFile=rustdesk.jks` (không có prefix `app/`).

### `validateSigningRelease` failed

Kiểm tra `key.properties` tồn tại tại `flutter/android/key.properties` và các giá trị đúng.

### Gradle heap space khi build release

Tăng heap trong `flutter/android/gradle.properties`:
```properties
org.gradle.jvmargs=-Xmx4096M
```
