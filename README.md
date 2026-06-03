# Sword Health QA Challenge — Maestro Mobile Automation

## Overview
Cross-platform mobile test suite for the [Sauce Labs My Demo App](https://github.com/saucelabs/my-demo-app-android) using [Maestro](https://maestro.mobile.dev/). The suite covers authentication, navigation, and checkout flows on both Android and iOS.

---

## Setup Instructions

### Prerequisites
- macOS (tested on MacBook Air M4)
- [Maestro](https://maestro.mobile.dev/) v1.40.1
- Java (Temurin 21+)
- Android Studio with Pixel 8 emulator (API 33+)
- Xcode + iOS Simulator (iPhone 17, iOS 26.5)

### Install Maestro
```bash
export MAESTRO_VERSION=1.40.1
curl -Ls "https://get.maestro.mobile.dev" | bash
```

### Download the Demo Apps
- **Android APK:** [mda-2.2.0-25.apk](https://github.com/saucelabs/my-demo-app-android/releases/tag/2.2.0)
- **iOS Simulator:** [SauceLabs-Demo-App.Simulator.zip](https://github.com/saucelabs/my-demo-app-ios/releases)

### Install on Android Emulator
```bash
adb install ~/Downloads/mda-2.2.0-25.apk
```

### Install on iOS Simulator
```bash
xcrun simctl boot CDA7B182-4BD7-41E5-AE9E-4C28D5413528
open -a Simulator
unzip ~/Downloads/SauceLabs-Demo-App.Simulator.zip -d ~/Downloads/SauceLabs-iOS
xcrun simctl install booted "/Users/macbook/Downloads/SauceLabs-iOS/Payload/My Demo App.app"
```
---
## How to Run the Tests

### Android
```bash
# Run all Android tests
maestro test flows/android/login/ && maestro test flows/android/navigation/ && maestro test flows/android/checkout/

# Run a specific flow
maestro test flows/android/login/01-login-success.yaml
```

### iOS
```bash
# Run all iOS tests
maestro test flows/ios/01-login-success.yaml && maestro test flows/ios/03-product-catalog.yaml && maestro test flows/ios/04-product-detail.yaml && maestro test flows/ios/05-side-menu.yaml

# Run a specific flow
maestro test flows/ios/01-login-success.yaml
```

### Running with both devices connected
When both Android emulator and iOS simulator are running, specify the device:
```bash
# Android
maestro --device emulator-5554 test flows/android/login/01-login-success.yaml

# iOS
maestro --device CDA7B182-4BD7-41E5-AE9E-4C28D5413528 test flows/ios/01-login-success.yaml
```
---

## Project Structure

flows/
├── android/
│   ├── login/
│   │   ├── 01-login-success.yaml
│   │   ├── 02-login-locked-out.yaml
│   │   ├── 03-login-empty-credentials.yaml
│   │   └── 04-login-logout.yaml
│   ├── navigation/
│   │   ├── 01-product-catalog.yaml
│   │   ├── 02-product-detail.yaml
│   │   └── 03-side-menu.yaml
│   └── checkout/
│       └── 01-checkout.yaml
├── ios/
│   ├── 01-login-success.yaml
│   ├── 02-login-locked.yaml          # Skipped — see Known Limitations
│   ├── 03-product-catalog.yaml
│   ├── 04-product-detail.yaml
│   └── 05-side-menu.yaml
└── shared/
└── helpers/
├── android-login-helper.yaml
├── android-dismiss-popup-helper.yaml
└── ios-login-helper.yaml

---

## Test Coverage

### Android ✅
| Flow | Scenario |
|------|----------|
| `01-login-success` | Valid credentials → product catalog |
| `02-login-locked-out` | Locked account → error message |
| `03-login-empty-credentials` | Empty submit → validation feedback |
| `04-login-logout` | Login → logout → session cleared |
| `01-product-catalog` | Catalog loads with products visible |
| `02-product-detail` | Product detail shows name, price, description; back returns to catalog |
| `03-side-menu` | Menu opens, shows options, closes correctly |
| `01-checkout` | Full checkout: cart → shipping → payment → order confirmation |

### iOS ✅ (partial)
| Flow | Scenario | Status |
|------|----------|--------|
| `01-login-success` | Valid login → product catalog | ✅ Passing |
| `02-login-locked` | Locked account → error | ⚠️ Skipped — keyboard limitation |
| `03-product-catalog` | Catalog loads with products | ✅ Passing |
| `04-product-detail` | Product detail screen | ✅ Passing |
| `05-side-menu` | Menu opens, shows options, closes | ✅ Passing |

---

## Design Decisions

### Platform-specific flows with shared helpers
Android flows live in `flows/android/`, iOS in `flows/ios/`. Reusable login and popup logic is extracted into `flows/shared/helpers/` and called via `runFlow`. Each helper includes its own `appId` as required by Maestro 1.40.1.

### Resource IDs over text labels (Android)
Form fields use full resource IDs (e.g. `com.saucelabs.mydemoapp.android:id/cityET`) rather than text labels. Text labels disappear after tapping and the keyboard toolbar interferes with element detection, causing flaky selectors.

### Auto-fill list for iOS login
iOS login uses the credential auto-fill list (`tapOn: "bob@example.com"`) rather than typing into fields. This avoids the keyboard dismissal issue that blocks the Login button on iOS in Maestro 1.40.1.

### Maestro version pinned to 1.40.1
Maestro 2.6.0 has an incompatibility with Java 26 causing ADB TCP connection failures. v1.40.1 is stable for this setup.

### hideKeyboard after form sections (Android)
After filling grouped form fields, `hideKeyboard` is called before tapping navigation buttons. Without this, the keyboard overlaps buttons and causes tap failures.

---

## Cross-Platform Differences (Task 3)

Both platforms were tested. Key differences found:

| Area | Android | iOS |
|------|---------|-----|
| App ID | `com.saucelabs.mydemoapp.android` | `com.saucelabs.mydemo.app.ios` |
| Navigation | Hamburger menu (top left) | Bottom tab bar ("More" tab) |
| Login menu item ID | `menuIV` resource ID | `LogOut-menu-item` accessibility ID |
| Field selectors | Full resource IDs | Text labels or point coordinates |
| Username field label | "Username" | "User Name" (two words) |
| Valid credentials | `bod@example.com` | `bob@example.com` |
| Popup on launch | 16KB compatibility warning | None |
| Back navigation | Hardware back button (`pressKey: Back`) | Tap back arrow or swipe |
| Keyboard dismiss | `hideKeyboard` works | `hideKeyboard` does not work in v1.40.1 |
| Validation errors | Inline text below field | Modal dialog with OK button |
| Products title selector | `assertVisible: "Products"` | `assertVisible: id: "title"` |
| Product item selector | `productIV` resource ID | `ProductItem` accessibility ID |

---

## Bugs Found During Testing

### Bug 1 — 16KB Page Alignment Popup (Android)
**Severity:** Medium
**Description:** On every fresh launch, Android shows a system warning about the app not being 16KB page-aligned, blocking automation.
**Workaround:** `optional: true` tap in dismiss-popup helper.

### Bug 2 — Sequential Field Validation on Login (Android)
**Severity:** Low
**Description:** Submitting empty login form shows only "Username is required". Password is only validated after username is filled.
**Expected:** Both errors shown simultaneously.
**Reproduced by:** `03-login-empty-credentials.yaml`

### Bug 3 — Username Field Concatenation (Android)
**Severity:** Low
**Description:** On re-runs without `clearState`, the username field retains previous content and new input is appended.
**Workaround:** Always use `clearState: true` on `launchApp`.

### Bug 4 — Session Persistence Despite clearState (Android)
**Severity:** Medium
**Description:** In some runs, `clearState: true` does not fully reset the session, causing the app to land on Products instead of the guest catalog.

### Bug 5 — Logout Confirmation Dialog (Android)
**Severity:** Low
**Description:** Tapping "Log Out" shows a confirmation dialog requiring a second tap on "LOGOUT".

### Bug 6 — iOS Keyboard Blocks Login Button (iOS)
**Severity:** Medium
**Description:** After typing in the password field on iOS, the software keyboard cannot be dismissed programmatically in Maestro 1.40.1. The Login button is obscured and cannot be tapped.
**Impact:** Flows requiring manual credential input on iOS cannot complete login.
**Workaround:** Use the credential auto-fill list which avoids keyboard entirely.

### Bug 7 — iOS Validation Uses Modal Dialog (iOS)
**Severity:** Low / UX
**Description:** iOS shows validation errors as a blocking modal dialog ("Validation Error! / Username is required") rather than inline field errors as on Android. Inconsistent UX across platforms.

---

## Known Limitations & Trade-offs

- **iOS checkout not automated:** Checkout requires typing in multiple form fields. The iOS keyboard dismissal bug (Bug 6) blocks this flow. Android checkout is fully automated end-to-end.
- **iOS locked account flow skipped:** `02-login-locked.yaml` is documented but not runnable — blocked by the same keyboard limitation. See inline comments in the file.
- **Checkout resource IDs inferred:** Field IDs (`fullNameET`, `cityET`, etc.) were inferred from naming conventions and validated at runtime. May break if the app is updated.

---

## Versions Used
- Maestro: 1.40.1
- Android APK: mda-2.2.0-25 (version 2.2.0)
- iOS App: SauceLabs-Demo-App.Simulator.zip (version 2.2.2)
- Android Emulator: Pixel 8, API 33
- iOS Simulator: iPhone 17, iOS 26.5