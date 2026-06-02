# Sword Health QA Challenge — Maestro Mobile Automation

## Overview
Cross-platform mobile test suite for the [Sauce Labs My Demo App](https://github.com/saucelabs/my-demo-app-android) using [Maestro](https://maestro.mobile.dev/). The suite covers authentication, navigation, and checkout flows on Android, with iOS structure prepared for extension.

---

## Setup Instructions

### Prerequisites
- macOS (tested on MacBook Air M4)
- [Maestro](https://maestro.mobile.dev/) v1.40.1
- Java (Temurin 21+)
- Android Studio with Pixel 8 emulator (API 33+)
- Xcode + iOS Simulator (for iOS flows)

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

---

## How to Run the Tests

### Run all tests
```bash
cd ~/sword-maestro-tests
maestro test flows/shared/login flows/shared/navigation flows/shared/checkout
```

### Run a specific flow
```bash
maestro test flows/shared/login/01-login-success.yaml
```

---

## Project Structure

```
flows/
├── android/          # Android-specific overrides
├── ios/              # iOS-specific overrides  
└── shared/           # Cross-platform flows
    ├── helpers/
    │   ├── dismiss-popup-helper.yaml
    │   └── login-helper.yaml
    ├── login/
    │   ├── 01-login-success.yaml
    │   ├── 02-login-locked-out.yaml
    │   ├── 03-login-empty-credentials.yaml
    │   └── 04-login-logout.yaml
    ├── navigation/
    │   ├── 01-product-catalog.yaml
    │   ├── 02-product-detail.yaml
    │   └── 03-side-menu.yaml
    └── checkout/
        └── 01-checkout.yaml
```

---

## Design Decisions

### Shared flows over platform duplication
All flows live in `shared/` by design. The `android/` and `ios/` folders are reserved for platform-specific overrides if behavioral differences are found. This avoids duplication and makes maintenance easier.

### Resource IDs over text labels for inputs
Form fields are targeted using resource IDs (e.g. `com.saucelabs.mydemoapp.android:id/nameET`) rather than placeholder text, because text labels disappear after tapping and the keyboard toolbar interferes with element detection.

### No reusable subflows via runFlow
Maestro 1.40.1 requires `appId` in every flow file, including subflows called via `runFlow`. This causes subflows to behave as independent flows rather than composable units. Helper files exist in the project but flows are currently inlined for reliability. This is a known limitation documented at [Maestro GitHub Issues](https://github.com/mobile-dev-inc/maestro/issues).

### Maestro version pinned to 1.40.1
Maestro 2.6.0 (latest at time of writing) has an incompatibility with Java 26 that causes an ADB TCP connection failure. Downgrading to 1.40.1 resolves this.

---

## Cross-Platform Notes (Task 3)

All flows were written and validated on Android (Pixel 8, API 33). The `shared/` structure is intentionally platform-agnostic.

### iOS Status
iOS Simulator (iPhone 17, iOS 26.5) was configured and the Maestro device list confirmed availability. The iOS app (`SauceLabs-Demo-App.Simulator.zip`) was downloaded. Full iOS flow execution was not completed due to time constraints, but the following differences are expected based on app inspection:

| Area | Android | iOS |
|------|---------|-----|
| Element IDs | `com.saucelabs.mydemoapp.android:id/nameET` | Accessibility labels (no resource IDs) |
| App ID | `com.saucelabs.mydemoapp.android` | `com.saucelabs.mydemo.app.ios` |
| Popup | 16KB compatibility warning | No equivalent |
| Navigation | Hardware back button | Swipe back gesture |

To run on iOS, flows would need `appId` updated and selectors changed to use `accessibilityLabel` or visible text.

---

## Bugs Found During Testing

### Bug 1 — 16KB Page Alignment Compatibility Popup
**Severity:** Medium  
**Description:** On every fresh app launch (`clearState: true`), Android displays a system-level warning about the app not being 16KB page-aligned. This blocks automation until dismissed.  
**Impact:** Requires `optional: true` tap workaround in every flow.  
**Expected:** Warning should not appear on a production app, or should be dismissible persistently.

### Bug 2 — Sequential Field Validation on Login
**Severity:** Low  
**Description:** Submitting the login form with both fields empty only shows "Username is required". The password field is only validated after the username is filled.  
**Expected:** Both validation errors should appear simultaneously.  
**Reproduced by:** `03-login-empty-credentials.yaml`

### Bug 3 — Username Field Auto-Focus Issue
**Severity:** Low  
**Description:** After navigating to the login screen via the side menu, text input behaves inconsistently when targeting fields by text label. Both username and password text sometimes typed into the username field.  
**Workaround:** Use resource IDs (`nameET`, `passwordET`) to target fields reliably.  
**Expected:** Fields should be individually focusable by label.

### Bug 4 — Session Persistence Despite clearState
**Severity:** Medium  
**Description:** In some runs, `clearState: true` does not fully reset the login session, causing the app to land on the Products screen instead of the Login screen.  
**Impact:** Flows that assume a logged-out state may fail intermittently.  
**Expected:** `clearState: true` should guarantee a clean app state on every launch.

### Bug 5 — Logout Confirmation Dialog
**Severity:** Low / UX  
**Description:** Tapping "Log Out" in the side menu shows a confirmation dialog requiring a second tap on "LOGOUT". This is an extra step not mentioned in the app's UX flow.  
**Note:** This may be intentional to prevent accidental logout, but worth flagging.

---

## Known Limitations & Trade-offs

- **iOS flows not executed:** iOS Simulator was configured but full flow execution was deferred due to time constraints. Structure and documentation are in place for extension.
- **Helpers not fully reusable:** Due to Maestro 1.40.1 subflow limitations, login steps are inlined in most flows rather than called via `runFlow`.
- **Checkout flow uses guessed resource IDs:** Some checkout form field IDs (`fullNameET`, `address1ET`, `cityET`, etc.) were inferred from naming conventions. If the app is updated these may break.

---

## Versions Used
- Maestro: 1.40.1
- Android APK: mda-2.2.0-25 (version 2.2.0)
- iOS App: SauceLabs-Demo-App.Simulator.zip (version 2.2.2)
- Android Emulator: Pixel 8, API 33
- iOS Simulator: iPhone 17, iOS 26.5