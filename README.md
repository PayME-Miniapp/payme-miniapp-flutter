# PayME Miniapp Flutter

**Tiếng Việt** | [English](README.en.md)

Plugin Flutter `payme_miniapp_flutter` giúp ứng dụng của đối tác mở PayME Miniapp (ví, thanh toán, nạp, rút, chuyển tiền, dịch vụ, KYC) trên iOS và Android.

| Thành phần | Phiên bản |
| --- | --- |
| `payme_miniapp_flutter` (pub.dev) | **3.0.1** |
| PayME Miniapp iOS (CocoaPods `PayME-Miniapp`) | 0.8.3 |
| PayME Miniapp Android (`com.github.PayME-Miniapp:native-android`) | 0.8.4 |
| PayME SDK | 1.0.24 |

> [!IMPORTANT]
> 3.0.0 là bản không tương thích ngược với 2.x: mọi chỗ trước đây nhận số điện thoại (`phone`) nay nhận `identityToken`. Xem [Nâng cấp từ 2.x](#nâng-cấp-từ-2x).

> [!WARNING]
> **3.0.1 đổi cách báo lỗi của các hàm.** `init`, `open`, `close`, `getBalance`, `getAccountInfo`, `setLanguage` hoàn tất với `null` khi thành công và **ném `PlatformException`** khi lỗi (thiếu/sai tham số, chưa `init`...). Trước 3.0.1, plugin tự bắt lỗi, chỉ in `debugPrint` và trả `null`. Hãy bọc các lời gọi bằng `try/catch`; xem [Lỗi phía plugin](#lỗi-phía-plugin-platformexception).

## Mục lục

- [Yêu cầu](#yêu-cầu)
- [Cài đặt](#cài-đặt)
  - [Thêm thư viện](#thêm-thư-viện)
  - [iOS: Podfile](#ios-podfile)
  - [Android: JitPack và Gradle](#android-jitpack-và-gradle)
- [Cấu hình ứng dụng](#cấu-hình-ứng-dụng)
  - [iOS: Info.plist](#ios-infoplist)
  - [iOS: Capabilities](#ios-capabilities)
  - [Android: quyền, Activity và ProGuard](#android-quyền-activity-và-proguard)
- [Bắt đầu nhanh](#bắt-đầu-nhanh)
- [identityToken](#identitytoken)
- [Tham chiếu API](#tham-chiếu-api)
  - [Lưu ý chung về Future](#lưu-ý-chung-về-future)
  - [init](#init)
  - [open](#open)
  - [PayMEOpeningParam](#paymeopeningparam)
  - [Các lớp tham số](#các-lớp-tham-số)
  - [getBalance](#getbalance)
  - [getAccountInfo](#getaccountinfo)
  - [setLanguage](#setlanguage)
  - [close](#close)
  - [onResponse và onError](#onresponse-và-onerror)
  - [Hằng số](#hằng-số)
- [Mã lỗi](#mã-lỗi)
  - [Lỗi nhận qua onError](#lỗi-nhận-qua-onerror)
  - [Lỗi phía plugin (PlatformException)](#lỗi-phía-plugin-platformexception)
- [Nâng cấp từ 2.x](#nâng-cấp-từ-2x)
- [Lịch sử thay đổi và hỗ trợ](#lịch-sử-thay-đổi-và-hỗ-trợ)

## Yêu cầu

| Hạng mục | Yêu cầu |
| --- | --- |
| Dart SDK | `>=3.2.0 <4.0.0` |
| Flutter | `>=3.3.0` (phiên bản Flutter phải build được với Android Gradle Plugin 9, xem bên dưới) |
| iOS | iOS 13.0 trở lên, **Xcode 26.2 trở lên**, CocoaPods |
| Android | `minSdkVersion` 26, `targetSdkVersion` 33 trở lên, `compileSdkVersion` 37 |
| Android build | JDK 17, Gradle 9.3.1, Android Gradle Plugin 9.1.1, Kotlin Gradle Plugin 2.2.20, NDK 28.2.13676358 |
| Tài khoản | `appId`, `publicKey`, `privateKey` do PayME cấp; server của đối tác gọi được API cấp `identityToken` của PayME |

## Cài đặt

### Thêm thư viện

```sh
flutter pub add payme_miniapp_flutter
```

hoặc khai báo trong `pubspec.yaml`:

```yaml
dependencies:
  payme_miniapp_flutter: ^3.0.1
```

### iOS: Podfile

Plugin kéo pod `PayME-Miniapp` 0.8.3. Một số pod phụ thuộc phải build dưới dạng dynamic framework. Cập nhật `ios/Podfile`:

```ruby
platform :ios, '13.0'

target 'Runner' do
  use_frameworks!
  use_modular_headers!

  flutter_install_all_ios_pods File.dirname(File.realpath(__FILE__))

  $dynamic_frameworks = [
    'PayME-Miniapp', 'CryptoSwift', 'SwiftyRSA', 'GCDWebServer', 'NSLogger',
    'lottie-ios', 'SwiftyJSON', 'ZIPFoundation', 'Mixpanel-swift'
  ]

  pre_install do |installer|
    installer.pod_targets.each do |pod|
      if $dynamic_frameworks.include?(pod.name)
        def pod.build_type
          Pod::BuildType.dynamic_framework
        end
      end
    end
  end

  post_install do |installer|
    installer.pods_project.targets.each do |target|
      flutter_additional_ios_build_settings(target)
      target.build_configurations.each do |config|
        if $dynamic_frameworks.include?(target.name)
          config.build_settings['BUILD_LIBRARY_FOR_DISTRIBUTION'] = 'YES'
          config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '13.0'
        end
      end
    end
  end
end
```

Sau đó chạy:

```sh
cd ios && pod install --repo-update
```

### Android: JitPack và Gradle

**1. Thêm JitPack.** Thư viện native Android được phát hành qua JitPack, nên project ứng dụng phải khai báo repository này.

Nếu project khai báo repository trong `android/build.gradle`:

```groovy
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url "https://jitpack.io" }
    }
}
```

Nếu project dùng `dependencyResolutionManagement` trong `android/settings.gradle`:

```groovy
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}
```

**2. Android Gradle Plugin và Kotlin.** Với template Flutter mới, trong `android/settings.gradle`:

```groovy
plugins {
    id "dev.flutter.flutter-plugin-loader" version "1.0.0"
    id "com.android.application" version "9.1.1" apply false
    id "org.jetbrains.kotlin.android" version "2.2.20" apply false
}
```

Với template cũ khai báo trong `android/build.gradle`:

```groovy
buildscript {
    ext.kotlin_version = "2.2.20"
    dependencies {
        classpath "com.android.tools.build:gradle:9.1.1"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}
```

**3. Gradle Wrapper** (`android/gradle/wrapper/gradle-wrapper.properties`):

```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-9.3.1-all.zip
```

**4. Module app** (`android/app/build.gradle`):

```groovy
android {
    compileSdkVersion 37
    ndkVersion = "28.2.13676358"

    defaultConfig {
        minSdkVersion 26
        targetSdkVersion flutter.targetSdkVersion // phải từ 33 trở lên
    }
}
```

**5. `android/gradle.properties`:**

```properties
android.useAndroidX=true
android.enableJetifier=true
android.builtInKotlin=false
android.newDsl=false
android.uniquePackageNames=false
```

`android.builtInKotlin=false` và `android.newDsl=false` giữ project build bằng Kotlin Gradle Plugin trong giai đoạn Flutter và AGP 9 chuyển sang Built-in Kotlin. `android.uniquePackageNames=false` là cách xử lý tạm thời cho lỗi trùng namespace giữa `com.google.ai.edge.litert:litert-support` và `com.google.ai.edge.litert:litert-support-api` khi build với AGP 9.

## Cấu hình ứng dụng

### iOS: Info.plist

Thêm các key sau vào `ios/Runner/Info.plist`:

![Info.plist](./documents/info_plist.png)

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>vcbpartner</string>
</array>
<key>NSCameraUsageDescription</key>
<string>Ứng dụng cần dùng máy ảnh để định danh và quét mã thanh toán</string>
<key>NSContactsUsageDescription</key>
<string>Ứng dụng cần truy cập danh bạ để chọn nhanh số điện thoại</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>Ứng dụng cần quyền lưu ảnh mã QR vào thư viện ảnh</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Ứng dụng cần truy cập thư viện ảnh để chọn mã QR</string>
<key>NFCReaderUsageDescription</key>
<string>Ứng dụng cần dùng NFC để đọc căn cước công dân khi định danh (eKYC)</string>
<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
  <string>A0000002471001</string>
  <string>A0000002472001</string>
  <string>00000000000000</string>
</array>
```

| Key | Mục đích |
| --- | --- |
| `LSApplicationQueriesSchemes` | Mở app ngân hàng Vietcombank (`vcbpartner`) khi nạp tiền qua deeplink |
| `NSCameraUsageDescription` | Chụp giấy tờ, khuôn mặt khi KYC; quét mã QR |
| `NSContactsUsageDescription` | Chọn số trong danh bạ khi nạp tiền điện thoại |
| `NSPhotoLibraryUsageDescription` | Chọn ảnh mã QR từ thư viện |
| `NSPhotoLibraryAddUsageDescription` | Lưu ảnh mã QR vào thư viện |
| `NFCReaderUsageDescription` | Đọc chip căn cước công dân qua NFC khi KYC |
| `com.apple.developer.nfc.readersession.iso7816.select-identifiers` | Mã ứng dụng ISO 7816 cho phiên đọc NFC |

### iOS: Capabilities

Trong Xcode, chọn target `Runner` > **Signing & Capabilities** > nhấn **+ Capability**, rồi thêm:

1. **Near Field Communication Tag Reading** (KYC qua NFC).

   ![Near Field Communication Tag Reading](./documents/capabilities2.png)

SDK không cần **Background Modes** (Background fetch) và không cần xử lý `handleEventsForBackgroundURLSession` trong `AppDelegate`: bản cập nhật Miniapp được tải trong lúc mở, không tải nền.

### Android: quyền, Activity và ProGuard

- **Quyền:** thư viện native đã khai báo sẵn các quyền cần thiết (Internet, máy ảnh, danh bạ, NFC, ...) trong manifest của nó; Gradle tự gộp vào ứng dụng. Không cần khai báo thêm.
- **Activity:** `PayMEOpeningType.screen` chạy với `FlutterActivity`. `PayMEOpeningType.modal` cần `MainActivity` kế thừa `FlutterFragmentActivity`; nếu không, `open` ném `PlatformException` `MINIAPP_CANNOT_PRESENT` (Miniapp không mở).

  ```kotlin
  import io.flutter.embedding.android.FlutterFragmentActivity

  class MainActivity : FlutterFragmentActivity()
  ```

- **ProGuard / R8:** nếu bật `minifyEnabled true`, thêm vào `android/app/proguard-rules.pro`:

```proguard
-keep class vn.kalapa.ekyc.**{*;}
-keep class com.fis.ekyc.**{*;}
-keep class com.fis.nfc.**{*;}
-keep class retrofit2.**{*;}

-keepclassmembers,allowobfuscation class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
```

## Bắt đầu nhanh

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';
import 'package:payme_miniapp_flutter/payme_miniapp_flutter.dart';
import 'package:payme_miniapp_flutter/payme_miniapp_type.dart';

/// Gọi server của bạn để lấy identityToken (xem mục identityToken).
Future<String> fetchIdentityToken() async => throw UnimplementedError();

Future<void> setUpPayME() async {
  PaymeMiniappFlutter.onResponse.listen((event) => debugPrint('PayME response: ${event['action']}'));
  PaymeMiniappFlutter.onError.listen((event) => debugPrint('PayME error: $event'));
  try {
    await PaymeMiniappFlutter.init('<APP_ID>', '<PUBLIC_KEY>', '<PRIVATE_KEY>',
        env: PayMEEnv.SANDBOX); // không truyền mode: SANDBOX dùng PayMEMode.MINIAPP_SANDBOX
  } on PlatformException catch (e) {
    debugPrint('PayME init lỗi ${e.code}: ${e.message}');
  }
}

Future<void> openWallet() async {
  final identityToken = await fetchIdentityToken();
  try {
    await PaymeMiniappFlutter.open(
        PayMEOpeningType.screen, PayMEOpeningParam.open(identityToken: identityToken));
  } on PlatformException catch (e) {
    debugPrint('PayME open lỗi ${e.code}: ${e.message}');
  }
}
```

Gọi `setUpPayME()` một lần khi ứng dụng khởi động (ví dụ trong `initState` của màn hình gốc). Kết quả và lỗi nghiệp vụ (từ SDK) trả về qua hai stream `onResponse` và `onError`; lỗi tham số hoặc gọi khi chưa `init` làm Future ném `PlatformException`.

## identityToken

`identityToken` là chuỗi do backend PayME ký cho một người dùng của đối tác. Từ 3.0.0, Miniapp xác định người dùng bằng chuỗi này thay cho số điện thoại.

**Luồng 3 bước**

1. **Server đối tác lấy chuỗi.** Server của bạn gọi API `identityToken/init` của PayME (server tới server) với số điện thoại của người dùng. Liên hệ PayME để nhận tài liệu API này. Không gọi API này từ ứng dụng.
2. **Ứng dụng nhận chuỗi.** Ứng dụng lấy `identityToken` từ server của bạn qua kênh đã xác thực người dùng.
3. **Mở Miniapp.** Truyền chuỗi vào `PayMEOpeningParam` (hoặc `getBalance`, `getAccountInfo`).

**Thời hạn**

- Chuỗi có hiệu lực **15 phút** kể từ khi cấp và dùng lại được nhiều lần trong thời gian đó.
- Chuỗi sai hoặc hết hạn: `onError` trả `code` `"401"`. Lấy chuỗi mới từ server rồi gọi lại.
- Chuỗi rỗng ở thao tác bắt buộc: `onError` trả `code` `"IDENTITY_TOKEN_REQUIRED"`.

**Khi nào cần**

| Thao tác | identityToken |
| --- | --- |
| `PayMEOpeningParam.payme()` | Không dùng. Người dùng tự nhập số điện thoại và đăng nhập bằng PIN. |
| `PayMEOpeningParam.payment(...)` | Tuỳ chọn. Có: trả được bằng ví PayME. Không có: thanh toán như khách vãng lai (thẻ, QR). |
| Các `PayMEOpeningParam` còn lại, `getBalance`, `getAccountInfo` | Bắt buộc |

**Bảo mật**

- Chỉ giữ chuỗi trong bộ nhớ. Không ghi log, không lưu xuống máy (SharedPreferences, file, cơ sở dữ liệu), không đưa lên URL hay analytics.
- Không nhúng `privateKey` hay thông tin gọi API `identityToken/init` vào ứng dụng; chỉ server của bạn giữ chúng.

## Tham chiếu API

Mọi hàm là `static` trên lớp `PaymeMiniappFlutter` (`package:payme_miniapp_flutter/payme_miniapp_flutter.dart`). Các lớp tham số và hằng số nằm trong `package:payme_miniapp_flutter/payme_miniapp_type.dart`.

### Lưu ý chung về Future

- Các hàm trả về `Future<dynamic>` nhưng **không mang dữ liệu**: thành công thì hoàn tất với `null`. Kết quả và lỗi nghiệp vụ đến qua [`onResponse` và `onError`](#onresponse-và-onerror).
- Khi lỗi phía plugin (thiếu/sai tham số, chưa `init`...), Future **ném `PlatformException`** với `code` ở bảng [Lỗi phía plugin](#lỗi-phía-plugin-platformexception). Bọc lời gọi bằng `try { ... } on PlatformException catch (e) { ... }`. Trước 3.0.1, plugin tự bắt lỗi và trả `null`.
- Các hàm là `async`: lỗi Dart khi tạo tham số (ví dụ `extraData` không encode được JSON) cũng trả qua Future, không ném ngay lúc gọi.
- Từ 3.0.1, mọi lời gọi đều hoàn tất trên cả iOS và Android (trước đó một số Future không bao giờ hoàn tất). Future hoàn tất nghĩa là lệnh đã được chuyển cho SDK, không có nghĩa thao tác (ví dụ thanh toán) đã xong.

### init

```dart
static Future<dynamic> init(String appId, String publicKey, String privateKey,
    {String? locale = PayMELocale.VI,
    String? env = PayMEEnv.PRODUCTION,
    String? mode});
```

| Tham số | Bắt buộc | Kiểu | Mặc định | Mô tả |
| --- | --- | --- | --- | --- |
| `appId` | Có | `String` | | Mã ứng dụng PayME cấp (giá trị `x-api-client` trên dashboard) |
| `publicKey` | Có | `String` | | Public key RSA PayME cấp, viết trên một dòng (bỏ ký tự xuống dòng) |
| `privateKey` | Có | `String` | | Private key RSA của đối tác, viết trên một dòng |
| `locale` | Không | `String` | `PayMELocale.VI` | Ngôn ngữ ban đầu: `PayMELocale.VI`, `PayMELocale.EN` |
| `env` | Không | `String` | `PayMEEnv.PRODUCTION` | `PayMEEnv.SANDBOX` hoặc `PayMEEnv.PRODUCTION` |
| `mode` | Không | `String` | Theo `env` | Kênh cập nhật Miniapp. Không truyền (hoặc `null`): `PayMEEnv.SANDBOX` dùng `PayMEMode.MINIAPP_SANDBOX`, còn lại dùng `PayMEMode.MINIAPP_PRODUCT`. Truyền thì giữ nguyên giá trị đã truyền |

`appId`, `publicKey` hoặc `privateKey` rỗng: ném `PlatformException` `MISSING_INFO`. `locale`, `env`, `mode` là `null` thì dùng giá trị mặc định. Gọi lại `init` với cùng `appId`/`publicKey`/`privateKey` chỉ cập nhật `env`/`mode`/`locale`, miniapp đang mở vẫn `close()` được; trên iOS, đổi key thì miniapp đang mở bị đóng trước.

> [!WARNING]
> Không commit key thật vào mã nguồn public, README hay ảnh chụp màn hình. Dùng bộ key PayME cấp riêng cho từng môi trường.

### open

```dart
static Future<dynamic> open(String type, PayMEOpeningParam openingParam);
```

| Tham số | Bắt buộc | Kiểu | Mô tả |
| --- | --- | --- | --- |
| `type` | Có | `String` | `PayMEOpeningType.screen` (toàn màn hình) hoặc `PayMEOpeningType.modal` (trượt từ dưới lên). Không phân biệt hoa thường; giá trị khác được hiểu là `screen`. Android: `modal` cần `FlutterFragmentActivity` (xem [Activity](#android-quyền-activity-và-proguard)). |
| `openingParam` | Có | `PayMEOpeningParam` | Thao tác cần mở, tạo bằng một trong các named constructor bên dưới |

```dart
Future<void> payOrder(String identityToken) async {
  try {
    await PaymeMiniappFlutter.open(
      PayMEOpeningType.screen,
      PayMEOpeningParam.pay(
        identityToken: identityToken,
        data: PayMEPaymentParam(amount: 10000, transactionId: '<PARTNER_ORDER_ID>'),
      ),
    );
  } on PlatformException catch (e) {
    // INVALID_PARAM, PAYME_NOT_INITIALIZED, ...: Miniapp không mở
    debugPrint('PayME open lỗi ${e.code}: ${e.message}');
  }
}
```

`open` ném `PlatformException` khi: thiếu `action` (`MISSING_INFO`), action không hỗ trợ hoặc `data` thiếu/sai (`INVALID_PARAM`), chưa `init` (`PAYME_NOT_INITIALIZED`), hoặc không hiển thị được Miniapp (`MINIAPP_CANNOT_PRESENT`: iOS không có view controller để present; Android mở `modal` khi chưa có Activity, `MainActivity` không kế thừa `FlutterFragmentActivity` hoặc Activity đã lưu trạng thái). Khi đó Miniapp không mở.

#### Mở, đóng và nút back

- **Mỗi lúc chỉ một Miniapp.** Gọi `open` khi đang có Miniapp mở (iOS: hoặc đang kiểm tra/tải bản cập nhật trước khi mở): Future vẫn hoàn tất, `onError` trả `MINIAPP_ALREADY_OPEN`, Miniapp đang mở giữ nguyên.
- **Mở lại ngay trong callback đóng** (`USER_CANCEL`, hoặc kết quả làm Miniapp đóng) hay ngay sau `close()` được. iOS hoãn lần mở đến khi Miniapp cũ đã đóng hẳn; Android mở ngay.
- **Một kết quả đóng mỗi phiên.** Mỗi lần mở chỉ có một kết quả đóng: `USER_CANCEL`, hoặc `onResponse`/`onError` làm Miniapp đóng. Miniapp chỉ bị che (máy ảnh eKYC, webview phụ) rồi hiện lại thì không phát `USER_CANCEL`.
- **Nút back Android** (cả `screen` và `modal`): Miniapp đóng popup trên cùng (hoặc chặn back ở màn đang thanh toán); nếu không có, lùi về trang trước trong Miniapp; hết trang thì đóng Miniapp và `onError` trả `USER_CANCEL`.
- iOS: plugin mở Miniapp từ view controller trên cùng của ứng dụng (mở được khi đang có màn hình present khác).

### PayMEOpeningParam

| Named constructor | Action | `identityToken` | `data` | Mô tả |
| --- | --- | --- | --- | --- |
| `PayMEOpeningParam.payme()` | `PAYME` | Không dùng | Không | Mở ví PayME; người dùng tự đăng nhập bằng số điện thoại và PIN |
| `PayMEOpeningParam.open(identityToken:)` | `OPEN` | Bắt buộc | Không | Mở trang chủ ví; kích hoạt ví nếu người dùng chưa có |
| `PayMEOpeningParam.pay(identityToken:, data:)` | `PAY` | Bắt buộc | `PayMEPaymentParam` (bắt buộc) | Thanh toán đơn hàng |
| `PayMEOpeningParam.payment(identityToken:, data:)` | `PAYMENT` | Tuỳ chọn | `PayMEPaymentDirectParam` (bắt buộc) | Thanh toán giao dịch đối tác đã tạo sẵn qua API PayME |
| `PayMEOpeningParam.transferQR(identityToken:, data:)` | `TRANSFER_QR` | Bắt buộc | `PayMETransferQRParam` (bắt buộc) | Chuyển tiền từ ví PayME tới tài khoản ngân hàng đích |
| `PayMEOpeningParam.service(identityToken:, data:)` | `SERVICE` | Bắt buộc | `PayMEServiceParam` (bắt buộc) | Thanh toán dịch vụ (điện, nước, thẻ, ...) |
| `PayMEOpeningParam.deposit(identityToken:, data:)` | `DEPOSIT` | Bắt buộc | `PayMEPayParam` (bắt buộc) | Nạp tiền vào ví |
| `PayMEOpeningParam.withdraw(identityToken:, data:)` | `WITHDRAW` | Bắt buộc | `PayMEPayParam` (bắt buộc) | Rút tiền từ ví |
| `PayMEOpeningParam.transfer(identityToken:, data:)` | `TRANSFER` | Bắt buộc | `PayMEPayParam` (bắt buộc) | Chuyển tiền |
| `PayMEOpeningParam.kyc(identityToken:)` | `KYC` | Bắt buộc | Không | Mở luồng định danh (KYC) |

Với `PAYMENT`, đối tác lấy danh sách phương thức và tạo giao dịch qua API PayME (liên hệ PayME), rồi truyền mã giao dịch vào `PayMEPaymentDirectParam.transaction`.

Ví dụ:

```dart
void buildParams(String identityToken) {
  // Ví PayME, không cần identityToken
  final wallet = PayMEOpeningParam.payme();

  // Thanh toán giao dịch có sẵn; bỏ identityToken để thanh toán như khách vãng lai
  final payment = PayMEOpeningParam.payment(data: PayMEPaymentDirectParam(transaction: '<TRANSACTION>'));

  // Chuyển tiền tới tài khoản ngân hàng
  final transferQR = PayMEOpeningParam.transferQR(
    identityToken: identityToken,
    data: PayMETransferQRParam(
      amount: 20000,
      bankNumber: '<BANK_ACCOUNT_NUMBER>',
      swiftCode: '<SWIFT_CODE>',
      cardHolder: '<ACCOUNT_HOLDER_NAME>',
    ),
  );

  // Thanh toán dịch vụ điện
  final service = PayMEOpeningParam.service(identityToken: identityToken, data: PayMEServiceParam(service: 'POWE'));

  // Nạp tiền
  final deposit = PayMEOpeningParam.deposit(identityToken: identityToken, data: PayMEPayParam(amount: 10000));
}
```

### Các lớp tham số

Số tiền tính bằng VND và phải là số nguyên, kể cả khi kiểu Dart là `double` (`20000` hoặc `20000.0`), tối đa 2.147.483.647. Số lẻ (ví dụ `20000.5`) hoặc vượt giới hạn làm `open` ném `INVALID_PARAM` trên cả iOS và Android, kể cả với `amount` tuỳ chọn của `PayMEPayParam`.

**`PayMEPaymentParam`** (dùng cho `pay`)

| Thuộc tính | Bắt buộc | Kiểu | Mô tả |
| --- | --- | --- | --- |
| `amount` | Có | `int` | Số tiền giao dịch |
| `transactionId` | Có | `String` | Mã giao dịch của đối tác |
| `note` | Không | `String?` | Ghi chú |
| `ipnUrl` | Không | `String?` | URL server PayME gọi (IPN) khi giao dịch thành công hoặc thất bại |
| `isShowResult` | Không | `bool?` | Hiện màn hình kết quả của PayME (mặc định `true`) |

**`PayMEPaymentDirectParam`** (dùng cho `payment`)

| Thuộc tính | Bắt buộc | Kiểu | Mô tả |
| --- | --- | --- | --- |
| `transaction` | Có | `String` | Mã giao dịch đã tạo qua API PayME |
| `isShowResult` | Không | `bool?` | Hiện màn hình kết quả của PayME (mặc định `true`) |

**`PayMETransferQRParam`** (dùng cho `transferQR`)

| Thuộc tính | Bắt buộc | Kiểu | Mô tả |
| --- | --- | --- | --- |
| `amount` | Có | `double` | Số tiền (truyền `0` nếu để người dùng tự nhập) |
| `bankNumber` | Có | `String` | Số tài khoản hoặc số thẻ đích |
| `swiftCode` | Có | `String` | Mã SWIFT của ngân hàng đích |
| `cardHolder` | Có | `String` | Tên chủ tài khoản |
| `note` | Không | `String?` | Ghi chú |
| `partnerTransaction` | Không | `String?` | Mã giao dịch của đối tác |
| `extraData` | Không | `Map<String, dynamic>?` | Dữ liệu tuỳ ý của đối tác, trả lại khi giao dịch thành công |
| `isShowResult` | Không | `bool?` | Hiện màn hình kết quả của PayME (mặc định `true`) |

**`PayMEServiceParam`** (dùng cho `service`)

| Thuộc tính | Bắt buộc | Kiểu | Mô tả |
| --- | --- | --- | --- |
| `service` | Có | `String` | Mã dịch vụ (bảng dưới) |
| `isBackToApp` | Không | `bool?` | Quay về ứng dụng đối tác sau khi hoàn tất |
| `isShowResult` | Không | `bool?` | Hiện màn hình kết quả của PayME (mặc định `true`) |

| Mã dịch vụ | Dịch vụ |
| --- | --- |
| `POWE` | Điện |
| `WATE` | Nước |
| `ADSL` | Internet |
| `TIVI` | Truyền hình |
| `PPMB` | Điện thoại trả sau |
| `MOBILE_CARD` | Thẻ điện thoại |
| `MOBILE_TOPUP` | Nạp tiền điện thoại |
| `GAME_CARD` | Thẻ game |

**`PayMEPayParam`** (dùng cho `deposit`, `withdraw`, `transfer`)

| Thuộc tính | Bắt buộc | Kiểu | Mô tả |
| --- | --- | --- | --- |
| `amount` | Không | `double?` | Số tiền gợi ý |
| `description` | Không | `String?` | Mô tả giao dịch |
| `isBackToApp` | Không | `bool?` | Quay về ứng dụng đối tác sau khi hoàn tất |
| `isShowResult` | Không | `bool?` | Hiện màn hình kết quả của PayME (mặc định `true`) |

### getBalance

```dart
static Future<dynamic> getBalance(String identityToken);
```

Lấy số dư ví của người dùng. Kết quả về `onResponse` với `action` `GET_BALANCE`:

| Trường | Kiểu | Mô tả |
| --- | --- | --- |
| `linked` | `bool` | Người dùng đã liên kết ví với ứng dụng hay chưa |
| `balance` | `int` | Số dư (VND); chỉ có khi `linked` là `true` |
| `message` | `String` | Thông điệp kèm theo |

### getAccountInfo

```dart
static Future<dynamic> getAccountInfo(String identityToken);
```

Lấy thông tin tài khoản PayME. Kết quả về `onResponse` với `action` `GET_ACCOUNT_INFO`, kèm các trường thông tin tài khoản do PayME trả về (ví dụ `accountId`).

Cả hai hàm đều trả lỗi qua `onError`: `"401"` khi `identityToken` sai hoặc hết hạn, `"IDENTITY_TOKEN_REQUIRED"` khi chuỗi rỗng.

### setLanguage

```dart
static Future<dynamic> setLanguage(String language);
```

Đổi ngôn ngữ Miniapp sau khi đã `init`. `language` là `PayMELocale.VI` (`'vi'`) hoặc `PayMELocale.EN` (`'en'`).

### close

```dart
static Future<dynamic> close();
```

Đóng Miniapp đang mở, trên cả iOS và Android (iOS hỗ trợ từ 3.0.1). Không có Miniapp nào mở thì không làm gì. Gọi trước `init`: ném `PAYME_NOT_INITIALIZED`.

| | iOS | Android |
| --- | --- | --- |
| Cách đóng | Đóng chính Miniapp, kể cả khi đang bị che (máy ảnh, màn hình khác); không dismiss nhầm màn hình của ứng dụng | Đóng Miniapp (`screen` và `modal`) |
| `onError` | `USER_CANCEL` một lần khi Miniapp đã đóng | Không phát |

### onResponse và onError

```dart
static Stream<dynamic> get onResponse;
static Stream<dynamic> get onError;
```

Đây là broadcast stream; đăng ký một lần trước khi gọi `open`, `getBalance` hay `getAccountInfo`, và huỷ `StreamSubscription` khi không dùng nữa.

| Stream | Kiểu sự kiện | Nội dung |
| --- | --- | --- |
| `onResponse` | `Map` | `action` (`String`) cộng các trường kết quả của thao tác đó, ví dụ `{action: GET_BALANCE, linked: true, balance: 150000, message: ...}` |
| `onError` | `Map` | `{action: String, code: String, description: String}` trên cả hai nền tảng |

Giá trị `action`: `PAYME`, `OPEN`, `PAY`, `PAYMENT`, `TRANSFER_QR`, `SERVICE`, `DEPOSIT`, `WITHDRAW`, `TRANSFER`, `KYC`, `GET_BALANCE`, `GET_ACCOUNT_INFO`.

Mảng JSON trong `onResponse` được trả dưới dạng `List`, giá trị `null` giữ là `null` (trước 3.0.1, Android bỏ cả response có mảng). Lỗi kiểm tra tham số không đi qua `onError` mà qua `PlatformException` của Future.

```dart
final errorSub = PaymeMiniappFlutter.onError.listen((event) {
  final code = event['code'] as String?;
  switch (code) {
    case '401':
    case 'IDENTITY_TOKEN_REQUIRED':
      // Lấy identityToken mới từ server rồi mở lại Miniapp.
      break;
    case 'USER_CANCEL':
      // Miniapp đã đóng; thường không cần báo lỗi cho người dùng.
      break;
    default:
      debugPrint('PayME ${event['action']} lỗi $code: ${event['description']}');
  }
});

// Khi không dùng nữa:
// errorSub.cancel();
```

### Hằng số

| Lớp | Hằng số | Giá trị |
| --- | --- | --- |
| `PayMEEnv` | `SANDBOX`, `PRODUCTION` | `'SANDBOX'`, `'PRODUCTION'` |
| `PayMEMode` | `MINIAPP_SANDBOX`, `MINIAPP_PRODUCT` | `'miniapp_sandbox'`, `'miniapp_product'` |
| `PayMELocale` | `VI`, `EN` | `'vi'`, `'en'` |
| `PayMEOpeningType` | `screen`, `modal` | `'screen'`, `'modal'` |

## Mã lỗi

### Lỗi nhận qua `onError`

Mã lỗi nằm ở `event['code']`.

| Mã | Khi nào | Cách xử lý |
| --- | --- | --- |
| `401` | `identityToken` sai hoặc hết hạn (mở Miniapp, `getBalance`, `getAccountInfo`). Miniapp tự đóng. | Gọi server lấy `identityToken` mới rồi thực hiện lại. |
| `IDENTITY_TOKEN_REQUIRED` | `identityToken` rỗng ở thao tác bắt buộc | Kiểm tra luồng lấy chuỗi; không mở Miniapp khi chưa có chuỗi. |
| `USER_CANCEL` | Miniapp đóng (nút đóng, nút back Android khi hết trang, chạm ngoài `modal`); iOS: cả khi gọi `close()` | Coi là sự kiện đóng Miniapp, không phải lỗi hệ thống. Báo một lần mỗi phiên. |
| `SYSTEM_CANCEL` | (iOS) Miniapp không khởi động được (ví dụ web server cục bộ không chạy được). Miniapp tự đóng | Thử mở lại; nếu lặp lại, báo PayME. |
| `MINIAPP_ALREADY_OPEN` | Gọi `open` khi đang có Miniapp mở. Miniapp mới không mở, Miniapp đang mở giữ nguyên | Đợi Miniapp hiện tại đóng, hoặc gọi `close()` rồi mở lại. |
| `MINIAPP_CANNOT_PRESENT` | SDK không hiển thị được Miniapp dù plugin đã chuyển lệnh (ví dụ Android lỗi khi hiện màn hình) | Gọi `open` khi ứng dụng đang ở foreground. |
| `CONNECTION_LOST`, `TIMED_OUT`, `NO_RESPONSE` (iOS) | Lỗi mạng hoặc server không phản hồi | Báo người dùng kiểm tra kết nối và thử lại. |
| `ENCODE_FAILED` (iOS), `DECODE_FAILED` | Lỗi mã hoá hoặc giải mã dữ liệu, thường do sai key hoặc sai môi trường | Kiểm tra `appId`, cặp key và cặp `env`/`mode`. |
| `SERVER` (iOS), `SERVER_ERROR` (Android) | Server PayME trả lỗi | Thử lại sau; hiển thị `description`. |
| `OTHER` | Lỗi khác (ví dụ không tìm thấy thông tin tài khoản) | Hiển thị `description`. |
| Mã khác | Lỗi nghiệp vụ do Miniapp trả về | Hiển thị `description`; liên hệ PayME nếu cần tra cứu. |

### Lỗi phía plugin (`PlatformException`)

Từ 3.0.1, các mã dưới đây được ném dưới dạng `PlatformException` từ Future của hàm được gọi (`e.code`, `e.message`). Khi đó lệnh không được thực hiện (ví dụ Miniapp không mở).

```dart
try {
  await PaymeMiniappFlutter.getBalance(identityToken);
} on PlatformException catch (e) {
  if (e.code == 'PAYME_NOT_INITIALIZED') {
    // Gọi init trước rồi thử lại.
  }
}
```

| Mã | Nền tảng | Khi nào | Cách xử lý |
| --- | --- | --- | --- |
| `MISSING_INFO` | iOS, Android | `init` có `appId`/`publicKey`/`privateKey` rỗng; `open` thiếu `action`; `getBalance`/`getAccountInfo`/`setLanguage` thiếu đối số | Truyền đủ tham số. |
| `INVALID_PARAM` | iOS, Android | Action không hỗ trợ, hoặc `data` thiếu/sai kiểu cho `pay`, `payment`, `transferQR`, `service`; `amount` không nguyên (mọi action) | Truyền đúng lớp tham số và trường bắt buộc. |
| `PAYME_NOT_INITIALIZED` | iOS, Android | Gọi `open`/`close`/`getBalance`/`getAccountInfo`/`setLanguage` trước `init` | Gọi `init` khi ứng dụng khởi động, đợi Future hoàn tất. |
| `MINIAPP_CANNOT_PRESENT` | iOS, Android | `open` không hiển thị được Miniapp. iOS: không có view controller để present. Android (`modal`): chưa có Activity, `MainActivity` không kế thừa `FlutterFragmentActivity` hoặc Activity đã lưu trạng thái | Gọi `open` khi ứng dụng đang hiển thị; Android: dùng `FlutterFragmentActivity` hoặc `PayMEOpeningType.screen`. |
| `JSON_PARSE_ERROR` | Android | Plugin không đọc được tham số của `init` hoặc `open` (hiếm gặp) | Xem log; báo PayME kèm thông tin tái hiện. |
| `INIT_ERROR` | Android | SDK native báo lỗi khi khởi tạo | Kiểm tra bộ key và `env`; báo PayME nếu lặp lại. |
| `OPEN_ERROR` | Android | SDK native ném lỗi khi mở Miniapp (phòng hờ; SDK thường báo lỗi mở qua `onError`) | Xem log; báo PayME kèm thông tin tái hiện. |

## Nâng cấp từ 2.x

| 2.x (SDK native iOS 0.7.6 / Android 0.7.2) | 3.0.1 (SDK native iOS 0.8.3 / Android 0.8.4) |
| --- | --- |
| `PayMEOpeningParam.open(phone: phone)` | `PayMEOpeningParam.open(identityToken: identityToken)` |
| `PayMEOpeningParam.pay(phone: phone, data: ...)` | `PayMEOpeningParam.pay(identityToken: identityToken, data: ...)` |
| `PayMEOpeningParam.payment(data: ...)` | `PayMEOpeningParam.payment(identityToken: identityToken /* tuỳ chọn */, data: ...)` |
| `PayMEOpeningParam.transferQR(phone: phone, data: ...)` | `PayMEOpeningParam.transferQR(identityToken: identityToken, data: ...)` |
| `PayMEOpeningParam.service(phone: phone, data: ...)` | `PayMEOpeningParam.service(identityToken: identityToken, data: ...)` |
| `PayMEOpeningParam.deposit` / `withdraw` / `transfer(phone: phone, data: ...)` | `PayMEOpeningParam.deposit` / `withdraw` / `transfer(identityToken: identityToken, data: ...)` |
| `PayMEOpeningParam.kyc(phone: phone)` | `PayMEOpeningParam.kyc(identityToken: identityToken)` |
| (không có) | `PayMEOpeningParam.payme()` |
| `PaymeMiniappFlutter.getBalance(phone)` | `PaymeMiniappFlutter.getBalance(identityToken)` |
| `PaymeMiniappFlutter.getAccountInfo(phone)` | `PaymeMiniappFlutter.getAccountInfo(identityToken)` |

Các bước nâng cấp:

1. Đổi dependency thành `payme_miniapp_flutter: ^3.0.1`, chạy `flutter pub get` rồi `cd ios && pod install --repo-update`.
2. Dùng Xcode 26.2 trở lên để build iOS.
3. Xây dựng endpoint trên server để gọi API `identityToken/init` của PayME và trả chuỗi cho ứng dụng (xem [identityToken](#identitytoken)).
4. Thay mọi tham số `phone` bằng `identityToken` theo bảng trên.
5. Xử lý thêm mã `401` và `IDENTITY_TOKEN_REQUIRED` trong `onError`.
6. Bọc các lời gọi `init`, `open`, `close`, `getBalance`, `getAccountInfo`, `setLanguage` bằng `try/catch` `PlatformException` (xem [Lỗi phía plugin](#lỗi-phía-plugin-platformexception)).

Chữ ký của `init`, `setLanguage`, `close`, `onResponse`, `onError` và các lớp tham số `data` không thay đổi. Khác biệt so với 2.x ngoài `identityToken`: từ 3.0.1 các hàm ném `PlatformException` khi lỗi thay vì trả `null`.

Đã dùng 3.0.0: nâng lên `^3.0.1` không cần đổi mã gọi hàm, nhưng cần thêm `try/catch` như bước 6.

## Lịch sử thay đổi và hỗ trợ

| Phiên bản | Thay đổi chính |
| --- | --- |
| 3.0.1 | SDK native iOS 0.8.3, Android 0.8.4 (SDK 1.0.24): `USER_CANCEL` báo một lần mỗi phiên; mỗi lúc chỉ một Miniapp (`MINIAPP_ALREADY_OPEN`); Android: nút back đóng popup → lùi trang → đóng Miniapp, không còn ép ứng dụng sang giao diện sáng hay đổi ngôn ngữ của ứng dụng. Android mở được `modal` với `FlutterFragmentActivity`; không hiển thị được thì `open` ném `MINIAPP_CANNOT_PRESENT`. `mode` mặc định theo `env`. `amount` tối đa 2.147.483.647. iOS: mở từ view controller trên cùng, `close()` đóng chính Miniapp. Các hàm ném `PlatformException` khi lỗi và luôn hoàn tất; iOS hỗ trợ `close`. Android: `INVALID_PARAM` thay vì mở ví, `PAYME_NOT_INITIALIZED` khi chưa `init`, `onError` luôn là `Map`, mảng trong `onResponse` là `List`. `amount` không nguyên: `INVALID_PARAM` |
| 3.0.0 | `identityToken` thay số điện thoại; SDK native iOS 0.8.1, Android 0.8.2 (SDK 1.0.24) |

- Lịch sử thay đổi đầy đủ: [pub.dev/packages/payme_miniapp_flutter/changelog](https://pub.dev/packages/payme_miniapp_flutter/changelog)
- Bộ key (`appId`, `publicKey`, `privateKey`), tài liệu API `identityToken/init`, API tạo giao dịch cho `PAYMENT` và tài khoản sandbox: liên hệ PayME.
