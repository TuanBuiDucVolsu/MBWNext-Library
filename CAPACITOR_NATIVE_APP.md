# POS Next — Native App (Capacitor)

Hướng dẫn build/chạy/test bản native (iOS/Android) của POS Next, đóng gói bằng Capacitor.

Kiến trúc: **full native bundle** — toàn bộ Vue app được build cục bộ vào app (không load từ URL remote), nên app luôn mở được kể cả mất mạng hoàn toàn khi khởi động. Chi tiết thiết kế/lý do xem trong lịch sử trao đổi hoặc hỏi lại khi cần; file này chỉ tập trung vào **cách chạy**.

> **Nhánh làm việc:** toàn bộ code Capacitor (native app) nằm trên nhánh **`vnpost`** của repo `MBW-Digital/mbwnext-pos` — không phải `develop` hay `ha_vang`.

## 1. Yêu cầu môi trường

- Node/yarn như frontend web bình thường (`POS/package.json`)
- Android SDK (`ANDROID_HOME`), khuyến nghị cài qua Android Studio
- **JDK 21** để build Gradle — nếu máy chỉ có JDK 17/11 hệ thống, dùng JBR đi kèm Android Studio:
  ```bash
  export JAVA_HOME=/snap/android-studio/232/jbr   # đường dẫn có thể khác tuỳ bản Android Studio
  ```
- iOS chỉ build được trên **macOS + Xcode** (không làm được trên Linux)

## 2. Cấu hình server đích (`.env.capacitor`)

File `POS/.env.capacitor` quyết định app native sẽ nói chuyện với site Frappe nào. Có 2 biến:

```bash
VITE_POS_SERVER_URL=...   # base URL tuyệt đối của server (bắt buộc)
VITE_POS_SITE_NAME=...    # tên site folder, chỉ cần khi base URL không trùng hostname site
```

**Vì sao cần `VITE_POS_SITE_NAME` riêng:** Frappe multi-tenant chọn site dựa vào `Host` header hoặc header `X-Frappe-Site-Name` (`frappe/app.py`). Khi base URL là domain thật (production) thì hostname = site name, không cần khai báo thêm. Nhưng khi base URL là IP/port (bench dev), site name phải khai riêng để app tự đính kèm header `X-Frappe-Site-Name` vào mọi request (xem `POS/src/utils/native/serverConfig.js`, `POS/src/main.js`).

**Production (VNPost):**
```bash
VITE_POS_SERVER_URL=https://vnpost.mbwnext.com
```

**Local dev — test qua Android Emulator (AVD):**
```bash
VITE_POS_SERVER_URL=http://10.0.2.2:8040   # 10.0.2.2 = localhost của máy host, nhìn từ trong AVD
VITE_POS_SITE_NAME=vnp.com
```

**Local dev — test bằng thiết bị Android thật, cùng mạng LAN với máy dev:**
```bash
VITE_POS_SERVER_URL=http://192.168.1.44:8040   # IP LAN của máy dev, xem bằng `hostname -I`
VITE_POS_SITE_NAME=vnp.com
```
`10.0.2.2` **chỉ resolve được trong Android Emulator**, thiết bị thật sẽ không kết nối được tới địa chỉ này — phải đổi sang IP LAN thật của máy dev. Nếu IP LAN đổi (đổi mạng wifi, DHCP cấp lại IP...) phải sửa lại `.env.capacitor` và build lại.

Đổi xong phải build lại (bước 3) — giá trị được nhúng cứng vào bundle lúc build, không đọc runtime.

## 3. Build & sync

```bash
cd POS
yarn build:capacitor        # build ra POS/dist-capacitor/
npx cap sync android        # copy bundle + plugin config vào android/
# npx cap sync ios          # tương tự cho iOS (cần macOS)
```

## 4. Build APK (debug)

```bash
cd POS/android
export ANDROID_HOME=/home/mbw12345/Android/Sdk
export JAVA_HOME=/snap/android-studio/232/jbr   # nếu JDK hệ thống < 21
./gradlew :app:assembleDebug
```
APK ra tại `android/app/build/outputs/apk/debug/app-debug.apk`.

> **Lưu ý cleartext HTTP:** `android/app/src/debug/AndroidManifest.xml` bật `usesCleartextTraffic` **chỉ cho debug build**, để test được với bench local (`http://...`). Bản release build cho production (`https://vnpost.mbwnext.com`) không bị ảnh hưởng, vẫn HTTPS-only mặc định.

## 5. Test trên Android Emulator (AVD)

### Tạo AVD lần đầu (chỉ cần làm 1 lần trên máy dev)
```bash
export ANDROID_HOME=/home/mbw12345/Android/Sdk
export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/emulator:$ANDROID_HOME/platform-tools:$PATH"

sdkmanager --licenses                                                    # accept hết (gõ y liên tục)
sdkmanager "system-images;android-34;google_apis;x86_64" "platform-tools" "emulator"
avdmanager create avd -n pos_next_test -k "system-images;android-34;google_apis;x86_64" -d "pixel_5"
```

### Chạy emulator (cửa sổ thật, thao tác trực tiếp bằng chuột/bàn phím)
```bash
export ANDROID_HOME=/home/mbw12345/Android/Sdk
export PATH="$ANDROID_HOME/emulator:$ANDROID_HOME/platform-tools:$PATH"

emulator -avd pos_next_test -gpu swiftshader_indirect -no-snapshot
```
- `emulator -list-avds` — xem danh sách AVD hiện có
- Bỏ `-no-snapshot` nếu muốn boot nhanh hơn ở lần sau (dùng quick-boot snapshot)
- Test airplane mode: mở **Extended controls** (nút `...` cạnh cửa sổ emulator) → **Cellular** → **None**

### Tắt emulator khi xong (đỡ nặng máy)
```bash
adb -s emulator-5554 emu kill
adb kill-server   # tuỳ chọn, tắt luôn adb daemon
```

## 6. Test trên thiết bị Android thật (USB / cùng LAN Wi-Fi)

Dùng khi cần test trên phần cứng thật (camera, máy in, hiệu năng thật...) thay vì emulator.

### Chuẩn bị điện thoại (chỉ cần làm 1 lần)
1. **Settings → About phone** → bấm liên tục vào **Build number** ~7 lần để mở khoá **Developer options**
2. **Settings → Developer options** → bật **USB debugging**

### Kết nối qua USB
```bash
export PATH="/home/mbw12345/Android/Sdk/platform-tools:$PATH"
adb devices -l   # cắm cáp, xác nhận popup "Allow USB debugging" trên điện thoại, chạy lại lệnh này tới khi thấy device "device" (không phải "unauthorized")
```

### Hoặc kết nối qua Wi-Fi (không cần cáp, phải cùng mạng LAN)
```bash
adb tcpip 5555                        # chạy 1 lần khi máy còn đang cắm cáp
adb connect 192.168.1.xx:5555         # IP của điện thoại (Settings → About phone → Status → IP address)
adb devices -l                        # xác nhận thấy device qua wifi
```

### Build đúng cấu hình rồi cài
1. Đảm bảo `.env.capacitor` đang trỏ IP LAN của máy dev (xem mục 2, phần "thiết bị Android thật"), **không phải** `10.0.2.2`
2. Build lại: `yarn build:capacitor && npx cap sync android` (bước 3) rồi build APK (bước 4)
3. Cài & chạy: xem mục 7

> **Lưu ý mạng:** điện thoại và máy dev phải cùng mạng LAN/wifi và không bị cô lập bởi "AP/client isolation" của router. Nếu bench chỉ bind `127.0.0.1` thay vì `0.0.0.0`, thiết bị ngoài sẽ không kết nối được — kiểm tra bằng `ss -tlnp | grep 8040` trên máy dev, cột địa chỉ phải là `0.0.0.0:8040`.

## 7. Cài & chạy app, xem log

```bash
export PATH="/home/mbw12345/Android/Sdk/platform-tools:$PATH"

adb install -r POS/android/app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n com.mbwdigital.posnext/.MainActivity

# Xem log JS/network của app (rất hữu ích để debug thay vì chỉ nhìn màn hình)
adb logcat -d | grep -iE "posnext|capacitor"

# Chụp màn hình (lưu ý: emulator -no-window có thể chụp ra ảnh splash tĩnh
# thay vì frame WebView thật — không phải app bị treo, xem mục 8)
adb shell screencap -p /sdcard/screen.png && adb pull /sdcard/screen.png .
```
Nếu có nhiều device/emulator cùng lúc, thêm `-s <device_id>` vào các lệnh `adb` (xem id bằng `adb devices -l`).

## 8. Các vướng mắc đã gặp & cách xử lý

- **`compileSdk`/`platforms;android-36` báo lỗi tải:** SDK Platform 36 có thể bị cache dở dang. Chạy lại `sdkmanager "platforms;android-36"` (cần mạng) hoặc mở Android Studio → SDK Manager để tự sửa.
- **`invalid source release: 21` khi build Gradle:** JDK hệ thống thấp hơn 21 → set `JAVA_HOME` trỏ vào JBR của Android Studio (xem mục 1/4). Khi mở project bằng Android Studio (không dùng `gradlew` tay) thì không gặp lỗi này vì Android Studio tự dùng JBR nội bộ.
- **Emulator chạy `-no-window` (headless) rồi chụp màn hình chỉ thấy logo Capacitor mặc định (hình X xanh):** đây là `splash.png` mặc định của template Capacitor (chưa thay icon/splash thương hiệu — việc branding icon để sau), bị kẹt do giới hạn compositor của chế độ headless, **không phải app bị treo** — xác nhận app thật sự chạy bằng `adb logcat` (xem mục 7) thay vì chỉ tin ảnh chụp màn hình. Muốn thấy hình thật, chạy emulator **có cửa sổ** (bỏ `-no-window`, xem mục 5) trên máy có màn hình thật.
- **Không có `avdmanager`/`sdkmanager`:** máy chỉ cài Android Studio (GUI) chứ chưa có gói `cmdline-tools`. Tải thủ công:
  ```bash
  curl -o /tmp/cmdline-tools.zip -L "https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip"
  mkdir -p $ANDROID_HOME/cmdline-tools/tmp_extract
  unzip -q /tmp/cmdline-tools.zip -d $ANDROID_HOME/cmdline-tools/tmp_extract
  mv $ANDROID_HOME/cmdline-tools/tmp_extract/cmdline-tools $ANDROID_HOME/cmdline-tools/latest
  rmdir $ANDROID_HOME/cmdline-tools/tmp_extract
  ```
- **Thiết bị thật `adb devices` báo `unauthorized`:** chưa bấm "Allow" trên popup xác nhận RSA key của điện thoại — rút cáp cắm lại, hoặc kiểm tra màn hình điện thoại có đang khoá popup không.
- **Thiết bị thật cài được nhưng app không load được gì (trắng màn hình/lỗi network):** thường do `.env.capacitor` vẫn còn để `10.0.2.2` (giá trị chỉ dùng cho emulator) — đổi sang IP LAN thật rồi build lại (xem mục 2 và 6).
- **Cài đè debug ↔ release lỗi `INSTALL_FAILED_UPDATE_INCOMPATIBLE`:** debug và release build ký bằng key khác nhau nhưng cùng `applicationId` (`com.mbwdigital.posnext`) — phải gỡ bản đang cài (`adb uninstall com.mbwdigital.posnext`) trước khi cài bản kia.

## 9. Build production (release) — ký app & phát hành

### 9.1. Chuyển cấu hình sang server production
Sửa `POS/.env.capacitor`:
```bash
VITE_POS_SERVER_URL=https://vnpost.mbwnext.com
# bỏ/comment VITE_POS_SITE_NAME — hostname domain thật đã trùng site name
```
Rồi build & sync lại (mục 3): `yarn build:capacitor && npx cap sync android`.

### 9.2. Bump version
Trong `android/app/build.gradle` (`defaultConfig`), tăng trước mỗi lần phát hành:
```gradle
versionCode 2        // số nguyên, PHẢI tăng mỗi lần nộp Store/gửi bản mới
versionName "1.1"     // chuỗi hiển thị cho người dùng
```

### 9.3. Tạo keystore ký app (chỉ làm 1 lần cho vòng đời app)
```bash
keytool -genkey -v -keystore pos-next-release.keystore \
  -alias pos_next -keyalg RSA -keysize 2048 -validity 10000
```
> **Quan trọng:** lưu file `.keystore` này + toàn bộ mật khẩu (store password, key password) ở nơi an toàn (password manager/vault nội bộ). **Mất file này thì không thể phát hành bản cập nhật cho app đã có trên Store nữa** — phải phát hành app mới với package name khác. Tuyệt đối không commit file này vào git.

### 9.4. Khai báo signing config cho Gradle
Tạo `android/keystore.properties` (thêm vào `.gitignore`, không commit):
```properties
storeFile=/duong/dan/tuyet/doi/toi/pos-next-release.keystore
storePassword=...
keyAlias=pos_next
keyPassword=...
```

Thêm vào đầu `android/app/build.gradle` (trước block `android { ... }`):
```gradle
def keystorePropertiesFile = rootProject.file("keystore.properties")
def keystoreProperties = new Properties()
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}
```

Và bên trong block `android { ... }`, thêm `signingConfigs` + gán cho `release`:
```gradle
signingConfigs {
    release {
        if (keystorePropertiesFile.exists()) {
            storeFile file(keystoreProperties['storeFile'])
            storePassword keystoreProperties['storePassword']
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
        }
    }
}
buildTypes {
    release {
        signingConfig signingConfigs.release
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
}
```

### 9.5. Build bản release
```bash
cd POS/android
export ANDROID_HOME=/home/mbw12345/Android/Sdk
export JAVA_HOME=/snap/android-studio/232/jbr

./gradlew :app:assembleRelease   # APK đã ký, cài trực tiếp qua adb install
./gradlew :app:bundleRelease     # AAB — bắt buộc nếu nộp lên Google Play
```
Output:
- APK: `android/app/build/outputs/apk/release/app-release.apk`
- AAB: `android/app/build/outputs/bundle/release/app-release.aab`

### 9.6. Cài thử bản release trước khi phát hành
```bash
adb uninstall com.mbwdigital.posnext   # gỡ bản debug trước (khác chữ ký, xem mục 8)
adb install -r android/app/build/outputs/apk/release/app-release.apk
```
Xác nhận app kết nối đúng `https://vnpost.mbwnext.com` (không phải bench local) — xem log `adb logcat` (mục 7) hoặc thử một thao tác cần gọi API thật.

### 9.7. Icon & splash thương hiệu (chưa làm, cần trước khi phát hành thật)
Hiện vẫn dùng icon/splash mặc định của template Capacitor. Khi có asset thật (icon nguồn ≥1024×1024px, nền trong suốt), dùng:
```bash
cd POS
npx @capacitor/assets generate --android
npx cap sync android
```
rồi build lại release (mục 9.5).

## 10. Sửa code production: backend vs frontend — hệ quả khác nhau

Vì kiến trúc là **full native bundle** (mục kiến trúc ở đầu file), server production và app đã cài trên máy khách là 2 thứ **tách biệt hoàn toàn** về mặt phân phối code. Tuỳ code sửa ở đâu mà quy trình cập nhật khác hẳn nhau:

### 10.1. Sửa backend Python (`pos_next/api/...`, doctype, hooks, `overrides/`...)
App native gọi các phần này qua API/network, không có gì nhúng cứng trong bundle — **không cần build lại app**. Chỉ cần deploy code lên server production như một app Frappe bình thường:
```bash
# trên server production
git pull                          # nhánh vnpost/ha_vang
bench build --app pos_next        # rebuild static assets (cho bản web, không ảnh hưởng app native)
bench migrate                     # nếu có thay đổi doctype/fixtures
# restart theo cơ chế server đang dùng (supervisor/systemd)
```
App khách gọi API lần tiếp theo sẽ thấy ngay code mới, không cần làm gì trên điện thoại.

### 10.2. Sửa frontend Vue (`POS/src/...`)
Toàn bộ Vue app được build cứng vào file cài đặt lúc `yarn build:capacitor` — sửa server **không** có tác dụng gì với app khách đã cài. Bắt buộc phải phát hành **bản app mới**:
```bash
cd POS
yarn build:capacitor && npx cap sync android
cd android
./gradlew :app:bundleRelease   # ký bằng keystore release, xem mục 9.3–9.5
```
rồi phân phối cho khách qua Google Play (upload `.aab`, tăng `versionCode`, mục 9.2) hoặc gửi APK release để khách tự cài đè thủ công nếu chưa lên Store — **không** dùng `adb install` vì đó là máy khách, không cắm dây với máy dev.

### 10.3. Tóm tắt

| Sửa ở đâu | Cần build lại app? | Khách cần làm gì? |
|---|---|---|
| Backend Python/API/doctype | Không | Không — tự động thấy code mới |
| Frontend Vue (`POS/src`) | Có — bản release mới | Cập nhật app (Play Store hoặc cài APK mới) |
