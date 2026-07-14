# Mobile Application Security Testing: A Complete Android and iOS Guide

Mobile applications process credentials, tokens, personal information, payment data, identity documents, camera images, and biometric decisions. A secure assessment must therefore examine more than the visible screens. It must test the mobile client, operating-system storage, network channel, API authorization, backend workflow, and resistance to tampering as one connected system.

This guide presents an authorized, defensive methodology for assessing Android and iOS applications. It is especially useful for banking, fintech, healthcare, identity verification, NID/eKYC, and other high-trust mobile products.

> Perform these tests only on applications, devices, accounts, and environments you own or have explicit permission to assess. Use dedicated test data and staging systems whenever possible.

## 1. Start with Scope and Threat Modeling

Before opening a testing tool, define what is in scope:

- Android and iOS application identifiers;
- production-like staging API endpoints;
- test user roles and tenant boundaries;
- permitted physical devices and operating-system versions;
- allowed network interception and runtime instrumentation;
- test payment, OTP, NID, face, and biometric data;
- prohibited destructive actions;
- evidence handling and deletion requirements;
- assessment start and end dates.

Map sensitive assets before selecting tests. Common assets include:

- passwords, PINs, OTPs, and recovery codes;
- access and refresh tokens;
- NID or passport numbers and images;
- face images, liveness recordings, and biometric templates;
- bank balances, payment instructions, and account numbers;
- private API keys and cryptographic material;
- user roles, KYC status, and transaction approval state.

A useful threat model asks four questions:

1. What must remain confidential?
2. What decisions must not be modified by the client?
3. Which operations require fresh user authorization?
4. What happens if the device or application process is compromised?

## 2. Use OWASP MASVS as the Assessment Backbone

The [OWASP Mobile Application Security Verification Standard](https://mas.owasp.org/MASVS/) organizes mobile security requirements into practical control groups. The [OWASP Mobile Application Security Testing Guide](https://mas.owasp.org/MASTG/) provides techniques for testing them.

An assessment should cover:

- secure local storage;
- cryptography and key management;
- authentication and authorization;
- secure network communication;
- platform interaction;
- code quality and configuration;
- resilience against reverse engineering and tampering;
- privacy and minimization of personal data.

MASVS is a baseline, not a complete product threat model. An eKYC application also needs domain-specific tests for identity-document spoofing, face presentation attacks, replay, workflow bypass, and backend verification integrity.

## 3. Build a Controlled Security Test Matrix

Create a matrix before testing so every finding has a requirement, expected result, and owner.

| Area | Example test | Secure expectation |
| --- | --- | --- |
| Network | Intercept traffic through a controlled proxy | Sensitive traffic remains protected and validated |
| API | Change an object identifier | Server denies access to another user's object |
| Session | Replay a token after logout | Revoked session is rejected |
| Storage | Inspect app files and caches | Sensitive data is absent or appropriately protected |
| Upload | Rename malformed data as an image | Server validates decoded content and rejects it |
| Liveness | Present a printed or replayed face | Presentation attack is rejected |
| Tampering | Modify a client-side verification flag | Backend independently rejects invalid state |

Record the application version, device, OS version, account role, endpoint, input, actual result, expected result, and evidence for every test.

## 4. Network Interception and Manipulation

### Security Objective

An attacker controlling a network should not be able to read or modify sensitive application traffic.

Use a controlled proxy such as Burp Suite, Proxyman, or mitmproxy in a permitted staging environment. Exercise important workflows:

- login and registration;
- password reset and OTP verification;
- token refresh and logout;
- profile and account updates;
- NID or image upload;
- face verification and liveness;
- payment or transaction approval.

Look for sensitive values in URLs, headers, request bodies, responses, and error messages:

- credentials;
- bearer tokens;
- API keys;
- identity numbers;
- raw document or face images;
- internal server details;
- device identifiers.

Verify that:

- HTTPS is used for all sensitive endpoints;
- certificate and hostname validation are correct;
- cleartext fallback is disabled;
- secrets are not placed in URLs;
- responses do not expose excessive personal data;
- error messages do not reveal internal implementation details.

### Certificate Pinning

For developer-controlled, high-value endpoints, certificate or public-key pinning can add defense in depth. Test the production-like build with an untrusted interception certificate. It should reject the connection according to the product's security policy.

Pinning does not replace server authentication, authorization, short-lived sessions, or secure storage. A compromised client may still bypass local pinning logic through runtime modification, so the server must remain secure even when client controls fail.

## 5. API Authorization and IDOR/BOLA

Broken Object Level Authorization occurs when an API accepts an object identifier but does not verify that the authenticated user may access that object.

A controlled example is:

```http
GET /api/applications/5001
Authorization: Bearer TEST_USER_A_TOKEN
```

Change only the identifier to another controlled record:

```http
GET /api/applications/5002
Authorization: Bearer TEST_USER_A_TOKEN
```

The secure result is `403 Forbidden` or a deliberately non-revealing `404 Not Found`. Returning another user's record is a critical authorization failure.

Test object boundaries for:

- profiles and addresses;
- uploaded documents;
- NID verification records;
- applications and transactions;
- support tickets;
- organization or tenant resources;
- reports and generated files.

Never trust `user_id`, `tenant_id`, `role`, `is_verified`, or similar values because the mobile client supplied them. The server must derive identity from the authenticated session and enforce authorization on every operation.

## 6. Authentication Workflow Bypass

Mobile screens are not security boundaries. Hiding a dashboard until OTP or biometric success does not protect the API.

Model the intended state transition:

```text
credentials accepted
    -> OTP verified
    -> device or biometric requirement satisfied
    -> authenticated session issued
    -> sensitive operation authorized
```

Then test whether later endpoints independently require the correct server-side state. Examples include:

- calling profile APIs before OTP verification;
- calling payment APIs before transaction approval;
- skipping onboarding or KYC screens;
- changing a client field from `false` to `true`;
- reusing an old workflow identifier;
- performing steps in the wrong order;
- submitting the same completion request twice.

The backend should maintain the authoritative workflow state and reject missing, expired, repeated, or out-of-order steps.

## 7. OTP Security Testing

Use designated test accounts and agreed rate limits. Verify:

- OTP lifetime;
- one-time use enforcement;
- attempt limits;
- account-level throttling;
- device and network throttling where appropriate;
- invalidation after a new OTP is issued;
- binding to the intended account, action, and transaction;
- resistance to concurrent verification attempts;
- safe error messages that do not enable account enumeration.

Useful controlled sequences include:

```text
request OTP A -> request OTP B -> submit OTP A
```

and:

```text
verify OTP B -> submit OTP B again
```

The expected behavior must match a documented server policy. High-value transaction OTPs should be bound to the exact operation, not merely to a login session.

## 8. JWT and Session Replay

Capture a token belonging to your own test account and verify the complete session lifecycle.

### Logout Test

```text
login -> capture Token A -> logout -> replay Token A
```

The result should match the documented revocation model. If access tokens are intentionally valid until a short expiry, high-risk operations should still require appropriate server-side checks or fresh authorization.

### Password Change Test

```text
login -> capture Token A -> change password -> replay Token A
```

### Device and Refresh Tests

Verify:

- access-token expiration;
- refresh-token rotation;
- reuse detection for rotated refresh tokens;
- session revocation after password or security changes;
- concurrent session policy;
- device binding where justified by the threat model;
- cryptographic algorithm and signature validation;
- rejection of altered, expired, or incorrectly scoped tokens.

Do not treat token contents as authorization simply because a signature is valid. The server must also check issuer, audience, expiry, session state, user status, and operation-specific permissions.

## 9. Secure Local Storage

Assume an attacker gains temporary device access, obtains a backup, or controls a rooted or jailbroken test device.

Inspect permitted application locations:

### Android

- SharedPreferences or DataStore;
- SQLite and Room databases;
- internal and external files;
- caches and temporary files;
- WebView data;
- logs and crash reports;
- backup artifacts.

### iOS

- UserDefaults;
- Documents and Library directories;
- Caches and temporary files;
- Core Data or SQLite stores;
- property lists;
- WebKit data;
- logs and crash reports;
- device backups.

Search for access tokens, refresh tokens, passwords, NID data, OCR output, addresses, face images, recovery codes, and private keys.

Use platform-protected facilities such as [Android Keystore](https://developer.android.com/privacy-and-security/keystore) and [Apple Keychain data protection](https://support.apple.com/guide/security/keychain-data-protection-secb0694df1a/web) for appropriate secrets and cryptographic keys. Minimize what is stored, clear temporary media promptly, and apply the strongest suitable data-protection class.

## 10. Logging, Clipboard, Screenshots, and App Switcher

Sensitive data frequently escapes through secondary operating-system features.

### Logs

Exercise login, NID capture, upload, liveness, OTP, refresh, and logout while monitoring release-build logs. Passwords, tokens, identity data, image paths, and complete API payloads should not be logged.

### Clipboard

Check whether the application unnecessarily copies OTPs, account numbers, identity numbers, tokens, or recovery codes. Where copying is required, minimize retention and use platform privacy controls.

### Screenshots and Recording

Test sensitive screens for:

- screenshots;
- screen recording;
- application-switcher previews;
- background snapshots;
- screen sharing or mirroring.

Protection should be risk-based. NID images, passwords, OTPs, card details, balances, and recovery codes generally need stronger treatment than ordinary content.

## 11. Biometric Authentication Logic

Biometrics can unlock a local secret or authorize use of a platform-protected key. A mutable Boolean such as `biometricPassed = true` is not sufficient protection for a high-value backend operation.

Verify:

- the application uses platform biometric APIs;
- sensitive keys require an appropriate biometric or device-authentication policy;
- enrollment changes are handled according to risk;
- fallback to device credentials is intentional;
- backend operations require valid server authorization;
- transaction approval is bound to the operation when needed;
- cancellation and failed authentication do not leak protected state.

The security question is not only whether Face ID or fingerprint UI appears. It is whether bypassing local UI would grant access to server data or operations.

## 12. Reverse Engineering and Hardcoded Secrets

Assume an attacker can obtain the APK or IPA. Analyze an authorized release build for:

- API endpoints and private routes;
- hardcoded credentials or encryption keys;
- debug flags and verbose logging;
- development certificates;
- feature flags and hidden administrative paths;
- Firebase or cloud configuration;
- exported Android components;
- URL schemes and universal links;
- iOS entitlements and capabilities;
- third-party SDKs and outdated dependencies.

Public client identifiers and endpoint URLs are not necessarily secrets. Private credentials, signing material, master encryption keys, and unrestricted service keys must never depend on application-package secrecy.

Obfuscation can increase analysis cost, but it does not make embedded secrets safe.

## 13. Root, Jailbreak, Debugging, and Runtime Tampering

Test on a dedicated rooted Android or jailbroken iPhone only when explicitly permitted. Determine whether runtime tooling can:

- attach to the process;
- modify client-side verification values;
- bypass local checks;
- inspect sensitive data in memory;
- invoke hidden application functions;
- alter network validation;
- trigger privileged UI paths.

Root and jailbreak detection are defense-in-depth controls, not a trustworthy authorization source. Detection can be bypassed, may create false positives, and should not be the only protection for sensitive data.

The decisive test is server behavior:

```text
client says: NID verified
server record: verification missing
expected server response: reject
```

## 14. Application Tampering

Using a dedicated test build, alter a non-authoritative client value such as:

```text
isPremium = false -> true
isKYCVerified = false -> true
isLivenessPassed = false -> true
```

Then attempt the corresponding server operation. The backend must reject the action unless its own trusted records confirm the state.

Critical server-owned decisions include:

- user and administrator roles;
- subscription or payment status;
- KYC and NID verification;
- liveness and face-match result;
- transaction approval;
- risk score and account restrictions.

Code signing, integrity checks, attestation, and anti-tampering can improve resilience, but they supplement rather than replace server authorization.

## 15. Deep Links and App-to-App Interaction

Enumerate custom URL schemes, universal links, Android App Links, exported activities, services, receivers, and content providers.

Test controlled links for:

- authentication bypass;
- untrusted identifiers;
- unauthorized navigation;
- parameter injection;
- open redirects;
- arbitrary WebView loading;
- leakage through callback URLs;
- unsafe handling by another installed app.

For example, a link such as:

```text
sampleapp://application?application_id=5002
```

must not reveal the object merely because the application opened the correct screen. The API still needs object-level authorization.

## 16. WebView Security

Review every WebView or embedded browser for:

- untrusted URL loading;
- unrestricted navigation;
- unnecessary JavaScript;
- exposed native JavaScript interfaces;
- local file access;
- mixed HTTP and HTTPS content;
- unsafe redirects;
- cookie and session leakage;
- weak allowlists;
- missing isolation between trusted and untrusted content.

Avoid designs where arbitrary user input is sent directly to `loadUrl`. Validate scheme, host, port, and path, and open external content in an appropriate system browser when possible.

## 17. File and Image Upload Security

For NID, document, and avatar uploads, test controlled inputs such as:

- valid JPEG, PNG, and permitted PDF files;
- an unsupported type renamed with an image extension;
- a zero-byte or corrupted file;
- oversized byte length;
- extreme pixel dimensions;
- malformed metadata;
- repeated uploads;
- the wrong document category;
- front submitted as back;
- the same image submitted for both sides.

The backend should validate:

- authenticated authorization to upload;
- actual decoded media type;
- safe file size and dimensions;
- successful decoding;
- expected document category and side;
- malware or active-content policy where applicable;
- storage key isolation;
- retention and deletion policy.

Filename extension and `Content-Type` headers are attacker-controlled hints, not proof of content.

## 18. NID and Identity-Document Spoofing

Create an approved test corpus with synthetic or properly protected data:

- genuine test NID front and back;
- photocopy;
- screenshot;
- image displayed on another phone;
- printed copy;
- edited number or date of birth;
- replaced portrait;
- mismatched front and back;
- passport or driving license;
- employee or university card;
- blank card and random image;
- AI-generated NID-like document.

Evaluate more than document classification. A strong eKYC flow can verify:

- correct document type and side;
- image quality and completeness;
- expected visual security features where technically feasible;
- OCR field consistency;
- checksum or format rules;
- front/back relationship;
- document-face and live-face relationship;
- duplicate and replay detection;
- backend source verification where legally and technically available.

Store test media securely and remove it after the assessment according to the agreed retention policy.

## 19. Face Liveness and Presentation Attack Detection

Presentation Attack Detection should be tested on real devices and realistic lighting conditions.

Use an authorized matrix such as:

| Presentation | Expected behavior |
| --- | --- |
| Printed face | Reject |
| Face photo on another screen | Reject |
| Recorded video | Reject |
| Replayed challenge sequence | Reject |
| Multiple faces | Reject |
| Partial or heavily occluded face | Reject or issue a safe challenge |
| Genuine live user | Accept within the target false-reject rate |

Also test masks, sunglasses, glare, low light, motion blur, slow-motion replay, looping video, deepfake video, and camera-injection resistance where the authorized environment supports it.

For active challenges, generate unpredictable server-owned challenges with short expiry and one-time use. A recorded `turn left` sequence should not satisfy a new `look up` challenge.

Do not rely on one signal. Combine device integrity, capture continuity, challenge response, texture or depth cues, replay indicators, face matching, document consistency, and server-side risk policy according to product requirements.

## 20. Input Validation and Robustness

Test permitted API fields with controlled edge cases:

```text
empty string
null
zero and negative numbers
very large values
long Unicode text
mixed writing directions
unexpected arrays or objects
malformed dates and phone numbers
unsupported media encodings
```

The goal is to verify parser safety, type enforcement, business validation, error handling, and authorization. The API should return stable, non-sensitive errors without stack traces or partial data changes.

## 21. Prioritization for NID/eKYC Applications

A practical order is:

1. API object-level and function-level authorization.
2. Session, refresh-token, logout, and replay behavior.
3. Authentication and OTP workflow enforcement.
4. NID front/back spoofing and upload validation.
5. Face photo, video, and random-challenge replay.
6. Local identity-image, token, and log leakage.
7. Screenshot, clipboard, and background-preview exposure.
8. Runtime modification of KYC and liveness state.
9. Certificate validation and network hardening.
10. Deep links, WebViews, exported components, and input robustness.

High severity should be driven by impact and exploitability, not by a tool's default score. An authorization failure exposing another person's identity data is normally more urgent than a bypassable root-detection warning.

## 22. Reporting Findings Clearly

Each finding should contain:

- concise title;
- affected app version and endpoint;
- security requirement;
- preconditions and test account role;
- reproducible authorized steps;
- expected and actual behavior;
- impact on users and business;
- evidence with secrets redacted;
- severity and reasoning;
- recommended remediation;
- retest result.

A useful finding title is:

```text
Broken object-level authorization exposes another user's NID verification record
```

This is clearer than:

```text
API issue
```

## Final Checklist

- Define written scope and use dedicated test identities.
- Map assets, roles, APIs, trust boundaries, and workflow states.
- Cover MASVS areas and product-specific threats.
- Test the server even when the client UI blocks an action.
- Treat the client as potentially compromised.
- Validate authorization on every object and function.
- Test token revocation, rotation, expiry, and replay.
- Inspect storage, logs, screenshots, clipboard, and temporary media.
- Validate decoded upload content, not filenames alone.
- Build NID spoofing and liveness presentation-attack datasets.
- Preserve source evidence without retaining unnecessary personal data.
- Retest every remediation in the same environment.

Mobile security is strongest when controls remain effective after the application package is inspected, client state is modified, and network requests are replayed. The server should own identity, authorization, verification status, and high-value decisions, while the mobile app minimizes sensitive data and uses platform security features correctly.

## References

- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [OWASP MASTG](https://mas.owasp.org/MASTG/)
- [OWASP MAS Checklist](https://mas.owasp.org/checklists/)
- [Android Keystore](https://developer.android.com/privacy-and-security/keystore)
- [Android Network Security Configuration](https://developer.android.com/privacy-and-security/security-config)
- [Apple Platform Security](https://support.apple.com/guide/security/welcome/web)
