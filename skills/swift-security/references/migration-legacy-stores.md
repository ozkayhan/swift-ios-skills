# Migration & Legacy Stores

> **Scope:** Migrating sensitive data from UserDefaults, plists, NSCoding archives, and other insecure storage to Apple Keychain Services. Covers secure deletion of legacy data, first-launch keychain cleanup, versioned migration patterns, and the Team ID transfer edge case.
>
> **Applies to:** iOS 15+ (actor support), iOS 17+ (recommended deployment target)
>
> **Cross-references:** `keychain-fundamentals.md` (SecItem CRUD), `keychain-access-control.md` (accessibility classes), `common-anti-patterns.md` (UserDefaults secrets anti-pattern), `credential-storage-patterns.md` (token lifecycle post-migration), `testing-security-code.md` (protocol-based mocking)

---

## Contents

- [Why Migrate — The Risk of Legacy Storage](#why-migrate-the-risk-of-legacy-storage)
- [The Five Correctness Traps](#the-five-correctness-traps)
- [First-Launch Keychain Cleanup](#first-launch-keychain-cleanup)
- [Verified Migration: Read → Write → Verify → Delete](#verified-migration-read-write-verify-delete)
- [Versioned Migration with Schema Tracking](#versioned-migration-with-schema-tracking)
- [Renaming a Keychain Service or Account](#renaming-a-keychain-service-or-account)
- [Background Launch and the Locked-Device Trap](#background-launch-and-the-locked-device-trap)
- [The Phantom Mismatch Bug](#the-phantom-mismatch-bug)
- [Team ID Change: The App Transfer Edge Case](#team-id-change-the-app-transfer-edge-case)
- [Deferred Legacy Cleanup with Rollback Window](#deferred-legacy-cleanup-with-rollback-window)
- [Complete App Launch Sequence](#complete-app-launch-sequence)
- [Thread Safety Note](#thread-safety-note)
- [Testing Migration Paths](#testing-migration-paths)
- [Handling Very Old Versions and Collapse Strategy](#handling-very-old-versions-and-collapse-strategy)
- [Secure Deletion: Trust Cryptographic Erasure](#secure-deletion-trust-cryptographic-erasure)
- [Conclusion](#conclusion)
- [Summary Checklist](#summary-checklist)

## Why Migrate — The Risk of Legacy Storage

UserDefaults, `.plist` files, and NSCoding archives store data as unencrypted plaintext within the app sandbox. This data is readable on jailbroken devices and included in unencrypted iTunes/Finder backups — anyone with backup access can extract tokens, passwords, and PII. OWASP ranks insecure data storage as a top-10 mobile risk (M9).

| Store             | Encrypted at rest | In backups                         | Survives app uninstall | Suitable for secrets |
| ----------------- | ----------------- | ---------------------------------- | ---------------------- | -------------------- |
| UserDefaults      | No                | Yes                                | No                     | **No**               |
| .plist files      | No (default)      | Yes                                | No                     | **No**               |
| NSCoding archives | No (default)      | Yes                                | No                     | **No**               |
| Keychain          | Yes (AES-256-GCM) | `ThisDeviceOnly` variants excluded | May persist                | **Yes**              |

Keychain items are managed by the `securityd` daemon, encrypted with per-row keys protected by the Secure Enclave, and isolated from the app sandbox. This is the only appropriate location for tokens, passwords, API keys, and PII on Apple platforms.

---

## The Five Correctness Traps

Most AI-generated migration code contains at least one of these errors. Each passes testing but fails catastrophically in production.

**Trap 1 — Legacy data survives after migration.** Calling `UserDefaults.standard.removeObject(forKey:)` removes the key-value pair from the in-memory cache and plist file, but does not securely overwrite NAND flash. However, iOS achieves secure deletion through _cryptographic erasure_: every file has a per-file AES-256 key, and standard deletion APIs destroy that key via Effaceable Storage, rendering physical bits permanently inaccessible. The real risk vector is **unencrypted backups** created before migration completes — the plist stays on disk until the filesystem reclaims space. **Always delete all legacy keys explicitly after verified keychain writes.**

**Trap 2 — Keychain items may survive app deletion.** On reinstall, validate retained credentials before restoring a session. Also handle the case where the item is missing.

**Trap 3 — A migration treats unavailable data as absent.** On a locked-device launch, defer migration when protected data is unavailable. Treat a read error differently from a confirmed missing value.

**Trap 4 — Deletion before verification can lose data.** A crash after a successful keychain write can leave both copies. Verify the destination before deleting the legacy value, and make retries safe when both copies exist.

**Trap 5 — A new lookup misses items saved under the old service or account.** Migrate existing entries before switching lookups. Check for duplicate destination items before deciding how to resolve conflicts.

---

## First-Launch Keychain Cleanup

Keychain items may outlive the app’s sandbox. An absent local marker does not prove a reinstall, especially when introducing the marker to an existing app. Validate the account/session before restoring credentials. If product policy requires per-install cleanup, distinguish existing installations before enabling it and scope deletion to the specific app-private credential. See [Apple Developer Technical Support](https://developer.apple.com/forums/thread/36442).

For a confirmed fresh install under that policy, run this operation before
restoring the local session. Defer initialization if protected data is
unavailable, and mark cleanup complete only after a successful deletion or a
confirmed absence.

```swift
import Foundation
import Security

func removeLocalCredential(service: String, account: String, appPrivateGroup: String) -> OSStatus {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: service,
        kSecAttrAccount as String: account,
        kSecAttrAccessGroup as String: appPrivateGroup,
        kSecAttrSynchronizable as String: false
    ]
    return SecItemDelete(query as CFDictionary)
}
```

Run the blocking keychain call away from the main thread. Treat only
`errSecSuccess` and `errSecItemNotFound` as completion. Avoid a class-wide
query or `kSecAttrSynchronizableAny` for app-local cleanup.

---

## Verified Migration: Read → Write → Verify → Delete

The most dangerous pattern is deleting legacy data before confirming the keychain write succeeded. The correct sequence is always: **read → write → verify → delete**.

```swift
// Verify each destination before deleting its legacy value
actor VerifiedMigrator {
    struct MigrationResult {
        let key: String
        let succeeded: Bool
        let error: Error?
    }

    private let keychain: any MigrationKeychainProtocol

    init(keychain: any MigrationKeychainProtocol) {
        self.keychain = keychain
    }

    /// Failed keys remain in UserDefaults for retry on next launch.
    func migrateUserDefaultsKeys(
        _ keys: [String],
        service: String,
        accessible: CFString = kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
    ) async -> [MigrationResult] {
        var results: [MigrationResult] = []

        for key in keys {
            do {
                // STEP 1: Read from legacy storage
                guard let legacyValue = UserDefaults.standard.string(forKey: key),
                      let data = legacyValue.data(using: .utf8) else {
                    results.append(.init(key: key, succeeded: true, error: nil))
                    continue
                }

                // STEP 2: Write to keychain (add-or-update handles duplicates)
                try await keychain.save(data, service: service,
                                        account: key, accessible: accessible)

                // STEP 3: Verify by reading back
                let readBack = try await keychain.read(service: service, account: key)
                guard readBack == data else {
                    throw MigrationError.verificationFailed(key: key)
                }

                // STEP 4: Delete from UserDefaults ONLY after verified write
                UserDefaults.standard.removeObject(forKey: key)
                results.append(.init(key: key, succeeded: true, error: nil))

            } catch {
                // Keep UserDefaults intact for retry
                results.append(.init(key: key, succeeded: false, error: error))
            }
        }
        return results
    }

    enum MigrationError: Error {
        case verificationFailed(key: String)
        case corruptArchive(path: String)
    }
}
```

```swift
// ❌ INCORRECT: Deletes legacy data BEFORE verifying keychain write
func dangerousMigration() {
    let keys = ["authToken", "refreshToken"]
    for key in keys {
        guard let value = UserDefaults.standard.string(forKey: key) else { continue }

        // Deletes FIRST — if keychain write fails, data is gone forever
        UserDefaults.standard.removeObject(forKey: key) // ← CATASTROPHIC

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: "com.myapp",
            kSecAttrAccount as String: key,
            kSecValueData as String: value.data(using: .utf8)!
        ]
        let status = SecItemAdd(query as CFDictionary, nil)
        // If status != errSecSuccess, the token is permanently lost.
    }
}
```

Run this migration before normal credential writes. A retry skips keys whose legacy value has already been deleted. When a crash leaves both copies, verify the destination and finish cleanup before resuming normal writes.

---

## Versioned Migration with Schema Tracking

A production system needs version tracking to avoid re-running completed migrations and to handle users who skip versions. Store the schema version alongside the credentials it tracks. Handle a missing version when no keychain data remains.

```swift
// ✅ CORRECT: Versioned chain migration with schema version in keychain
actor MigrationCoordinator {
    static let shared = MigrationCoordinator()

    private let serviceName = "com.myapp.credentials"
    private let schemaVersionAccount = "com.myapp.schema.version"
    private static let currentSchemaVersion: Int = 3

    enum MigrationState {
        case upToDate
        case migrated(from: Int, to: Int)
        case deferred(reason: String)
        case failed(Error)
    }

    func migrateIfNeeded() async -> MigrationState {
        // Guard: protected data must be available (locked-device launch)
        let dataAvailable = await MainActor.run {
            UIApplication.shared.isProtectedDataAvailable
        }
        guard dataAvailable else {
            return .deferred(reason: "Device locked — protected data unavailable")
        }

        let storedVersion = readSchemaVersion()
        guard storedVersion < Self.currentSchemaVersion else { return .upToDate }

        do {
            // Chain migration: each step runs sequentially
            if storedVersion < 1 {
                try await migrateV0toV1_UserDefaultsToKeychain()
            }
            if storedVersion < 2 {
                try await migrateV1toV2_NSCodingArchivesToKeychain()
            }
            if storedVersion < 3 {
                try await migrateV2toV3_UpgradeAccessibilityClass()
            }

            // Update version ONLY after all steps succeed
            try saveSchemaVersion(Self.currentSchemaVersion)
            return .migrated(from: storedVersion, to: Self.currentSchemaVersion)
        } catch {
            // Do NOT update schema version — retry on next launch
            os_log(.error, log: .migration,
                   "Migration failed: %{public}@", error.localizedDescription)
            return .failed(error)
        }
    }

    // MARK: - Schema Version (stored alongside the keychain data)

    private func readSchemaVersion() -> Int {
        guard let data = try? keychainRead(
                  service: serviceName, account: schemaVersionAccount),
              let str = String(data: data, encoding: .utf8),
              let version = Int(str) else { return 0 }
        return version
    }

    private func saveSchemaVersion(_ version: Int) throws {
        let data = "\(version)".data(using: .utf8)!
        try keychainSave(data, service: serviceName,
                         account: schemaVersionAccount)
    }

    // MARK: - V1: UserDefaults → Keychain

    private func migrateV0toV1_UserDefaultsToKeychain() async throws {
        let migrator = VerifiedMigrator(keychain: KeychainManager.shared)
        let results = await migrator.migrateUserDefaultsKeys(
            ["authToken", "refreshToken", "apiSecret"],
            service: serviceName
        )
        // Check for critical failures (non-nil keys that didn't migrate)
        let failures = results.filter { !$0.succeeded }
        if !failures.isEmpty {
            os_log(.error, log: .migration,
                   "V1 migration: %d keys failed", failures.count)
        }
        // Force-sync UserDefaults deletions to disk
        UserDefaults.standard.synchronize()
    }

    // MARK: - V2: NSCoding Archives → Keychain

    private func migrateV1toV2_NSCodingArchivesToKeychain() async throws {
        let documentsURL = FileManager.default.urls(
            for: .documentDirectory, in: .userDomainMask).first!
        let archiveURL = documentsURL.appendingPathComponent("UserSession.archive")

        guard FileManager.default.fileExists(atPath: archiveURL.path) else { return }

        let archiveData = try Data(contentsOf: archiveURL)
        guard let session = try NSKeyedUnarchiver.unarchivedObject(
            ofClass: LegacySession.self, from: archiveData) else {
            throw VerifiedMigrator.MigrationError.corruptArchive(path: archiveURL.path)
        }

        let sessionData = try JSONEncoder().encode(session.toModernSession())
        try keychainSave(sessionData, service: serviceName, account: "userSession")

        // Verify before deleting archive file
        let verified = try keychainRead(service: serviceName, account: "userSession")
        guard verified == sessionData else {
            throw VerifiedMigrator.MigrationError.verificationFailed(key: "userSession")
        }
        try FileManager.default.removeItem(at: archiveURL)
    }

    // MARK: - V3: Upgrade accessibility class on existing items

    private func migrateV2toV3_UpgradeAccessibilityClass() async throws {
        let accounts = ["authToken", "refreshToken", "apiSecret", "userSession"]
        for account in accounts {
            guard let data = try? keychainRead(
                      service: serviceName, account: account) else { continue }
            // Re-save with updated accessibility — add-or-update pattern
            // updates the accessibility class via SecItemUpdate
            try keychainSave(data, service: serviceName, account: account,
                             accessible: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly)
        }
    }
}

private extension OSLog {
    static let migration = OSLog(
        subsystem: Bundle.main.bundleIdentifier ?? "com.myapp",
        category: "KeychainMigration"
    )
}
```

```swift
// ❌ INCORRECT: Runs every launch, no version check, no verification, no legacy delete
func brokenMigration() {
    // No version check — runs every single launch
    // No isProtectedDataAvailable check — may fail during locked-device launch
    if let token = UserDefaults.standard.string(forKey: "authToken") {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: "com.myapp",
            kSecAttrAccount as String: "authToken",
            kSecValueData as String: token.data(using: .utf8)!
        ]
        // No errSecDuplicateItem handling — crashes on second launch
        SecItemAdd(query as CFDictionary, nil)
        // Never deletes from UserDefaults — plaintext secret persists
        // No verification that write succeeded
    }
}
```

The chain migration approach (v1 → v2 → v3 sequentially) is deliberately chosen over direct migration because it reuses tested migration logic from each version. For users upgrading from v1.0 directly to v3.0, all three steps run. The schema version only advances after all steps succeed — a crash mid-migration leaves the version at the old number for clean retry.

---

## Renaming a Keychain Service or Account

Changing only the attributes used for lookups makes older items unreachable
through the new lookup. For an in-place attribute change, use a narrowly
scoped `SecItemUpdate`, supplying the old attributes in the query and the new
attributes in the update dictionary. Check the returned status. See [Updating and deleting keychain items](https://sosumi.ai/documentation/security/updating-and-deleting-keychain-items).

When intentionally keeping a separate copy for a migration window, use a
read, write, verify, and delete sequence:

```swift
// Copy-based migration when a separate destination item is needed
func migrateServiceName() async throws {
    let oldService = "OldServiceName"
    let newService = "com.mycompany.myapp"
    let accounts = ["authToken", "refreshToken"]

    for account in accounts {
        let oldData: Data
        do {
            oldData = try keychainRead(service: oldService, account: account)
        } catch KeychainError.itemNotFound {
            continue // Already migrated or never existed
        }

        try keychainSave(oldData, service: newService, account: account)

        // Verify new location before deleting old
        let verified = try keychainRead(service: newService, account: account)
        guard verified == oldData else {
            throw VerifiedMigrator.MigrationError.verificationFailed(key: account)
        }
        try keychainDelete(service: oldService, account: account)
    }
}
```

Choose a stable `kSecAttrService`, such as the app’s bundle identifier, and plan a migration before changing it.

---

## Background Launch and the Locked-Device Trap

Background execution can launch your app while the device is locked. The `kSecAttrAccessible` value you choose determines whether keychain operations succeed in these contexts.

> For the complete accessibility constant selection matrix with data protection tiers and security trade-offs, see `keychain-access-control.md` § The "When" Layer: Seven Accessibility Constants. The table below summarizes the four constants most relevant to background migration scenarios.

| Accessibility constant                             | Available when locked | Background safe | Notes                                                |
| -------------------------------------------------- | --------------------- | --------------- | ---------------------------------------------------- |
| `kSecAttrAccessibleWhenUnlocked` (default)         | No                    | No              | Foreground only                                      |
| `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | After first unlock    | Yes             | **Recommended** — background + device-bound          |
| `kSecAttrAccessibleAfterFirstUnlock`               | After first unlock    | Yes             | Background + backup migration (use only when needed) |
| `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`  | No                    | No              | Biometric-gated items                                |
| `kSecAttrAccessibleAlways`                         | Yes                   | Yes             | **Deprecated iOS 12** — do not use                   |

**Recommended default for migrated credentials:** `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` — background-safe, not synced to iCloud, not included in backups. Apple uses `AfterFirstUnlock` for Wi-Fi passwords and mail account credentials.

A critical trap: **`SecItemDelete` does NOT require the item's protection-class key material** — it succeeds even when the item's data is unreadable due to lock state. This enables a devastating anti-pattern:

```swift
// ❌ DANGEROUS: Delete-on-read-failure destroys data during background launch
func dangerousTokenRefresh() {
    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)

    if status != errSecSuccess {
        // "Can't read? Must be corrupted. Delete and start fresh."
        SecItemDelete(query as CFDictionary) // ← DESTROYS VALID TOKEN
        // During background launch with WhenUnlocked, the read fails
        // with -25308 (interaction not allowed), but delete succeeds.
    }
}

// ✅ CORRECT: Distinguish "not found" from "device locked"
func safeTokenRead() throws -> Data? {
    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)

    switch status {
    case errSecSuccess:
        return result as? Data
    case errSecItemNotFound:
        return nil // Genuinely absent
    case errSecInteractionNotAllowed:
        // Device locked — item exists but unreadable right now.
        // Do NOT delete. Do NOT treat as missing. Retry later.
        throw KeychainError.interactionNotAllowed
    default:
        throw KeychainError.unexpectedStatus(status)
    }
}
```

**Migration rule:** Always guard migration behind `UIApplication.shared.isProtectedDataAvailable`. If the device is locked, defer using `protectedDataDidBecomeAvailableNotification`. Never interpret an empty read during a locked state as "nothing to migrate."

---

## The Phantom Mismatch Bug

Including `kSecAttrAccessible` in a search query causes a "not-found then duplicate" paradox. The search filters by accessibility class, but the item was stored with a different class — so `SecItemCopyMatching` returns `errSecItemNotFound` while `SecItemAdd` sees the item via primary key and returns `errSecDuplicateItem`.

```swift
// ❌ INCORRECT: kSecAttrAccessible in search query causes phantom mismatches
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrService as String: service,
    kSecAttrAccount as String: account,
    kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlocked, // ← BUG
    kSecReturnData as String: kCFBooleanTrue as Any
]
// If stored with AfterFirstUnlock, query returns errSecItemNotFound.
// But SecItemAdd sees the item via primary key → errSecDuplicateItem. Deadlock.
```

**Rule:** Use **only primary key attributes** (`kSecClass`, `kSecAttrService`, `kSecAttrAccount`) in search queries. Set `kSecAttrAccessible` only during `SecItemAdd` or in the update dictionary of `SecItemUpdate`.

```swift
// ✅ CORRECT: search by primary key only
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrService as String: service,
    kSecAttrAccount as String: account,
    kSecReturnData as String: kCFBooleanTrue as Any
]
```

---

## Team ID Change: The App Transfer Edge Case

When an app is transferred to a different Apple Developer account, the Team ID changes. Keychain access is permanently tied to the original Team ID — all existing keychain items become inaccessible under the new signing identity. Users are effectively logged out and lose all locally stored secrets on the first launch after updating.

**If a Team ID change is unavoidable**, you must release a "bridge" update under the **old** Team ID before the transfer:

1. Bridge update reads all keychain items and exports them to a temporary, app-group-shared container (or encrypted file in the app sandbox)
2. Transfer the app to the new developer account
3. First release under the new Team ID reads from the temporary store, writes to the new keychain, verifies, and deletes the temporary data

This is a one-way operation and must be planned well in advance. There is no way to recover keychain items after a Team ID change without the bridge update.

---

## Deferred Legacy Cleanup with Rollback Window

The safest approach keeps legacy data as backup for one release cycle after migration. Track a migration timestamp in keychain:

```swift
// ✅ CORRECT: Deferred cleanup with 30-day rollback window
actor DeferredCleanup {
    private let cleanupDelayDays = 30
    private let timestampAccount = "com.myapp.migration.timestamp"
    private let serviceName = "com.myapp.credentials"

    func cleanupIfExpired() async {
        guard let data = try? keychainRead(
                  service: serviceName, account: timestampAccount),
              let str = String(data: data, encoding: .utf8),
              let migrationDate = ISO8601DateFormatter().date(from: str) else { return }

        let days = Calendar.current.dateComponents(
            [.day], from: migrationDate, to: Date()).day ?? 0
        guard days >= cleanupDelayDays else { return }

        // Past rollback window — safe to permanently delete legacy files
        let documentsURL = FileManager.default.urls(
            for: .documentDirectory, in: .userDomainMask).first!
        for file in ["UserSession.archive", "Credentials.plist", "TokenCache.dat"] {
            try? FileManager.default.removeItem(
                at: documentsURL.appendingPathComponent(file))
        }
        if let bundleID = Bundle.main.bundleIdentifier {
            UserDefaults.standard.removePersistentDomain(forName: bundleID)
        }
    }
}
```

---

## Complete App Launch Sequence

The correct ordering at app startup is critical. Keychain cleanup must happen before SDK initialization, migration must wait for protected data, and schema version gates all logic.

```swift
// ✅ CORRECT: Complete launch sequence with migration
@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var delegate
    var body: some Scene { WindowGroup { ContentView() } }
}

class AppDelegate: NSObject, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        Task {
            // 1. First-launch cleanup (stale keychain from previous install)
            await FirstLaunchGuard.shared.performCleanupIfNeeded()

            // 2. Versioned migration
            let state = await MigrationCoordinator.shared.migrateIfNeeded()
            switch state {
            case .upToDate: break
            case .migrated(let from, let to):
                os_log(.info, "Migrated schema v%d → v%d", from, to)
            case .deferred(let reason):
                os_log(.info, "Migration deferred: %{public}@", reason)
            case .failed(let error):
                os_log(.error, "Migration failed: %{public}@",
                       error.localizedDescription)
            }

            // 3. Deferred cleanup of legacy files past rollback window
            await DeferredCleanup().cleanupIfExpired()

            // 4. NOW initialize Firebase, analytics, auth SDKs
            // Stale data cleared, migration complete or safely deferred
        }
        return true
    }
}
```

---

## Thread Safety Note

`SecItem*` functions are safe to call from concurrent code, but your wrapper's mutable state (caches, migration flags, version tracking) still needs synchronization. An `actor` provides this naturally in modern Swift concurrency; prefer actors over serial queues for new code when the deployment target allows it.

---

## Testing Migration Paths

Keychain behavior differs between Simulator and real devices:

| Aspect                        | Simulator                | Real device                            |
| ----------------------------- | ------------------------ | -------------------------------------- |
| Data Protection enforcement   | Not enforced             | Fully enforced (hardware)              |
| Keychain entitlements         | Loosely enforced         | Strictly enforced                      |
| `errSecInteractionNotAllowed` | Rarely triggered         | Triggered when locked                  |
| Lock state testing            | Cannot meaningfully test | Essential for accessibility validation |

Use **protocol-based abstraction** for unit tests (runs in CI on simulators) and real-device integration tests for accessibility-class validation:

```swift
// ✅ Protocol-based keychain abstraction for testable migrations
protocol MigrationKeychainProtocol: Actor {
    func save(_ data: Data, service: String, account: String,
              accessible: CFString) throws
    func read(service: String, account: String) throws -> Data
    func delete(service: String, account: String) throws
    func deleteAll()
}

// In-memory mock for unit tests
actor MockMigrationKeychain: MigrationKeychainProtocol {
    var store: [String: [String: Data]] = [:]
    var simulatedError: KeychainError?

    func save(_ data: Data, service: String, account: String,
              accessible: CFString) throws {
        if let error = simulatedError { throw error }
        store[service, default: [:]][account] = data
    }

    func read(service: String, account: String) throws -> Data {
        if let error = simulatedError { throw error }
        guard let data = store[service]?[account] else {
            throw KeychainError.itemNotFound
        }
        return data
    }

    func delete(service: String, account: String) throws {
        store[service]?[account] = nil
    }

    func deleteAll() { store.removeAll() }
}
```

```swift
// Verify that a failed write leaves the legacy value available for retry
@Test func migrationPreservesLegacyDataOnKeychainFailure() async {
    let mock = MockMigrationKeychain()
    mock.simulatedError = .unexpectedStatus(-25308) // Simulate locked device

    let defaults = UserDefaults(suiteName: "test")!
    defaults.set("secret-token", forKey: "authToken")

    let migrator = VerifiedMigrator(keychain: mock)
    let results = await migrator.migrateUserDefaultsKeys(
        ["authToken"], service: "com.myapp"
    )

    #expect(results.contains(where: { !$0.succeeded }))
    #expect(defaults.string(forKey: "authToken") == "secret-token") // Still intact
}
```

Always clean up keychain items in `setUp()`/`tearDown()` — items persist between test runs on the same simulator. For integration tests hitting real keychain, create a Test Host app target with the Keychain capability enabled.

---

## Handling Very Old Versions and Collapse Strategy

The App Store always delivers the latest binary — a user jumping from v1.0 to v3.0 never installs v2.0. Your v3.0 binary must contain migration logic for every historical schema version.

Pragmatically, after sufficient time (when analytics show <1% of users on legacy versions), **collapse old migrations into a single mega-migration** from v0 to current, reducing code maintenance. For users on versions so old that the legacy format is unknown or corrupted, the migration should **fail gracefully** and prompt a fresh login rather than crashing.

---

## Secure Deletion: Trust Cryptographic Erasure

Do **not** attempt to manually overwrite files with zeros or random bytes before deletion — NAND flash wear-leveling makes this ineffective and wastes write cycles. iOS handles secure deletion through cryptographic erasure: every file has a per-file AES-256 key, and when the file is deleted via standard APIs (`FileManager.removeItem`, `UserDefaults.removeObject`), iOS destroys the per-file key through Effaceable Storage, rendering the physical bits permanently unrecoverable.

Standard deletion APIs are sufficient. The residual risk is unencrypted backups created _before_ migration — encourage users to use encrypted backups, and delete legacy data promptly after verified migration.

---

## Conclusion

Verify before deleting, defer migration when protected data is unavailable, and handle both retained and missing credentials after a reinstall.

---

## Summary Checklist

1. **Per-install cleanup is scoped** — run before session restore only if policy requires it. Match the app-private, non-synchronizable item and check the status.
2. **Verify before deleting** — read back the keychain write, then delete the legacy value. A failed write leaves the original available for retry.
3. **Schema version tracks keychain data** — advances after successful migration and handles an absent version.
4. **Protected data availability checked before any migration** — guards against locked-device scenarios; defers via `protectedDataDidBecomeAvailableNotification`
5. **`errSecInteractionNotAllowed` (-25308) is never treated as "item missing"** — distinguishes locked-device failures from genuine absence; never deletes on read failure without checking status code
6. **Service and account changes are deliberate migrations** — select the old keychain item before updating its attributes or copying it to the new location.
7. **`kSecAttrAccessible` is never included in search queries** — causes phantom "not-found then duplicate" mismatches; set only during add or in update dictionary
8. **Default accessibility is `AfterFirstUnlockThisDeviceOnly`** — background-safe, not synced, not backed up; matches Apple's own credential storage patterns
9. **Deferred legacy cleanup with rollback window** — keep legacy data for 30 days post-migration as safety net; timestamp stored in keychain
10. **Team ID changes sever all keychain access** — must release bridge update under old Team ID before app transfer; no recovery possible after transfer without bridge
11. **Migration tested via protocol-based abstraction** — mock keychain in unit tests; real-device integration tests for accessibility class validation; clean up items in setUp/tearDown
