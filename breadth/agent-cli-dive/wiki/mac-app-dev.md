# Mac App Development

**Building Mac apps with an AI agent is viable, but five specific failure modes — headless Keychain access, .pbxproj corruption, sandbox Apple Events blocking, notarytool credential routing, and quarantine removal — will each stop a build cold if not addressed up front.**

## Code Signing

[Ad-hoc signing](https://karol-mazurek.medium.com/snake-apple-ii-code-signing-f0a9967b7f02) (`codesign --sign -`) works in any agent context with no certificates or Keychain access. It produces a locally runnable but non-distributable binary that Gatekeeper blocks for any other user. Use it for build verification; never for distribution.

Developer ID signing in a headless context fails unless a writable, unlocked keychain holds the certificate and its private key. The CI/agent pattern:

```bash
security create-keychain -p "$PASSWORD" build.keychain
security unlock-keychain -p "$PASSWORD" build.keychain
security import cert.p12 -k build.keychain -T /usr/bin/codesign
security set-key-partition-list -S apple-tool:,apple: -k "$PASSWORD" build.keychain
```

Without `set-key-partition-list`, codesign triggers an interactive approval dialog that hangs headless processes. The canonical failure signature is [`errSecInternalComponent`](https://vpsmac.com/en/blog/mac-cloud-ios-ci-signing-keychain-headless-xcodebuild-2026.html) — the same error for a locked keychain, a missing partition list, a non-GUI SSH session, or an entitlement/provisioning profile mismatch. The error is identical in all cases, making diagnosis hard.

The login keychain is [only automatically unlocked in a GUI session](https://github.com/fastlane/fastlane/issues/19369). Any non-GUI invocation — cron, launchd daemon, SSH — does not inherit it. The fix: create a named custom keychain, add it to the search list with `security list-keychains -d user -s`, and use it as the target for all imports and signing operations.

`xcodebuild` fails with "Signing for App requires a development team" when `DEVELOPMENT_TEAM` is not set. [Pass it on the CLI](https://techsive.com/how-xcode-crashed-with-code-signing-error-signing-for-app-requires-a-development-team-and-the-detailed-fix-that-restored-ci-builds/): `DEVELOPMENT_TEAM=... CODE_SIGN_IDENTITY=... PROVISIONING_PROFILE_SPECIFIER=...`. For test-only builds, pass `CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO CODE_SIGN_IDENTITY=''` to skip signing entirely.

## Notarization

[`altool` was hard-deprecated November 1, 2023](https://developer.apple.com/documentation/technotes/tn3147-migrating-to-the-latest-notarization-tool); submissions are rejected. Use `xcrun notarytool`. Three credential modes:

- `--apple-id` + `--password` (app-specific) + `--team-id` — works, requires no separate setup
- `--keychain-profile` — requires a prior `store-credentials` run against an accessible keychain
- `--key` + `--key-id` + `--issuer` (App Store Connect API key) — **cleanest for agents**: no Keychain involvement at all

[Personal API keys are explicitly ineligible for the Notary API](https://developer.apple.com/forums/thread/749215). Use a Team API key with Developer permission. The `.p8` file can be passed directly via `--key` or placed at `~/private_keys/AuthKey_<KeyID>.p8`.

Hardened Runtime must be enabled during signing for notarization to succeed: pass `--options runtime` to `codesign`. Archive the `.app` as a zip before submitting — `ditto -c -k --keepParent App.app App.zip` — [submitting a raw directory fails](https://federicoterzi.com/blog/automatic-code-signing-and-notarization-for-macos-apps-using-github-actions/). After successful notarization, run `xcrun stapler staple` to attach the ticket for offline Gatekeeper validation.

## Claude Code Sandbox Constraints

Claude Code's [sandbox-exec (Seatbelt) layer blocks Apple Events by default](https://code.claude.com/docs/en/sandboxing). `osascript`, the `open` command, and browser-based OAuth flows (e.g., App Store Connect auth) fail with error -600. Two mitigations:

- Add the affected command to `excludedCommands` in settings — targeted, preserves sandbox for everything else
- Set `allowAppleEvents: true` — removes code-execution isolation entirely since sandboxed commands can then launch other apps unsandboxed

Go-based CLIs (`gh`, `gcloud`, `terraform`) fail TLS verification under Seatbelt due to `x509: OSStatus -26276`; add them to `excludedCommands`. `sandbox-exec` is marked deprecated in the macOS man page, adding long-term uncertainty for tooling built on it.

## .pbxproj Corruption

[`.pbxproj` corruption is a consistent and severe failure mode](https://blakecrosley.com/guides/ios-agent-development). The file uses a non-standard format with UUID cross-references requiring consistent updates across 3–5 sections. Agents report successful edits while the project becomes unreadable by Xcode. The documented mitigation: a `PreToolUse` hook blocking all writes to `.pbxproj`, `.xcworkspace/`, and `.xcodeproj/`. Recovery requires `git checkout` of the corrupted file.

Apply the same treatment to Interface Builder files. `.storyboard` and `.xib` are XML with auto-generated UUID constraint references; agents cannot meaningfully edit them. Use SwiftUI exclusively for new views; leave Interface Builder files untouched.

## Other Prerequisites

- Accept the Xcode license before any `xcodebuild` call in a fresh environment: `sudo xcodebuild -license accept`. Failure produces a license error that looks nothing like a build error.
- [Provisioning profiles](https://www.testdevlab.com/blog/xcode-provisioning-profile-automation-for-ci) go in `~/Library/Developer/Xcode/UserData/Provisioning Profiles/` (Xcode 16+) or `~/Library/MobileDevice/Provisioning Profiles/`. A `cp` command is sufficient; no GUI required.
- `xattr -d com.apple.quarantine` needs no `sudo` for agent-built binaries in the project directory. For artifacts placed in `/Applications`, `sudo` is required.

---
*See also: [Shell Execution Model](shell-execution-model.md), [Claude Code Permission and Trust Model](permission-trust-model.md), [File System Access](file-system-access.md)*
