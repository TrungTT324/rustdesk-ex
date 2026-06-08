# Build Changes — Android APK & macOS

Ghi lại tất cả thay đổi cần thiết để build thành công APK debug (Android) và macOS app trên môi trường hiện tại.

## Môi trường

| Thành phần | Phiên bản |
|---|---|
| OS | macOS 26.3 (arm64 / Apple Silicon) |
| Flutter | 3.41.7 (stable) |
| Dart | 3.11.5 |
| Rust | stable-aarch64-apple-darwin |
| Cargo | 1.89.0 |
| Java | OpenJDK 17.0.13 LTS |
| flutter_rust_bridge_codegen | 1.80.1 |

## Bước chuẩn bị (chạy một lần)

### 1. Khởi tạo git submodule

`libs/hbb_common` là submodule và cần được pull về trước:

```bash
git submodule update --init --recursive
```

### 2. Tạo lại file Dart FFI bridge

File `flutter/lib/generated_bridge.dart` và `flutter/lib/bridge_definitions.dart` phải được sinh ra từ Rust source. Cài tool và chạy:

```bash
cargo install flutter_rust_bridge_codegen --version 1.80.1

cd /path/to/rustdesk-ex
flutter_rust_bridge_codegen \
  --rust-input src/flutter_ffi.rs \
  --dart-output flutter/lib/generated_bridge.dart \
  --dart-decl-output flutter/lib/bridge_definitions.dart \
  --class-name Rustdesk
```

### 3. Fix export trong generated_bridge.dart

Sau khi codegen, mở `flutter/lib/generated_bridge.dart` và thêm dòng `export` ngay trước `import`:

```diff
+export "bridge_definitions.dart";
 import "bridge_definitions.dart";
```

Lý do: codegen chỉ sinh `import` nhưng không `export`, khiến các type như `EventToUI`, `EventToUI_Event` không truyền ra ngoài cho các file khác import.

---

## Thay đổi file cấu hình Android

### `flutter/android/gradle/wrapper/gradle-wrapper.properties`

```diff
-distributionUrl=https\://services.gradle.org/distributions/gradle-7.6.4-all.zip
+distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-all.zip
```

**Lý do:** Flutter 3.41.7's `FlutterPlugin.kt` dùng API `filePermissions` chỉ có từ Gradle 8.3+.

---

### `flutter/android/settings.gradle`

```diff
-    id "com.android.application" version "7.3.1" apply false
+    id "com.android.application" version "8.1.4" apply false
```

**Lý do:** Flutter 3.41.7 yêu cầu Android Gradle Plugin (AGP) tối thiểu 8.1.1.

---

### `flutter/android/app/build.gradle`

**Thay đổi 1 — Sửa đường dẫn `cargo metadata`:**

```diff
-        it.workingDir = new File("../..")
+        it.workingDir = new File(projectDir, "../../..")
```

`../..` từ `android/app/` trỏ vào `flutter/` (không có `Cargo.toml`). Đúng phải là `../../..` = root `rustdesk-ex/`.

**Thay đổi 2 — Thêm `namespace` và nâng `compileSdk`:**

```diff
 android {
+    namespace "com.carriez.flutter_hbb"
-    compileSdkVersion 34
+    compileSdkVersion 36
```

- `namespace`: bắt buộc từ AGP 8.x
- `compileSdk 36`: yêu cầu bởi `sqflite_android` và `flutter_plugin_android_lifecycle` phiên bản mới

---

### `flutter/android/build.gradle`

Thêm block tự động gán `namespace` cho các plugin legacy (không có `namespace` trong build.gradle của chúng):

```groovy
// Fix: set namespace for legacy plugins that lack it (required by AGP 8.x)
subprojects {
    plugins.withId("com.android.library") {
        def manifestFile = project.file("src/main/AndroidManifest.xml")
        def ns = manifestFile.exists() ?
            new groovy.xml.XmlParser().parse(manifestFile).attribute("package") : null
        if (!android.namespace) {
            android.namespace = ns ?: project.group
        }
    }
}
```

**Lý do:** AGP 8.x bắt buộc tất cả library module phải có `namespace`. Nhiều plugin Flutter cũ chưa có.

---

### `flutter/android/gradle.properties`

```diff
-org.gradle.jvmargs=-Xmx1024M
+org.gradle.jvmargs=-Xmx4096M
 android.useAndroidX=true
 android.enableJetifier=true
 org.gradle.daemon=false
+kotlin.jvm.target.validation.mode=IGNORE
```

- **Tăng heap 4096M:** Jetifier hết bộ nhớ khi xử lý Flutter engine JAR (~122MB)
- **`kotlin.jvm.target.validation.mode=IGNORE`:** Các plugin cũ dùng Kotlin JVM target 1.8 trong khi Kotlin 2.1.21 mặc định dùng 17 → bỏ qua validation này

---

## Thay đổi Flutter dependencies (`flutter/pubspec.yaml`)

### Dependencies chính

| Package | Trước | Sau | Lý do |
|---|---|---|---|
| `external_path` | `^1.0.3` | `^2.2.0` | v1.x thiếu `namespace` cho AGP 8.x |
| `file_picker` | `^5.1.0` | `^8.0.0` | v5.x dùng v1 plugin embedding đã bị xóa |
| `extended_text` | `14.0.0` | `^15.0.2` | v14 thiếu implement `SelectionHandler.contentLength` / `getSelection` (Flutter 3.x) |
| `sqflite` | `2.2.0` | `^2.4.2+1` | v2.2.x dùng v1 plugin embedding đã bị xóa |
| `google_fonts` | `^6.2.1` | `^6.2.2` | v6.2.1 lỗi `const` map với `FontWeight` trong Dart 3.x |

### Dependency overrides

| Package | Trước | Sau | Lý do |
|---|---|---|---|
| `flutter_plugin_android_lifecycle` | `2.0.17` | `2.0.35` | v2.0.17 dùng v1 embedding (`PluginRegistry.Registrar`) đã bị xóa khỏi Flutter engine |

---

## Thay đổi Dart source code

### `flutter/lib/common.dart`

```diff
-    dialogTheme: DialogTheme(
+    dialogTheme: DialogThemeData(
```

```diff
-    tabBarTheme: const TabBarTheme(
+    tabBarTheme: const TabBarThemeData(
```

(2 lần xuất hiện mỗi cái — cả light theme và dark theme)

**Lý do:** Flutter 3.27+ đổi tên class `DialogTheme` → `DialogThemeData` và `TabBarTheme` → `TabBarThemeData`.

---

### `flutter/lib/models/native_model.dart` (dòng 157)

```diff
-          _homeDir = (await ExternalPath.getExternalStorageDirectories())[0];
+          _homeDir = (await ExternalPath.getExternalStorageDirectories())![0];
```

**Lý do:** `external_path` 2.x đổi return type thành `List<String>?` (nullable) → cần `!` để unwrap.

---

## Lệnh build

```bash
cd flutter/
flutter pub get
flutter build apk --target-platform android-arm64 --debug
```

APK output: `flutter/build/app/outputs/flutter-apk/app-debug.apk` (~122MB)

---

## Build & Run macOS

### Bước 1 — Build Rust native library (release)

Xcode project tham chiếu đến `target/release/liblibrustdesk.dylib`:

```bash
# từ root rustdesk-ex/
cargo build --lib --release --features flutter
```

Thời gian build: ~4–5 phút lần đầu. Output: `target/release/liblibrustdesk.dylib` (~21MB).

### Bước 2 — Generate C header cho macOS

Cần thêm `--c-output` vào lệnh codegen (khác với Android):

```bash
flutter_rust_bridge_codegen \
  --rust-input src/flutter_ffi.rs \
  --dart-output flutter/lib/generated_bridge.dart \
  --dart-decl-output flutter/lib/bridge_definitions.dart \
  --c-output flutter/macos/Runner/bridge_generated.h \
  --class-name Rustdesk
```

Sau đó thêm lại `export` vào `generated_bridge.dart` (codegen overwrite):
```diff
+export "bridge_definitions.dart";
 import "bridge_definitions.dart";
```

### Bước 3 — Build và chạy Flutter macOS

```bash
cd flutter/
flutter build macos --debug
open build/macos/Build/Products/Debug/RustDesk.app

# Hoặc chạy trực tiếp với hot-reload:
flutter run -d macos
```

### Thay đổi macOS-specific

#### `flutter/macos/Runner/AppDelegate.swift`

```diff
+  override func applicationSupportsSecureRestorableState(_ app: NSApplication) -> Bool {
+    return true
+  }
+
   override func applicationShouldTerminateAfterLastWindowClosed(_ sender: NSApplication) -> Bool {
```

**Lý do:** macOS yêu cầu override này để tránh cảnh báo migration khi build.

#### `flutter/macos/Runner/MainFlutterWindow.swift`

```diff
-import sqflite
+import sqflite_darwin
```

**Lý do:** `sqflite` 2.4.x đổi tên CocoaPod từ `sqflite` → `sqflite_darwin`. Class `SqflitePlugin` vẫn giữ nguyên.

---

## Cảnh báo còn lại (không ảnh hưởng build)

- AGP 8.1.4 sắp bị Flutter drop support, khuyến nghị nâng lên ≥ 8.6.0
- SDK XML version 4 warning (do Android SDK tools mới hơn SDK manager)
- `file_picker` warning về platform plugin implementation trên Linux/macOS/Windows
- macOS warning về `CVPixelBufferRef _Nullable` trong texture_rgba_renderer (chỉ là warning, không lỗi)
