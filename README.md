**Tiếng Việt** | [English](README.en.md)

# payme_miniapp_flutter

Plugin Flutter để mở PayME Miniapp (ví, thanh toán, nạp, rút, chuyển tiền, dịch vụ, eKYC) trong ứng dụng của bạn trên iOS và Android.

Phiên bản hiện tại: **3.0.3**. Plugin đã kèm SDK native (iOS `PayME-Miniapp` 0.8.4, Android `native-android` 0.8.5), ứng dụng không cần khai báo thêm.

## Mục lục

1. [Yêu cầu](#yêu-cầu)
2. [Cài đặt](#cài-đặt)
3. [Cấu hình ứng dụng](#cấu-hình-ứng-dụng)
4. [Bắt đầu nhanh](#bắt-đầu-nhanh)
5. [identityToken](#identitytoken)
6. [API](#api)
7. [Mã lỗi](#mã-lỗi)
8. [Nâng cấp từ 2.x](#nâng-cấp-từ-2x)
9. [Hỗ trợ](#hỗ-trợ)

## Yêu cầu

- Dart `>=3.2.0 <4.0.0`, Flutter `>=3.3.0` (phải build được với Android Gradle Plugin 9).
- iOS 13.0 trở lên, build bằng Xcode 26.2 trở lên, CocoaPods.
- Android `minSdkVersion` 26, `targetSdkVersion` 33 trở lên, `compileSdkVersion` 37.
- Build Android: JDK 17, Gradle 9.3.1, Android Gradle Plugin 9.1.1, Kotlin Gradle Plugin 2.2.20, NDK 28.2.13676358.
- Bộ key `appId`, `publicKey`, `privateKey` do PayME cấp.
- Server của bạn gọi được API cấp `identityToken` của PayME.

## Cài đặt

```sh
flutter pub add payme_miniapp_flutter
```

Hoặc thêm vào `pubspec.yaml`:

```yaml
dependencies:
  payme_miniapp_flutter: ^3.0.3
```

### iOS

Cập nhật `ios/Podfile`:

```ruby
# Mức iOS tối thiểu của ứng dụng (giống `platform :ios` ở đầu Podfile). Build bằng Xcode 27 thì tối thiểu '15.0'.
app_min_ios = '13.0'
platform :ios, app_min_ios

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
        end
        # Nâng pod có iOS tối thiểu thấp hơn ứng dụng (Xcode 27 báo lỗi nếu dưới 15.0)
        if config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'].to_f < app_min_ios.to_f
          config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = app_min_ios
        end
      end
    end
  end
end
```

```sh
cd ios && pod install --repo-update
```

### Android

1. **JitPack.** Thêm `https://jitpack.io` vào danh sách repository. Nếu dự án khai báo repository trong `android/settings.gradle`:

   ```groovy
   dependencyResolutionManagement {
       repositories {
           google()
           mavenCentral()
           maven { url = uri("https://jitpack.io") }
       }
   }
   ```

   Nếu khai báo trong `allprojects { repositories { ... } }` của `android/build.gradle`, thêm `maven { url "https://jitpack.io" }` vào đó.
2. **Gradle và plugin.** Dùng Android Gradle Plugin `9.1.1` và Kotlin Gradle Plugin `2.2.20` (trong khối `plugins` của `android/settings.gradle`, hoặc `buildscript` của `android/build.gradle` với template cũ). Gradle Wrapper (`android/gradle/wrapper/gradle-wrapper.properties`):

   ```properties
   distributionUrl=https\://services.gradle.org/distributions/gradle-9.3.1-all.zip
   ```

3. **Module app** (`android/app/build.gradle`):

   ```groovy
   android {
       compileSdkVersion 37
       ndkVersion = "28.2.13676358"

       defaultConfig {
           minSdkVersion 26
           targetSdkVersion flutter.targetSdkVersion // từ 33 trở lên
       }
   }
   ```

4. **`android/gradle.properties`:**

   ```properties
   android.useAndroidX=true
   android.enableJetifier=true
   # Build bằng Kotlin Gradle Plugin trong khi Flutter chưa hỗ trợ Built-in Kotlin của AGP 9
   android.builtInKotlin=false
   android.newDsl=false
   # Tránh lỗi trùng namespace giữa các thư viện litert-support khi build với AGP 9
   android.uniquePackageNames=false
   ```

5. **Activity.** Để mở dạng `modal`, `MainActivity` phải kế thừa `FlutterFragmentActivity` (dạng `screen` chạy được với `FlutterActivity`):

   ```kotlin
   import io.flutter.embedding.android.FlutterFragmentActivity

   class MainActivity : FlutterFragmentActivity()
   ```

6. **ProGuard.** Nếu bản release bật `minifyEnabled`, thêm vào `android/app/proguard-rules.pro`:

   ```proguard
   -keep class vn.kalapa.ekyc.**{*;}
   -keep class com.fis.ekyc.**{*;}
   -keep class com.fis.nfc.**{*;}
   -keep class retrofit2.**{*;}
   -keepclassmembers,allowobfuscation class * {
       @com.google.gson.annotations.SerializedName <fields>;
   }
   ```

## Cấu hình ứng dụng

### iOS

Thêm vào `ios/Runner/Info.plist` (sửa câu mô tả theo ứng dụng của bạn):

```xml
<!-- Mở app ngân hàng khi nạp tiền -->
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>vcbpartner</string>
  <string>bidv.smartbanking.partner</string>
</array>
<key>NSCameraUsageDescription</key>
<string>Ứng dụng cần dùng máy ảnh để định danh và quét mã thanh toán</string>
<key>NSContactsUsageDescription</key>
<string>Ứng dụng cần truy cập danh bạ để chọn nhanh số điện thoại</string>
<key>NSFaceIDUsageDescription</key>
<string>Dùng Face ID để xác thực nhanh và an toàn</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>Ứng dụng cần quyền lưu ảnh mã QR vào thư viện ảnh</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Ứng dụng cần truy cập thư viện ảnh để chọn mã QR</string>
<!-- Đọc chip CCCD khi eKYC -->
<key>NFCReaderUsageDescription</key>
<string>Ứng dụng cần dùng NFC để đọc căn cước công dân khi định danh (eKYC)</string>
<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
  <string>A0000002471001</string>
  <string>A0000002472001</string>
  <string>00000000000000</string>
</array>
```

Trong Xcode, vào target `Runner` → **Signing & Capabilities** → **+ Capability** → thêm **Near Field Communication Tag Reading**.

![Near Field Communication Tag Reading](documents/capabilities2.png)

### Android

Không cần khai báo quyền. Các quyền cần thiết đã có sẵn trong SDK, Gradle tự gộp vào ứng dụng.

Miniapp chạy trên `http://localhost`, nên SDK khai báo `networkSecurityConfig` chỉ cho phép cleartext tới `localhost`. Khi đó `android:usesCleartextTraffic` của ứng dụng không còn tác dụng. Nếu ứng dụng có `networkSecurityConfig` riêng, thêm `tools:replace="android:networkSecurityConfig"` vào `<application>` và cho phép cleartext tới `localhost` trong file của bạn:

```xml
<domain-config cleartextTrafficPermitted="true">
    <domain>localhost</domain>
</domain-config>
```

## Bắt đầu nhanh

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';
import 'package:payme_miniapp_flutter/payme_miniapp_flutter.dart';
import 'package:payme_miniapp_flutter/payme_miniapp_type.dart';

// Gọi một lần khi ứng dụng khởi động
Future<void> setUpPayME() async {
  PaymeMiniappFlutter.onResponse.listen((e) => debugPrint('PayME response ${e['action']}'));
  PaymeMiniappFlutter.onError.listen((e) => debugPrint('PayME error ${e['action']} ${e['code']}'));
  try {
    await PaymeMiniappFlutter.init('<APP_ID>', '<PUBLIC_KEY>', '<PRIVATE_KEY>', env: PayMEEnv.SANDBOX);
  } on PlatformException catch (e) {
    debugPrint('PayME init lỗi ${e.code}: ${e.message}');
  }
}

// identityToken do server của bạn lấy từ PayME
Future<void> openWallet(String identityToken) async {
  try {
    await PaymeMiniappFlutter.open(
        PayMEOpeningType.modal, PayMEOpeningParam.open(identityToken: identityToken));
  } on PlatformException catch (e) {
    debugPrint('PayME open lỗi ${e.code}: ${e.message}');
  }
}
```

> Không đưa key thật vào mã nguồn công khai. `publicKey` và `privateKey` viết trên một dòng, bỏ ký tự xuống dòng (ví dụ `-----BEGIN PUBLIC KEY-----MFww...-----END PUBLIC KEY-----`).

## identityToken

`identityToken` là chuỗi PayME cấp cho từng người dùng của bạn. Ứng dụng truyền chuỗi này mỗi lần mở Miniapp.

1. Ứng dụng gọi server của bạn.
2. Server của bạn gọi API `token-init` (`POST /be/ewallet/sdk/token-init`) của PayME với số điện thoại của người dùng (liên hệ PayME để nhận tài liệu API). Không gọi API này từ ứng dụng.
3. Ứng dụng truyền `identityToken` nhận được vào `PayMEOpeningParam`, `getBalance` hoặc `getAccountInfo`.

| Thao tác | `identityToken` |
| --- | --- |
| `PayMEOpeningParam.payment(...)` | Tuỳ chọn. Có thì người dùng trả được bằng ví PayME; không có thì thanh toán như khách vãng lai (thẻ, QR) |
| Các thao tác còn lại, `getBalance`, `getAccountInfo` | Bắt buộc |

- Chuỗi có hiệu lực 15 phút, dùng lại được trong thời gian đó.
- Sai hoặc hết hạn: `onError` trả `401` (Miniapp đóng nếu đang mở). Lấy chuỗi mới rồi gọi lại.
- Chỉ giữ chuỗi trong bộ nhớ: không lưu, không ghi log.

## API

Mọi hàm là `static` trên `PaymeMiniappFlutter`. Lớp tham số và hằng số nằm trong `package:payme_miniapp_flutter/payme_miniapp_type.dart`.

Các hàm trả `Future` không mang dữ liệu. Kết quả và lỗi nghiệp vụ trả về qua stream `onResponse` và `onError`. Khi gọi sai (thiếu tham số, chưa `init`…), `Future` ném `PlatformException` và lệnh không được thực hiện. Hãy bọc lời gọi bằng `try/catch`.

### init

```dart
PaymeMiniappFlutter.init(appId, publicKey, privateKey, {locale, env, mode})
```

| Tham số | Mô tả |
| --- | --- |
| `appId` | Mã ứng dụng do PayME cấp |
| `publicKey` | Public key của PayME |
| `privateKey` | Private key của ứng dụng |
| `locale` | `PayMELocale.VI` (mặc định) hoặc `PayMELocale.EN` |
| `env` | `PayMEEnv.PRODUCTION` (mặc định) hoặc `PayMEEnv.SANDBOX` |
| `mode` | Thường không cần truyền. Mặc định `PayMEMode.MINIAPP_SANDBOX` khi `env` là `SANDBOX`, còn lại `PayMEMode.MINIAPP_PRODUCT` |

Gọi `init` một lần, trước mọi hàm khác. Gọi hàm khác khi chưa `init` sẽ nhận lỗi `PAYME_NOT_INITIALIZED`.

### open

```dart
PaymeMiniappFlutter.open(type, openingParam)
```

`type`: `PayMEOpeningType.screen` (toàn màn hình) hoặc `PayMEOpeningType.modal` (trượt từ dưới lên; Android cần [`FlutterFragmentActivity`](#android)).

`openingParam` tạo bằng một trong các named constructor của `PayMEOpeningParam`:

| Constructor | Chức năng | `data` |
| --- | --- | --- |
| `open(identityToken:)` | Trang chủ ví (chưa kích hoạt thì mở luồng kích hoạt) | Không có |
| `kyc(identityToken:)` | Định danh (eKYC) | Không có |
| `pay(identityToken:, data:)` | Thanh toán đơn hàng của bạn | [`PayMEPaymentParam`](#paymepaymentparam-pay) |
| `payment(identityToken:, data:)` | Thanh toán giao dịch đã tạo qua API PayME | [`PayMEPaymentDirectParam`](#paymepaymentdirectparam-payment) |
| `transferQR(identityToken:, data:)` | Chuyển tiền tới tài khoản ngân hàng | [`PayMETransferQRParam`](#paymetransferqrparam-transferqr) |
| `service(identityToken:, data:)` | Thanh toán dịch vụ (điện, nước, thẻ…) | [`PayMEServiceParam`](#paymeserviceparam-service) |
| `deposit`, `withdraw`, `transfer` `(identityToken:, data:)` | Nạp, rút, chuyển tiền | [`PayMEPayParam`](#paymepayparam-deposit-withdraw-transfer) |

```dart
await PaymeMiniappFlutter.open(
  PayMEOpeningType.screen,
  PayMEOpeningParam.pay(
    identityToken: identityToken,
    data: PayMEPaymentParam(amount: 10000, transactionId: 'ORDER_123', note: 'Đơn hàng 123'),
  ),
);
// Khách vãng lai: bỏ identityToken
await PaymeMiniappFlutter.open(PayMEOpeningType.modal,
    PayMEOpeningParam.payment(data: PayMEPaymentDirectParam(transaction: '<TRANSACTION>')));
await PaymeMiniappFlutter.open(PayMEOpeningType.screen,
    PayMEOpeningParam.service(identityToken: identityToken, data: PayMEServiceParam(service: 'POWE')));
await PaymeMiniappFlutter.open(PayMEOpeningType.screen,
    PayMEOpeningParam.deposit(identityToken: identityToken, data: PayMEPayParam(amount: 50000)));
```

Tham số sai (thiếu `data`, thiếu trường bắt buộc, `amount` không phải số nguyên) thì Miniapp không mở và `open` ném `PlatformException` `INVALID_PARAM`.

`amount` là số nguyên VND (kể cả khi kiểu là `double`), tối đa 2.147.483.647. Mọi lớp `data` dưới đây đều có thêm trường tuỳ chọn `isShowResult` (hiện màn hình kết quả của PayME, mặc định `true`).

#### PayMEPaymentParam (pay)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `transactionId` | Có | Mã giao dịch của bạn |
| `amount` | Có | Số tiền |
| `ipnUrl` | Không | URL nhận thông báo trạng thái giao dịch |
| `note` | Không | Ghi chú |

#### PayMEPaymentDirectParam (payment)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `transaction` | Có | Mã giao dịch PayME trả về khi bạn tạo giao dịch |

#### PayMETransferQRParam (transferQR)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `amount` | Có | Số tiền (`0` để người dùng tự nhập) |
| `bankNumber` | Có | Số tài khoản hoặc số thẻ nhận |
| `swiftCode` | Có | Mã SWIFT của ngân hàng nhận |
| `cardHolder` | Có | Tên chủ tài khoản |
| `note` | Không | Ghi chú |
| `partnerTransaction` | Không | Mã giao dịch của bạn |
| `extraData` | Không | Dữ liệu riêng của bạn (`Map`), trả lại khi giao dịch thành công |

#### PayMEServiceParam (service)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `service` | Có | `POWE` (điện), `WATE` (nước), `ADSL` (internet), `TIVI` (truyền hình), `PPMB` (trả sau), `MOBILE_CARD` (thẻ điện thoại), `MOBILE_TOPUP` (nạp điện thoại), `GAME_CARD` (thẻ game) |
| `isBackToApp` | Không | Quay về ứng dụng của bạn sau khi xong |

#### PayMEPayParam (deposit, withdraw, transfer)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `amount` | Không | Số tiền điền sẵn |
| `description` | Không | Mô tả giao dịch |
| `isBackToApp` | Không | Quay về ứng dụng của bạn sau khi xong |

### getBalance, getAccountInfo

```dart
PaymeMiniappFlutter.getBalance(identityToken)
PaymeMiniappFlutter.getAccountInfo(identityToken)
```

Kết quả trả qua `onResponse`:

- `GET_BALANCE`: `linked` (đã liên kết ví chưa), `balance` (có khi `linked` = `true`).
- `GET_ACCOUNT_INFO`: thông tin tài khoản ví.

### setLanguage, close

- `setLanguage(PayMELocale.VI | PayMELocale.EN)`: đổi ngôn ngữ Miniapp.
- `close()`: đóng Miniapp đang mở.

### Sự kiện

```dart
final errorSub = PaymeMiniappFlutter.onError.listen((e) { /* e['action'], e['code'], e['description'] */ });
final responseSub = PaymeMiniappFlutter.onResponse.listen((e) { /* e['action'], ... */ });
// Khi không dùng nữa:
errorSub.cancel();
responseSub.cancel();
```

| Stream | Nội dung (`Map`) | Khi nào |
| --- | --- | --- |
| `onResponse` | `{action, ...dữ liệu}` | Giao dịch thành công, hoặc có kết quả `getBalance` / `getAccountInfo` |
| `onError` | `{action, code, description}` | Có lỗi hoặc người dùng đóng Miniapp |

Đăng ký listener một lần, trước khi gọi `open`, `getBalance` hay `getAccountInfo`.

### Mở và đóng Miniapp

- Mỗi lúc chỉ mở một Miniapp. Gọi `open` khi đang có Miniapp mở sẽ nhận `MINIAPP_ALREADY_OPEN` qua `onError`.
- Khi Miniapp đóng, bạn nhận kết quả đúng một lần: `onResponse`, `onError`, hoặc `USER_CANCEL` khi người dùng đóng. Có thể mở lại ngay trong callback này.
- Android: nút back đóng popup, rồi lùi trang trong Miniapp; hết trang thì đóng Miniapp và báo `USER_CANCEL`.
- `close()`: iOS báo `USER_CANCEL`, Android không báo.

## Mã lỗi

Lỗi đến từ hai nơi: `PlatformException` của `Future` (`e.code`, lệnh không được thực hiện) và stream `onError` (`event['code']`).

| `code` | Nhận qua | Ý nghĩa | Cách xử lý |
| --- | --- | --- | --- |
| `401` | `onError` | `identityToken` sai hoặc hết hạn | Lấy `identityToken` mới rồi gọi lại |
| `IDENTITY_TOKEN_REQUIRED` | `onError` | Thiếu `identityToken` | Truyền `identityToken` |
| `MISSING_INFO` | `PlatformException` | Thiếu tham số bắt buộc (ví dụ key rỗng khi `init`) | Truyền đủ tham số |
| `INVALID_PARAM` | `PlatformException` | Tham số `open` sai | Sửa tham số theo bảng ở [open](#open) |
| `PAYME_NOT_INITIALIZED` | `PlatformException` | Chưa gọi `init` | Gọi `init` trước |
| `USER_CANCEL` | `onError` | Người dùng đóng Miniapp | Không phải lỗi |
| `MINIAPP_ALREADY_OPEN` | `onError` | Đang có Miniapp mở | Đợi Miniapp đóng, hoặc gọi `close()` trước |
| `MINIAPP_CANNOT_PRESENT` | Cả hai | Không hiển thị được Miniapp | Gọi `open` khi ứng dụng đang hiển thị; Android mở `modal` cần `FlutterFragmentActivity` |
| Mã khác | Cả hai | Lỗi mạng, máy chủ hoặc nghiệp vụ | Hiển thị `description`, thử lại; liên hệ PayME nếu lặp lại |

## Nâng cấp từ 2.x

Bản 3.x thay số điện thoại bằng `identityToken` và không tương thích ngược.

| 2.x | 3.x |
| --- | --- |
| `PayMEOpeningParam.open(phone: phone)` | `PayMEOpeningParam.open(identityToken: identityToken)` |
| `PayMEOpeningParam.pay(phone: phone, data: ...)` (tương tự `transferQR`, `service`, `deposit`, `withdraw`, `transfer`, `kyc`) | `PayMEOpeningParam.pay(identityToken: identityToken, data: ...)` |
| `PayMEOpeningParam.payment(data: ...)` | `PayMEOpeningParam.payment(identityToken: identityToken /* tuỳ chọn */, data: ...)` |
| `getBalance(phone)` | `getBalance(identityToken)` |
| `getAccountInfo(phone)` | `getAccountInfo(identityToken)` |

1. Server: tích hợp API `token-init` (liên hệ PayME).
2. Đổi dependency thành `payme_miniapp_flutter: ^3.0.3`, chạy `flutter pub get` rồi `cd ios && pod install --repo-update`.
3. Thay mọi `phone` bằng `identityToken`.
4. Xử lý lỗi `401` và `IDENTITY_TOKEN_REQUIRED`; bọc các lời gọi bằng `try/catch` `PlatformException`.
5. Build iOS bằng Xcode 26.2 trở lên.

## Hỗ trợ

PayME Technology Corporation, email [hotro@payme.vn](mailto:hotro@payme.vn). Bộ key, tài liệu API `token-init` và API tạo giao dịch `payment`: liên hệ PayME.

Danh sách phiên bản: [pub.dev `payme_miniapp_flutter`](https://pub.dev/packages/payme_miniapp_flutter/versions).
