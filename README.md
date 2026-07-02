# samsung_wallet

> Add cards to Samsung Wallet from your Flutter app — device support check, card clipping, and ready-to-use "Add to Samsung Wallet" buttons.

[![pub package](https://img.shields.io/pub/v/samsung_wallet.svg)](https://pub.dev/packages/samsung_wallet)
[![pub likes](https://img.shields.io/pub/likes/samsung_wallet)](https://pub.dev/packages/samsung_wallet/score)
[![pub points](https://img.shields.io/pub/points/samsung_wallet)](https://pub.dev/packages/samsung_wallet/score)
![platform](https://img.shields.io/badge/platform-android-brightgreen.svg)
[![license](https://img.shields.io/badge/license-BSD--2--Clause-blue.svg)](LICENSE)

A Flutter plugin that wraps the [Samsung Wallet](https://developer.samsung.com/wallet/api/overview.html) Android integration flow. It lets you check whether the current device supports Samsung Wallet, open the "Add to Wallet" flow with a signed card (`cdata`), and drop in the official Samsung Wallet buttons without shipping the button artwork yourself. It is built directly on top of the Samsung Wallet Android sample code.

> **Platform support:** Android only. There is no iOS implementation.

![demo](docs/demo.gif)

<!-- TODO: add a real screenshot or demo GIF at docs/demo.gif -->

## ✨ Features

- **Device support check** — queries the Samsung Wallet availability endpoint and returns whether the current device (model + country + partner) can use Samsung Wallet.
- **Add to Wallet flow** — opens the Samsung Wallet "Clip" deep link for a given card using your `cardID` and signed `cData`.
- **Impression logging** — connects to your Partner Portal impression URL on initialization.
- **Drop-in buttons** — the `AddToSamsungWalletButton` widget bundles the official Samsung Wallet button assets (16 style variants + a default), so no asset setup is required in your app.
- **Built-in Test Tool button** — `AddToSamsungWalletButton.testTool()` opens Samsung's [Add to Wallet Test Tool](https://developer.samsung.com/wallet/wallettest.html).
- **Debug logging** — support and connection results are logged to the debug console under the `[SAMSUNG WALLET SAMPLE]` tag.

## 🛠 Tech Stack & Architecture

| Layer | Detail |
| --- | --- |
| API surface | `SamsungWallet` (Dart facade) exposing `initialize`, `checkSamsungWalletSupported`, `addCardToSamsungWallet` |
| Platform bridge | `MethodChannel('flutter_samsung_wallet')` via the `plugin_platform_interface` pattern |
| Native (Android) | `SamsungWalletPlugin.java` (Java), `ActivityAware` plugin |
| Button links | [`url_launcher`](https://pub.dev/packages/url_launcher) (Test Tool button) |
| Buttons | Bundled PNG assets shipped inside the package |

**How it works under the hood:**

- **Support check** — the native side issues an HTTP `GET` to `https://api-us3.mpay.samsung.com/wallet/cmn/v2.0/device/available` with the `partnerCode` header and `serviceType` / `modelName` / `countryCode` query parameters, then parses `resultCode` / `resultMessage` / `available` from the JSON response. The call runs on a single-thread `ExecutorService`.
- **Add to Wallet** — launches an `Intent.ACTION_VIEW` to the Samsung Wallet clip deep link `http://a.swallet.link/atw/v1/{cardId}#Clip?cdata={cData}`.
- **Permission guard** — every method call first verifies that `INTERNET` and `ACCESS_NETWORK_STATE` are granted, returning an error otherwise.
- **serviceType** is fixed internally to `"WALLET"`; you never pass it yourself.

## 🚀 Getting Started

### Requirements

| Requirement | Version |
| --- | --- |
| Flutter | `>= 3.0.0` |
| Dart | `>= 2.17.0 < 4.0.0` |
| Android `compileSdkVersion` | `34` |
| Android `minSdkVersion` | `16` |

You will also need Samsung Wallet Partner credentials (`partnerCode`, `cardId`, impression URL, click URL) from the [Samsung Wallet Partners portal](https://developer.samsung.com/wallet/api/overview.html).

### 1. Add the dependency

```yaml
dependencies:
  samsung_wallet: ^1.1.0
```

Then run:

```bash
flutter pub get
```

### 2. Add Android permissions

Add these to your app's `android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

The plugin rejects every call with an error if either permission is missing. See Android's [Connecting to the network](https://developer.android.com/training/basics/network-ops/connecting) guide for details.

### 3. Ensure the binding is initialized

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(MyApp());
}
```

## 📖 Usage

### Initialize

Creating a `SamsungWallet` instance automatically calls `initialize()`, which checks Samsung Wallet support and connects to your impression URL. Results are printed to the debug console.

```dart
import 'package:samsung_wallet/samsung_wallet.dart';

final samsungWallet = SamsungWallet(
  partnerCode: partnerCode,     // required — Partner ID from the Partners portal
  impressionURL: impressionUrl, // required — impression logging URL
  countryCode: countryCode,     // optional — ISO 3166-2 country code
);
```

### Check device support

```dart
final bool? supported = await samsungWallet.checkSamsungWalletSupported(
  partnerCode: partnerCode,
  countryCode: 'KR', // optional
);
```

### Add a card to Samsung Wallet

```dart
Future<void> onTapAddCard() async {
  await samsungWallet.addCardToSamsungWallet(
    cardID: cardId,     // Wallet card identifier from the Partners portal
    cData: cdata,       // signed card payload (see "Creating cdata" below)
    clickURL: clickUrl, // click URL from the Partners portal
  );
}
```

### Add the button widget

The button ships with the official Samsung Wallet artwork — no asset configuration needed.

```dart
// Default button
AddToSamsungWalletButton(onTapAddCard: onTapAddCard)

// Customized button
AddToSamsungWalletButton(
  onTapAddCard: onTapAddCard,
  buttonDesignType: ButtonDesignType.iconBasic,
  buttonTextPositionType: ButtonTextPositionType.hor,
  buttonThemeType: ButtonThemeType.pos,
)
```

If all three style parameters are `null`, the default button image is shown. If any one is set, the remaining parameters fall back to `addTo` / `hor` / `pos`.

#### Button style options

| Parameter | Values | Meaning |
| --- | --- | --- |
| `buttonDesignType` | `addTo`, `basic`, `iconAddTo`, `iconBasic` | Button design (with/without icon, "Add to" vs. basic text) |
| `buttonTextPositionType` | `hor`, `ver` | Horizontal or vertical layout |
| `buttonThemeType` | `pos`, `rev` | `pos` for light theme, `rev` for dark theme |

### Add to Wallet Test Tool

```dart
AddToSamsungWalletButton.testTool()
```

This opens Samsung's [Add to Wallet Test Tool](https://developer.samsung.com/wallet/wallettest.html) in the browser via `url_launcher`.

## 🧩 API Reference

### `SamsungWallet`

| Member | Signature | Notes |
| --- | --- | --- |
| Constructor | `SamsungWallet({String? countryCode, required String impressionURL, required String partnerCode})` | Calls `initialize()` automatically |
| `initialize` | `Future<void> initialize({String? countryCode, required String partnerCode, required String impressionURL})` | Checks support + connects impression URL; logs results |
| `checkSamsungWalletSupported` | `Future<bool?> checkSamsungWalletSupported({String? countryCode, required String partnerCode})` | Returns device support |
| `addCardToSamsungWallet` | `Future<bool?> addCardToSamsungWallet({required String cardID, required String cData, required String clickURL})` | Opens the Add to Wallet flow |

## 📂 Project Structure

```
lib/
  samsung_wallet.dart                     # public exports
  src/
    samsung_wallet.dart                   # SamsungWallet facade (public API)
    samsung_wallet_platform_interface.dart
    samsung_wallet_method_channel.dart    # MethodChannel implementation
    enum/wallet_button_type.dart          # button style enums
    widgets/add_to_samsung_wallet_button.dart
android/
  src/main/java/com/monocsp/samsung_wallet/SamsungWalletPlugin.java
assets/
  wallet/                                 # bundled Samsung Wallet button artwork
example/                                  # runnable example app
```

## 🔐 Creating `cdata`

`cData` is a signed card payload. Generate it with Samsung's JWT Generator — see the [Samsung Wallet sample code](https://developer.samsung.com/wallet/samplecode.html) and the [Security](https://developer.samsung.com/wallet/api/security.html) documentation for details.

## 📚 References

- [Samsung Wallet API overview](https://developer.samsung.com/wallet/api/overview.html)
- [Samsung Wallet sample code](https://developer.samsung.com/wallet/samplecode.html) (this plugin is based on the Android sample)
- [Add to Wallet Test Tool](https://developer.samsung.com/wallet/wallettest.html)

## 🐛 Issues

Please file bugs and feature requests on the [GitHub issue tracker](https://github.com/monocsp/flutter_samsung_wallet/issues).

## 📄 License

BSD 2-Clause License. Copyright (c) 2023, Chan Seob Park. See [LICENSE](LICENSE).
