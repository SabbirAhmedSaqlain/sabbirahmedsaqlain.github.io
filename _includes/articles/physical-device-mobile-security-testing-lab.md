# Building a Physical-Device Mobile Security Testing Lab for Android and iOS

Some mobile security behaviors cannot be evaluated realistically in a simulator. Camera capture, face liveness, Secure Enclave or Android Keystore behavior, biometrics, hardware-backed keys, application attestation, device networking, and presentation attacks all benefit from testing on physical devices.

This guide explains how to build an authorized mobile security lab around a Mac, a dedicated Android phone, and an iPhone. It covers application builds, ADB and Xcode connectivity, network inspection, storage and log review, controlled API manipulation, NID upload testing, liveness presentation attacks, runtime instrumentation, evidence collection, and safe cleanup.

> Use this lab only for applications and systems you own or are explicitly authorized to assess. Keep it connected to staging services, use synthetic test identities, and never expose a debug or instrumented build to real users.

## 1. Why Use Physical Devices

A real-device lab gives access to behavior that may be absent or simplified in emulators:

- actual camera focus, exposure, frame rate, and image processing;
- printed-photo and screen-replay presentation attacks;
- Face ID, Touch ID, and Android biometric prompts;
- Secure Enclave, Keychain, Android Keystore, and hardware-backed keys;
- real Wi-Fi proxy and certificate trust behavior;
- USB debugging and device logs;
- application-switcher snapshots and screen recording;
- rooted or jailbroken resilience testing on dedicated hardware;
- performance, thermal, memory, and network conditions.

Use emulators and simulators for rapid automation, broad OS coverage, and repeatable fixtures. Use physical devices for the security behaviors that depend on real hardware and platform configuration. A mature assessment uses both where appropriate.

## 2. Recommended Lab Architecture

```text
                         Staging network
                               |
                               v
Physical Android ----> Mac security workstation <---- Physical iPhone
       |                 |       |       |                  |
       |                 |       |       |                  |
       |                 |       |       +-- Xcode          +-- Camera/biometrics
       |                 |       +---------- Burp/Proxyman
       |                 +------------------ MobSF/Frida
       +-- Camera/ADB/Keystore
```

The Mac acts as the controlled analysis workstation. It hosts the proxy, static-analysis tools, mobile SDKs, runtime tooling, test evidence, and access to staging logs.

## 3. Safety and Authorization Controls

Before connecting a device, establish:

- written scope and named application packages;
- staging base URLs and test tenant IDs;
- dedicated test accounts and synthetic identity data;
- approved request rates and file sizes;
- whether rooted, jailbroken, or instrumented testing is permitted;
- where screenshots, proxy history, and device logs may be stored;
- how credentials and personal data will be redacted;
- when certificates, builds, accounts, and evidence must be deleted.

Use a dedicated security-test device for rooting, jailbreaking, custom trust stores, or instrumentation. These changes alter the device security model and can expose unrelated applications and data.

## 4. Use Three Deliberate Application Builds

A useful lab separates production behavior from test instrumentation.

### 4.1 Production-Like Staging Build

This build should resemble release security behavior:

- release optimization and signing policy;
- production-equivalent certificate validation or pinning;
- debugging disabled;
- release logging policy;
- staging API endpoint;
- test-only accounts and data.

Use it to determine what a normal attacker sees from the distributed application.

### 4.2 Security Test Debug Build

This build supports controlled inspection:

- debuggable configuration;
- staging environment only;
- test proxy certificate allowed through debug-only configuration;
- additional diagnostic observability without secret logging;
- a visible build watermark;
- distribution restricted to the security team.

### 4.3 Instrumented Test Build

This build permits runtime instrumentation such as Frida Gadget when authorized:

- separate bundle or package identity where practical;
- test signing credentials;
- staging-only endpoint allowlist;
- no production secrets;
- no access to real customer data;
- never submitted to an application store.

Keeping these builds separate prevents a temporary test control from weakening a release build.

## 5. Mac Workstation Tooling

Install only the tools required for the approved test plan.

### Core Platform Tools

- Android Studio and Android Platform Tools;
- Xcode and command-line developer tools;
- a modern browser for proxy configuration and documentation;
- Git for versioned test plans and remediation tracking.

### Analysis Tools

- Burp Suite, Proxyman, or mitmproxy for controlled traffic inspection;
- [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF) for automated static and dynamic analysis;
- JADX and apktool for authorized Android package review;
- Ghidra or Hopper for deeper native analysis;
- [Frida](https://frida.re/docs/home/) and Objection for permitted runtime inspection;
- LLDB for source-level debugging of owned iOS builds.

Verify versions before testing and record them in the assessment notes. Tool behavior changes, and findings must be reproducible.

## 6. Connect a Physical Android Device

Enable Developer Options and USB debugging on the dedicated Android test phone, then connect it by USB.

Verify connectivity:

```bash
adb version
adb devices -l
```

The phone should appear with status `device`. If it appears as `unauthorized`, unlock it and approve the workstation's debugging key.

Use [Android's hardware-device documentation](https://developer.android.com/studio/run/device) and [ADB reference](https://developer.android.com/tools/adb) for platform-specific setup details.

Record:

- device manufacturer and model;
- Android version and security patch level;
- CPU architecture;
- bootloader and root state;
- application package and version;
- whether the build is debug or release-like.

## 7. Identify and Inspect the Android Application

Find the owned test package:

```bash
adb shell pm list packages
adb shell dumpsys package com.example.securitytest
```

Review the package details for:

- version code and version name;
- requested and granted permissions;
- exported activities, services, receivers, and providers;
- debuggable and backup settings;
- deep links and App Links;
- network-security configuration;
- supported SDK versions.

Static analysis and manifest review should be performed against the exact build used during dynamic testing.

## 8. Android Log Leakage Testing

Clear irrelevant historical output, start log capture, and exercise the application:

```bash
adb logcat -c
adb logcat
```

Perform controlled workflows:

- login and logout;
- OTP request and verification;
- NID front and back capture;
- upload and OCR;
- liveness and face verification;
- token refresh;
- error and retry paths.

Search the captured evidence for:

```text
password
token
authorization
bearer
refresh
otp
nid
face
secret
```

The release-like build should not expose credentials, complete tokens, identity numbers, document text, private file paths, or raw API payloads. Redact any discovered secrets immediately in reports.

## 9. Android Local Storage Inspection

For an authorized debuggable build, check whether `run-as` is available:

```bash
adb shell run-as com.example.securitytest pwd
```

Inspect application-owned files without copying unrelated device data. Relevant locations include:

- `files`;
- `databases`;
- `shared_prefs`;
- `cache`;
- WebView data;
- app-specific external storage.

For an NID/eKYC application, look specifically for:

- front and back NID images;
- face or liveness frames;
- OCR responses;
- identity number, birth date, and address;
- access and refresh tokens;
- passwords, PINs, and OTPs;
- cryptographic keys;
- debug exports and crash attachments.

Confirm that temporary media is deleted after upload, cancellation, logout, background termination, and failed retries. Verify that backup policy does not copy sensitive data unexpectedly.

## 10. Configure an Android Device for Controlled Proxying

Place the Mac and phone on an isolated network where they can reach each other. Configure the proxy listener on a test-only port and bind it only as broadly as the lab requires.

On Android, set the current Wi-Fi network to use the Mac's IP address and proxy port. Confirm basic connectivity before launching the application.

Modern Android applications do not automatically trust every user-installed certificate. For an application you own, use an Android debug-only Network Security Configuration rather than weakening release security.

An example for the security-test debug build is:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

Reference it from the debug build's application manifest:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config">
</application>
```

Android documents these [debug-only trust anchors](https://developer.android.com/privacy-and-security/security-config). Confirm that the release manifest and merged release resources do not include the test override.

## 11. Inspect Mobile API Traffic

With the security-test build connected through the proxy, exercise each important flow and create an endpoint inventory.

For an identity application, this may include:

```text
/api/login
/api/otp/request
/api/otp/verify
/api/session/refresh
/api/nid/upload
/api/nid/extract
/api/liveness/challenge
/api/liveness/verify
/api/face/match
/api/profile
```

For each endpoint, record:

- HTTP method and route;
- authentication requirement;
- request and response schema;
- object identifiers;
- authorization decision;
- retry and idempotency behavior;
- personal data returned;
- cache headers;
- error behavior;
- latency and timeout behavior.

Do not retain real passwords, full tokens, NID numbers, or face images in shared project files. Use proxy redaction and replace values with stable evidence labels.

## 12. Controlled Parameter and Authorization Tests

Send requests from your own test account to the proxy's repeater tool. Change one field at a time so results remain interpretable.

Examples include:

- your controlled application ID to another controlled application ID;
- front-document type to back-document type;
- a mutable `verified` value from `false` to `true`;
- a permitted tenant ID to a second test tenant;
- a workflow step submitted before its prerequisite;
- the same idempotency key used twice.

Expected results include `403`, an intentionally non-revealing `404`, or a domain-specific validation error. A `200` response containing another user's data or accepting client-owned security state is a serious finding.

Keep server logs available during testing so developers can distinguish validation, authentication, and authorization failures.

## 13. Session and Token Lifecycle Tests

Using only designated accounts, test:

```text
login -> capture session -> logout -> replay
login -> capture session -> password change -> replay
refresh -> rotate token -> replay old refresh token
disable account -> replay existing session
```

Record access-token lifetime, refresh-token lifetime, rotation behavior, reuse detection, revocation timing, concurrent session policy, and whether high-risk operations require recent authentication.

Avoid pasting complete tokens into tickets or screenshots. A hash prefix or a short redacted fragment is enough to correlate evidence.

## 14. OTP Testing on a Real Device

Use provider-approved test numbers or a staging OTP service. Check:

- wrong-code attempt limits;
- resend throttling;
- expiration;
- one-time use;
- invalidation of older codes;
- account and action binding;
- race conditions between simultaneous requests;
- safe handling of delivery failure;
- resistance to account enumeration.

Do not perform uncontrolled brute force. The objective is to verify that agreed controls activate at the expected threshold without harming real users or third-party messaging services.

## 15. Build an NID Upload Test Corpus

Prepare synthetic or approved documents in a versioned, access-controlled test-data directory.

Suggested categories:

```text
valid/
  nid_front.jpg
  nid_back.jpg

wrong_document/
  passport.jpg
  driving_license.jpg
  employee_card.jpg

presentation/
  nid_screenshot.jpg
  nid_on_phone.jpg
  nid_photocopy.jpg

tampered/
  edited_number.jpg
  replaced_portrait.jpg
  mismatched_front_back.jpg

malformed/
  empty.jpg
  corrupted.jpg
  extreme_dimensions.jpg
  wrong_type_named_jpg.jpg
```

For every sample, define the expected server result before execution. Verify actual media decoding, dimensions, document type, side classification, field consistency, duplicate detection, authorization, and safe storage.

Never commit real identity documents to a public repository.

## 16. Test Liveness on a Physical Camera

Physical cameras make presentation-attack testing realistic. Prepare a controlled matrix:

| Test | Expected result |
| --- | --- |
| Genuine live user | Accept |
| Printed face | Reject |
| Face photograph on another phone | Reject |
| Recorded video | Reject |
| Looping or slow-motion video | Reject |
| Multiple faces | Reject |
| Partial face | Reject or challenge |
| Mask or sunglasses | Follow documented risk policy |

Repeat tests across:

- bright and low light;
- glare and screen reflections;
- different distances and angles;
- front and rear cameras if supported;
- slow and fast movement;
- poor and strong network conditions;
- application background and resume.

For active liveness, verify challenge unpredictability, short expiry, one-time use, order enforcement, and server validation. A recorded challenge response must not be reusable for a new session.

Measure false acceptance and false rejection, not only individual pass/fail examples.

## 17. Runtime Testing on Android

There are two common authorized configurations.

### Non-Rooted Dedicated Device

Use an instrumented build that contains Frida Gadget or equivalent test hooks. This is suitable when you own the source and want to examine application behavior without rooting the phone.

### Rooted Dedicated Device

A rooted lab device permits deeper process and filesystem analysis. Keep it isolated from personal accounts and follow [Frida's Android documentation](https://frida.re/docs/android/) for the matching architecture and tool version.

Runtime tests should answer questions such as:

- Can local biometric, KYC, or liveness flags be changed?
- Does changing them unlock only UI, or does the backend also accept them?
- Can sensitive material be observed in memory longer than necessary?
- Can certificate validation or root checks be modified?
- Does the application detect and respond according to its threat model?

The most important result remains server-side enforcement. A modified client must not create trusted KYC, role, payment, or liveness state.

## 18. Connect a Physical iPhone

Connect the iPhone by USB, trust the Mac, open Xcode, and select the phone as the run destination. Enable Developer Mode when required for development and debugging workflows.

Record:

- iPhone model;
- iOS version;
- application bundle ID and build number;
- signing team and configuration;
- entitlements;
- jailbreak state;
- whether the build is debug or release-like.

Use Apple's [Developer Mode documentation](https://developer.apple.com/documentation/xcode/enabling-developer-mode-on-a-device) for the current platform steps.

## 19. Configure iPhone Traffic Inspection

Put the Mac and iPhone on the controlled lab network. Configure a proxy listener, then set the iPhone's current Wi-Fi proxy to the Mac address and selected port.

Install the test proxy CA only on the dedicated device. On iOS, installing a profile and enabling full trust are separate actions. Follow the proxy vendor's current device instructions, such as [PortSwigger's iOS setup](https://portswigger.net/burp/documentation/desktop/mobile/config-ios-device).

Test Safari first to confirm the proxy path. Then run the security-test application and inspect permitted staging traffic.

After testing:

- remove the proxy configuration;
- remove the test CA profile;
- disable trust for the CA;
- delete captured secrets and personal data;
- verify ordinary network behavior is restored.

## 20. Inspect iOS Storage and Privacy Surfaces

With an owned debug build and Xcode tools, examine application-owned storage and behavior:

- UserDefaults;
- Keychain items and accessibility class;
- Documents, Library, Caches, and temporary directories;
- Core Data or SQLite;
- WebKit storage;
- unified logs and crash reports;
- screenshots and application-switcher previews;
- clipboard use;
- backup inclusion.

For NID/eKYC, verify deletion of captured documents and face frames after success, failure, cancellation, logout, and application termination. Confirm that the Keychain stores only necessary secrets and uses an appropriate accessibility policy.

## 21. Runtime Testing on an Owned iOS Build

For source-owned, debuggable applications, Xcode and LLDB provide the first layer of runtime testing. Frida also documents workflows for supported iOS devices and instrumented applications.

Test whether local values such as these are incorrectly treated as authoritative:

```text
isBiometricVerified
isNIDVerified
isLivenessPassed
isPremium
isAdministrator
```

Changing a local value may alter presentation, but the backend must reject unauthorized data and operations. Keep instrumentation restricted to the dedicated test build and staging environment.

## 22. Test Screenshots, Recording, and Backgrounding

On both platforms, inspect:

- sensitive-screen screenshots;
- screen recordings;
- app-switcher cards;
- background snapshots;
- interrupted camera sessions;
- task switching during OTP or payment approval;
- screen mirroring where relevant.

When an NID or face is visible, move the app to the background and verify that the operating-system preview does not preserve sensitive content contrary to product policy.

## 23. Collect Reproducible Evidence

Create one evidence folder per finding:

```text
FINDING-001/
  summary.md
  request-redacted.txt
  response-redacted.txt
  screenshot-redacted.png
  device-and-build.txt
  server-log-reference.txt
```

Evidence should include:

- UTC timestamp;
- application build identifier;
- device and OS version;
- test account role, not real identity;
- endpoint and request correlation ID;
- exact expected and actual results;
- a minimal reproducible sequence;
- secrets and personal data redacted.

Prefer correlation IDs over full sensitive payloads. Keep evidence encrypted and access-controlled.

## 24. Restore the Lab After Testing

At assessment completion:

- remove proxy settings from both devices;
- remove test CA certificates and profiles;
- uninstall instrumented builds;
- revoke test sessions and rotate exposed test credentials;
- delete temporary NID and face media;
- stop instrumentation services;
- restore device integrity if required;
- remove staging access no longer needed;
- archive only approved, redacted evidence;
- document the final device state.

Never continue using a rooted, jailbroken, or heavily instrumented security device as a personal device without an appropriate secure reset.

## 25. Recommended Test Order for NID/eKYC

Run tests in an order that finds high-impact server failures early:

1. Establish build, device, account, and staging scope.
2. Inspect network endpoints and sensitive data exposure.
3. Test object-level and function-level authorization.
4. Test access, refresh, logout, password-change, and replay behavior.
5. Test OTP lifecycle and workflow order.
6. Test NID front/back classification, malformed uploads, and spoofing.
7. Test face photos, video replay, and active challenge replay.
8. Inspect local document, face, token, and log retention.
9. Test screenshots, clipboard, and background previews.
10. Modify client-side KYC and liveness state in the instrumented build.
11. Test production-like certificate validation and resilience controls.
12. Retest fixes and restore the lab.

## Final Lab Checklist

- Use written authorization and staging services.
- Keep dedicated physical devices free of personal data.
- Separate release-like, debug, and instrumented builds.
- Ensure debug trust configuration cannot reach release builds.
- Inventory every sensitive endpoint and workflow.
- Use controlled accounts and synthetic NID/face data.
- Test backend authorization independently of UI state.
- Redact tokens and identity data in evidence.
- Measure both security rejection and genuine-user success.
- Remove certificates, proxy settings, test builds, and retained media afterward.

A physical-device lab is most valuable when it produces repeatable evidence about the full system. The objective is not merely to attach a tool or intercept a request. It is to prove whether the mobile client, operating system, network, API, and verification backend preserve security when each layer is placed under realistic, authorized pressure.

## References

- [OWASP Mobile Application Security](https://mas.owasp.org/)
- [Android Debug Bridge](https://developer.android.com/tools/adb)
- [Run applications on an Android hardware device](https://developer.android.com/studio/run/device)
- [Android Network Security Configuration](https://developer.android.com/privacy-and-security/security-config)
- [Frida documentation](https://frida.re/docs/home/)
- [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF)
- [PortSwigger Android device configuration](https://portswigger.net/burp/documentation/desktop/mobile/config-android-device)
- [PortSwigger iOS device configuration](https://portswigger.net/burp/documentation/desktop/mobile/config-ios-device)
- [Apple Developer Mode](https://developer.apple.com/documentation/xcode/enabling-developer-mode-on-a-device)
