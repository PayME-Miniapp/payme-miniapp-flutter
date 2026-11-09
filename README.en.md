[Tiếng Việt](README.md) | **English**

# payme_miniapp_flutter

A Flutter plugin that opens PayME Miniapp (wallet, payment, top-up, withdrawal, transfer, bill payment, eKYC) in your app on iOS and Android.

Current version: **3.0.2**. The plugin bundles the native SDKs (iOS `PayME-Miniapp` 0.8.3, Android `native-android` 0.8.5); your app does not need to declare them.

## Table of contents

1. [Requirements](#requirements)
2. [Installation](#installation)
3. [App configuration](#app-configuration)
4. [Quick start](#quick-start)
5. [identityToken](#identitytoken)
6. [API](#api)
7. [Error codes](#error-codes)
8. [Upgrading from 2.x](#upgrading-from-2x)
9. [Support](#support)

## Requirements

- Dart `>=3.2.0 <4.0.0`, Flutter `>=3.3.0` (must be able to build with Android Gradle Plugin 9).
- iOS 13.0 or later, built with Xcode 26.2 or later, CocoaPods.
- Android `minSdkVersion` 26, `targetSdkVersion` 33 or later, `compileSdkVersion` 37.
- Android build: JDK 17, Gradle 9.3.1, Android Gradle Plugin 9.1.1, Kotlin Gradle Plugin 2.2.20, NDK 28.2.13676358.
- The `appId`, `publicKey`, `privateKey` key set issued by PayME.
- Your server can call PayME's `identityToken` API.

## Installation

```sh
flutter pub add payme_miniapp_flutter
```

Or add it to `pubspec.yaml`:

```yaml
dependencies:
  payme_miniapp_flutter: ^3.0.2
```

### iOS

Update `ios/Podfile`:

```ruby
# Your app's minimum iOS version (same as `platform :ios` at the top of the Podfile). With Xcode 27 the minimum is '15.0'.
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
        # Raise pods whose minimum iOS is below the app's (Xcode 27 errors below 15.0)
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

1. **JitPack.** Add `https://jitpack.io` to your repositories. If your project declares repositories in `android/settings.gradle`:

   ```groovy
   dependencyResolutionManagement {
       repositories {
           google()
           mavenCentral()
           maven { url = uri("https://jitpack.io") }
       }
   }
   ```

   If it declares them in `allprojects { repositories { ... } }` in `android/build.gradle`, add `maven { url "https://jitpack.io" }` there.
2. **Gradle and plugins.** Use Android Gradle Plugin `9.1.1` and Kotlin Gradle Plugin `2.2.20` (in the `plugins` block of `android/settings.gradle`, or in `buildscript` of `android/build.gradle` with the older template). Gradle Wrapper (`android/gradle/wrapper/gradle-wrapper.properties`):

   ```properties
   distributionUrl=https\://services.gradle.org/distributions/gradle-9.3.1-all.zip
   ```

3. **App module** (`android/app/build.gradle`):

   ```groovy
   android {
       compileSdkVersion 37
       ndkVersion = "28.2.13676358"

       defaultConfig {
           minSdkVersion 26
           targetSdkVersion flutter.targetSdkVersion // 33 or later
       }
   }
   ```

4. **`android/gradle.properties`:**

   ```properties
   android.useAndroidX=true
   android.enableJetifier=true
   # Build with the Kotlin Gradle Plugin while Flutter does not yet support AGP 9 Built-in Kotlin
   android.builtInKotlin=false
   android.newDsl=false
   # Avoids a namespace clash between the litert-support libraries when building with AGP 9
   android.uniquePackageNames=false
   ```

5. **Activity.** To open as `modal`, `MainActivity` must extend `FlutterFragmentActivity` (`screen` works with `FlutterActivity`):

   ```kotlin
   import io.flutter.embedding.android.FlutterFragmentActivity

   class MainActivity : FlutterFragmentActivity()
   ```

6. **ProGuard.** If your release build enables `minifyEnabled`, add to `android/app/proguard-rules.pro`:

   ```proguard
   -keep class vn.kalapa.ekyc.**{*;}
   -keep class com.fis.ekyc.**{*;}
   -keep class com.fis.nfc.**{*;}
   -keep class retrofit2.**{*;}
   -keepclassmembers,allowobfuscation class * {
       @com.google.gson.annotations.SerializedName <fields>;
   }
   ```

## App configuration

### iOS

Add to `ios/Runner/Info.plist` (adjust the descriptions to your app):

```xml
<!-- Open bank apps when topping up -->
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>vcbpartner</string>
  <string>bidv.smartbanking.partner</string>
</array>
<key>NSCameraUsageDescription</key>
<string>The app uses the camera for identity verification and to scan payment codes</string>
<key>NSContactsUsageDescription</key>
<string>The app accesses your contacts to pick a phone number quickly</string>
<key>NSFaceIDUsageDescription</key>
<string>Use Face ID for quick, secure authentication</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>The app saves QR code images to your photo library</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>The app accesses your photo library to pick a QR code</string>
<!-- Read the ID card chip during eKYC -->
<key>NFCReaderUsageDescription</key>
<string>The app uses NFC to read your citizen ID card during identity verification (eKYC)</string>
<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
  <string>A0000002471001</string>
  <string>A0000002472001</string>
  <string>00000000000000</string>
</array>
```

In Xcode, open the `Runner` target → **Signing & Capabilities** → **+ Capability** → add **Near Field Communication Tag Reading**.

![Near Field Communication Tag Reading](documents/capabilities2.png)

### Android

No permissions to declare. The required permissions are already in the SDK, and Gradle merges them into your app.

Miniapp runs on `http://localhost`, so the SDK declares a `networkSecurityConfig` that only allows cleartext to `localhost`. Your app's `android:usesCleartextTraffic` then no longer applies. If your app has its own `networkSecurityConfig`, add `tools:replace="android:networkSecurityConfig"` to `<application>` and allow cleartext to `localhost` in your file:

```xml
<domain-config cleartextTrafficPermitted="true">
    <domain>localhost</domain>
</domain-config>
```

## Quick start

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';
import 'package:payme_miniapp_flutter/payme_miniapp_flutter.dart';
import 'package:payme_miniapp_flutter/payme_miniapp_type.dart';

// Call once at app startup
Future<void> setUpPayME() async {
  PaymeMiniappFlutter.onResponse.listen((e) => debugPrint('PayME response ${e['action']}'));
  PaymeMiniappFlutter.onError.listen((e) => debugPrint('PayME error ${e['action']} ${e['code']}'));
  try {
    await PaymeMiniappFlutter.init('<APP_ID>', '<PUBLIC_KEY>', '<PRIVATE_KEY>', env: PayMEEnv.SANDBOX);
  } on PlatformException catch (e) {
    debugPrint('PayME init failed ${e.code}: ${e.message}');
  }
}

// identityToken is fetched from PayME by your server
Future<void> openWallet(String identityToken) async {
  try {
    await PaymeMiniappFlutter.open(
        PayMEOpeningType.modal, PayMEOpeningParam.open(identityToken: identityToken));
  } on PlatformException catch (e) {
    debugPrint('PayME open failed ${e.code}: ${e.message}');
  }
}
```

> Do not put real keys in public source code. Write `publicKey` and `privateKey` on a single line, without line breaks (for example `-----BEGIN PUBLIC KEY-----MFww...-----END PUBLIC KEY-----`).

## identityToken

An `identityToken` is a string PayME issues for each of your users. The app passes it every time it opens Miniapp.

1. The app calls your server.
2. Your server calls PayME's `token-init` API (`POST /be/ewallet/sdk/token-init`) with the user's phone number (contact PayME for the API documentation). Never call this API from the app.
3. The app passes the returned `identityToken` to `PayMEOpeningParam`, `getBalance` or `getAccountInfo`.

| Operation | `identityToken` |
| --- | --- |
| `PayMEOpeningParam.payment(...)` | Optional. With it, the user can pay from the PayME wallet; without it, the user pays as a guest (card, QR) |
| All other operations, `getBalance`, `getAccountInfo` | Required |

- A token is valid for 15 minutes and can be reused within that time.
- Invalid or expired: `onError` emits `401` (Miniapp closes if open). Fetch a new token and retry.
- Keep the token in memory only: do not store it, do not log it.

## API

All methods are `static` on `PaymeMiniappFlutter`. Parameter classes and constants are in `package:payme_miniapp_flutter/payme_miniapp_type.dart`.

The methods return a `Future` that carries no data. Results and business errors arrive on the `onResponse` and `onError` streams. On a bad call (missing arguments, not initialized…), the `Future` throws a `PlatformException` and nothing is done. Wrap your calls in `try/catch`.

### init

```dart
PaymeMiniappFlutter.init(appId, publicKey, privateKey, {locale, env, mode})
```

| Parameter | Description |
| --- | --- |
| `appId` | App ID issued by PayME |
| `publicKey` | PayME's public key |
| `privateKey` | Your app's private key |
| `locale` | `PayMELocale.VI` (default) or `PayMELocale.EN` |
| `env` | `PayMEEnv.PRODUCTION` (default) or `PayMEEnv.SANDBOX` |
| `mode` | Usually omitted. Defaults to `PayMEMode.MINIAPP_SANDBOX` when `env` is `SANDBOX`, otherwise `PayMEMode.MINIAPP_PRODUCT` |

Call `init` once, before any other method. Calling another method before `init` gives `PAYME_NOT_INITIALIZED`.

### open

```dart
PaymeMiniappFlutter.open(type, openingParam)
```

`type`: `PayMEOpeningType.screen` (full screen) or `PayMEOpeningType.modal` (slides up from the bottom; Android needs [`FlutterFragmentActivity`](#android)).

Build `openingParam` with one of the `PayMEOpeningParam` named constructors:

| Constructor | Purpose | `data` |
| --- | --- | --- |
| `open(identityToken:)` | Wallet home (opens activation if the wallet is not active) | None |
| `kyc(identityToken:)` | Identity verification (eKYC) | None |
| `pay(identityToken:, data:)` | Pay for your order | [`PayMEPaymentParam`](#paymepaymentparam-pay) |
| `payment(identityToken:, data:)` | Pay a transaction created via the PayME API | [`PayMEPaymentDirectParam`](#paymepaymentdirectparam-payment) |
| `transferQR(identityToken:, data:)` | Transfer to a bank account | [`PayMETransferQRParam`](#paymetransferqrparam-transferqr) |
| `service(identityToken:, data:)` | Bill payment (electricity, water, cards…) | [`PayMEServiceParam`](#paymeserviceparam-service) |
| `deposit`, `withdraw`, `transfer` `(identityToken:, data:)` | Top up, withdraw, transfer | [`PayMEPayParam`](#paymepayparam-deposit-withdraw-transfer) |

```dart
await PaymeMiniappFlutter.open(
  PayMEOpeningType.screen,
  PayMEOpeningParam.pay(
    identityToken: identityToken,
    data: PayMEPaymentParam(amount: 10000, transactionId: 'ORDER_123', note: 'Order 123'),
  ),
);
// Guest checkout: omit identityToken
await PaymeMiniappFlutter.open(PayMEOpeningType.modal,
    PayMEOpeningParam.payment(data: PayMEPaymentDirectParam(transaction: '<TRANSACTION>')));
await PaymeMiniappFlutter.open(PayMEOpeningType.screen,
    PayMEOpeningParam.service(identityToken: identityToken, data: PayMEServiceParam(service: 'POWE')));
await PaymeMiniappFlutter.open(PayMEOpeningType.screen,
    PayMEOpeningParam.deposit(identityToken: identityToken, data: PayMEPayParam(amount: 50000)));
```

With invalid parameters (missing `data`, a missing required field, `amount` not a whole number), Miniapp does not open and `open` throws a `PlatformException` `INVALID_PARAM`.

`amount` is a whole number of VND (even when its type is `double`), at most 2,147,483,647. Every `data` class below also has the optional field `isShowResult` (show PayME's result screen, default `true`).

#### PayMEPaymentParam (pay)

| Field | Required | Description |
| --- | --- | --- |
| `transactionId` | Yes | Your transaction ID |
| `amount` | Yes | Amount |
| `ipnUrl` | No | URL that receives transaction status notifications |
| `note` | No | Note |

#### PayMEPaymentDirectParam (payment)

| Field | Required | Description |
| --- | --- | --- |
| `transaction` | Yes | Transaction ID PayME returned when you created the transaction |

#### PayMETransferQRParam (transferQR)

| Field | Required | Description |
| --- | --- | --- |
| `amount` | Yes | Amount (`0` lets the user enter it) |
| `bankNumber` | Yes | Receiving account or card number |
| `swiftCode` | Yes | SWIFT code of the receiving bank |
| `cardHolder` | Yes | Account holder name |
| `note` | No | Note |
| `partnerTransaction` | No | Your transaction ID |
| `extraData` | No | Your own data (`Map`), returned when the transaction succeeds |

#### PayMEServiceParam (service)

| Field | Required | Description |
| --- | --- | --- |
| `service` | Yes | `POWE` (electricity), `WATE` (water), `ADSL` (internet), `TIVI` (TV), `PPMB` (postpaid mobile), `MOBILE_CARD` (phone card), `MOBILE_TOPUP` (mobile top-up), `GAME_CARD` (game card) |
| `isBackToApp` | No | Return to your app when done |

#### PayMEPayParam (deposit, withdraw, transfer)

| Field | Required | Description |
| --- | --- | --- |
| `amount` | No | Prefilled amount |
| `description` | No | Transaction description |
| `isBackToApp` | No | Return to your app when done |

### getBalance, getAccountInfo

```dart
PaymeMiniappFlutter.getBalance(identityToken)
PaymeMiniappFlutter.getAccountInfo(identityToken)
```

Results arrive on `onResponse`:

- `GET_BALANCE`: `linked` (whether the wallet is linked), `balance` (present when `linked` is `true`).
- `GET_ACCOUNT_INFO`: wallet account information.

### setLanguage, close

- `setLanguage(PayMELocale.VI | PayMELocale.EN)`: change the Miniapp language.
- `close()`: close the open Miniapp.

### Events

```dart
final errorSub = PaymeMiniappFlutter.onError.listen((e) { /* e['action'], e['code'], e['description'] */ });
final responseSub = PaymeMiniappFlutter.onResponse.listen((e) { /* e['action'], ... */ });
// When no longer needed:
errorSub.cancel();
responseSub.cancel();
```

| Stream | Payload (`Map`) | When |
| --- | --- | --- |
| `onResponse` | `{action, ...data}` | A transaction succeeds, or a `getBalance` / `getAccountInfo` result arrives |
| `onError` | `{action, code, description}` | An error occurs or the user closes Miniapp |

Subscribe once, before calling `open`, `getBalance` or `getAccountInfo`.

### Opening and closing Miniapp

- Only one Miniapp can be open at a time. Calling `open` while one is open gives `MINIAPP_ALREADY_OPEN` on `onError`.
- When Miniapp closes, you get its result exactly once: `onResponse`, `onError`, or `USER_CANCEL` when the user closes it. You can open again right inside that callback.
- Android: back closes a popup, then goes back a page in Miniapp; on the first page it closes Miniapp and reports `USER_CANCEL`.
- `close()`: iOS reports `USER_CANCEL`, Android does not.

## Error codes

Errors come from two places: a `PlatformException` from the `Future` (`e.code`; nothing is done) and the `onError` stream (`event['code']`).

| `code` | Received via | Meaning | What to do |
| --- | --- | --- | --- |
| `401` | `onError` | `identityToken` invalid or expired | Fetch a new `identityToken` and retry |
| `IDENTITY_TOKEN_REQUIRED` | `onError` | `identityToken` missing | Pass an `identityToken` |
| `MISSING_INFO` | `PlatformException` | A required argument is missing (for example an empty key in `init`) | Pass all arguments |
| `INVALID_PARAM` | `PlatformException` | Invalid `open` parameters | Fix them per the tables in [open](#open) |
| `PAYME_NOT_INITIALIZED` | `PlatformException` | `init` not called | Call `init` first |
| `USER_CANCEL` | `onError` | The user closed Miniapp | Not an error |
| `MINIAPP_ALREADY_OPEN` | `onError` | A Miniapp is already open | Wait for it to close, or call `close()` first |
| `MINIAPP_CANNOT_PRESENT` | Both | Miniapp cannot be shown | Call `open` while your app is visible; on Android, `modal` needs `FlutterFragmentActivity` |
| Other codes | Both | Network, server or business error | Show `description` and retry; contact PayME if it persists |

## Upgrading from 2.x

3.x replaces the phone number with `identityToken` and is not backward compatible.

| 2.x | 3.x |
| --- | --- |
| `PayMEOpeningParam.open(phone: phone)` | `PayMEOpeningParam.open(identityToken: identityToken)` |
| `PayMEOpeningParam.pay(phone: phone, data: ...)` (likewise `transferQR`, `service`, `deposit`, `withdraw`, `transfer`, `kyc`) | `PayMEOpeningParam.pay(identityToken: identityToken, data: ...)` |
| `PayMEOpeningParam.payment(data: ...)` | `PayMEOpeningParam.payment(identityToken: identityToken /* optional */, data: ...)` |
| `getBalance(phone)` | `getBalance(identityToken)` |
| `getAccountInfo(phone)` | `getAccountInfo(identityToken)` |

1. Server: integrate the `token-init` API (contact PayME).
2. Change the dependency to `payme_miniapp_flutter: ^3.0.2`, run `flutter pub get`, then `cd ios && pod install --repo-update`.
3. Replace every `phone` with `identityToken`.
4. Handle `401` and `IDENTITY_TOKEN_REQUIRED`; wrap calls in `try/catch` for `PlatformException`.
5. Build iOS with Xcode 26.2 or later.

## Support

PayME Technology Corporation, email [hotro@payme.vn](mailto:hotro@payme.vn). Keys, the `token-init` API documentation and the transaction API for `payment`: contact PayME.

Versions: [pub.dev `payme_miniapp_flutter`](https://pub.dev/packages/payme_miniapp_flutter/versions).
