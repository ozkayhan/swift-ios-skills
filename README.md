# Swift iOS Skills

iOS 26 ve sonrası için Swift, SwiftUI ve Apple framework’lerini kapsayan 86 Agent Skill. Kütüphane Swift 6.4.2, Xcode 27.2 ve iOS 27.2 ile gözden geçirildi. Bazı yeni API’ler iOS 27 gerektirir; ilgili becerideki sürüm koşullarını kontrol edin.

Bu depo [dpearson2699/swift-ios-skills](https://github.com/dpearson2699/swift-ios-skills) projesinin fork’udur.

## Projeye kurulum

Terminalde becerileri kullanacağınız projenin kök dizinine geçin. Node.js ve npm kurulu olmalıdır.

Codex için tüm becerileri o projeye kurun:

```sh
npx skills add ozkayhan/swift-ios-skills --skill '*' --agent codex --yes
```

Kurulum proje kapsamındadır; `-g` eklenmedikçe genel kullanıcı kurulumunu değiştirmez. Codex proje becerileri `.agents/skills/` altında bulunur.

Yalnızca belirli becerileri kurmak için:

```sh
npx skills add ozkayhan/swift-ios-skills --skill swiftui-patterns swift-concurrency --agent codex --yes
```

Becerileri veya kullanılacak asistanı etkileşimli seçmek için:

```sh
npx skills add ozkayhan/swift-ios-skills
```

Kurulumu kontrol etmek ve proje becerilerini güncellemek için:

```sh
npx skills list --agent codex
npx skills update --project
```

## Skills

### SwiftUI

| Skill | What it covers |
|-------|---------------|
| [focus-engine](skills/focus-engine/) | @FocusState, defaultFocus, focusSection, focused scene values, focus restoration, UIFocusGuide |
| [swiftui-animation](skills/swiftui-animation/) | Spring animations, PhaseAnimator, KeyframeAnimator, matchedGeometryEffect, SF Symbols |
| [swiftui-gestures](skills/swiftui-gestures/) | Tap, drag, magnify, rotate, long press, simultaneous and sequential gestures |
| [swiftui-layout-components](skills/swiftui-layout-components/) | Grid, LazyVGrid, Layout protocol, ViewThatFits, custom layouts |
| [swiftui-liquid-glass](skills/swiftui-liquid-glass/) | iOS 26 Liquid Glass, glassEffect, GlassEffectContainer, morphing transitions |
| [swiftui-navigation](skills/swiftui-navigation/) | NavigationStack, NavigationSplitView, programmatic navigation, deep linking |
| [swiftui-patterns](skills/swiftui-patterns/) | @Observable, state ownership, environment wiring, view composition, async loading, MV-pattern architecture |
| [swiftui-performance](skills/swiftui-performance/) | Rendering performance, view update optimization, layout thrash, Instruments profiling |
| [swiftui-uikit-interop](skills/swiftui-uikit-interop/) | UIViewRepresentable, UIHostingController, Coordinator, incremental UIKit-to-SwiftUI migration |
| [swiftui-webkit](skills/swiftui-webkit/) | WebView, WebPage, navigation policies, JavaScript calls, local content, custom URL schemes |

### Core Swift

| Skill | What it covers |
|-------|---------------|
| [swift-api-design-guidelines](skills/swift-api-design-guidelines/) | Swift API Design Guidelines -- argument labels, mutating/nonmutating pairs, documentation comments, naming conventions |
| [swift-architecture](skills/swift-architecture/) | Architecture patterns: MV (@Observable), MVVM, MVI, TCA, Clean Architecture, Coordinator, decision framework |
| [swift-codable](skills/swift-codable/) | Swift Codable, JSONDecoder, JSONEncoder, CodingKeys, custom decoding, nested JSON |
| [swift-charts](skills/swift-charts/) | Bar, line, area, pie, donut, and 3D charts, scrolling, selection, annotations |
| [swift-concurrency](skills/swift-concurrency/) | Swift 6.4 concurrency, Sendable, actors, structured concurrency, data-race safety |
| [swift-formatstyle](skills/swift-formatstyle/) | FormatStyle protocol, number/currency/date/duration/measurement formatting, custom styles |
| [swift-language](skills/swift-language/) | Swift 6.4 language idioms, result builders, property wrappers, typed throws |
| [swift-testing](skills/swift-testing/) | Swift Testing framework, @Test, @Suite, #expect, parameterized tests, mocking |
| [core-data](skills/core-data/) | Core Data persistence, NSPersistentContainer, NSFetchedResultsController, batch operations, staged migration |
| [swiftdata](skills/swiftdata/) | @Model, @Query, #Predicate, ModelContainer, migrations, CloudKit sync, @ModelActor |

### App Experience Frameworks

| Skill | What it covers |
|-------|---------------|
| [activitykit](skills/activitykit/) | ActivityKit, Dynamic Island, Lock Screen Live Activities, push-to-update |
| [adattributionkit](skills/adattributionkit/) | Privacy-preserving ad attribution, postbacks, conversion values, re-engagement |
| [alarmkit](skills/alarmkit/) | AlarmKit system alarms and countdown timers, Lock Screen, Dynamic Island, Live Activities |
| [app-clips](skills/app-clips/) | App Clips, invocation URLs, NFC, QR, App Clip Codes, App Group handoff |
| [app-intents](skills/app-intents/) | App Intents for Siri, Shortcuts, Spotlight, widgets, and Apple Intelligence |
| [avkit](skills/avkit/) | AVPlayerViewController, VideoPlayer, Picture-in-Picture, AirPlay, subtitles |
| [carplay](skills/carplay/) | CarPlay templates, navigation, audio, communication, EV charging apps |
| [mapkit](skills/mapkit/) | MapKit, CoreLocation, annotations, geocoding, directions, geofencing |
| [paperkit](skills/paperkit/) | PaperMarkupViewController, markup editing, drawing, shapes (iOS 26) |
| [pdfkit](skills/pdfkit/) | PDFView, PDFDocument, annotations, text search, form filling, thumbnails |
| [photokit](skills/photokit/) | PhotosPicker, AVCaptureSession, photo library, video recording, media permissions |
| [push-notifications](skills/push-notifications/) | UNUserNotificationCenter, APNs, rich notifications, silent push, service extensions |
| [storekit](skills/storekit/) | StoreKit 2 purchases, subscriptions, SubscriptionStoreView, transaction verification |
| [tipkit](skills/tipkit/) | Feature discovery tooltips, contextual tips, tip rules, tip events |
| [widgetkit](skills/widgetkit/) | Home Screen, Lock Screen, and StandBy widgets, Control Center controls, timeline providers |

### Data & Service Frameworks

| Skill | What it covers |
|-------|---------------|
| [cloudkit](skills/cloudkit/) | CKContainer, CKRecord, subscriptions, sharing, CKSyncEngine, SwiftData sync |
| [contacts-framework](skills/contacts-framework/) | CNContactStore, fetch requests, key descriptors, CNContactPickerViewController, save requests |
| [eventkit](skills/eventkit/) | EKEventStore, EKEvent, EKReminder, recurrence rules, EventKitUI editors and choosers |
| [financekit](skills/financekit/) | Apple Card, Apple Cash, Wallet orders, transaction queries, account balances |
| [healthkit](skills/healthkit/) | HKHealthStore, queries, statistics, workout sessions, background delivery |
| [musickit](skills/musickit/) | MusicKit authorization, catalog search, ApplicationMusicPlayer, MPRemoteCommandCenter |
| [passkit](skills/passkit/) | Apple Pay, PKPaymentRequest, PKPaymentAuthorizationController, Wallet passes |
| [weatherkit](skills/weatherkit/) | WeatherService, current/hourly/daily forecasts, alerts, attribution requirements |

### AI & Machine Learning

| Skill | What it covers |
|-------|---------------|
| [apple-on-device-ai](skills/apple-on-device-ai/) | Foundation Models framework, Core ML, MLX Swift, on-device LLM inference |
| [coreml](skills/coreml/) | Core ML model loading, prediction, MLTensor, compute unit configuration, VNCoreMLRequest, MLComputePlan |
| [natural-language](skills/natural-language/) | NLTokenizer, NLTagger, sentiment analysis, language identification, embeddings, Translation |
| [speech-recognition](skills/speech-recognition/) | SpeechAnalyzer, SpeechTranscriber, SFSpeechRecognizer, on-device recognition, audio buffer processing |
| [vision-framework](skills/vision-framework/) | Vision text recognition, face/barcode detection, image segmentation, VisionKit DataScannerViewController |

### iOS Engineering

| Skill | What it covers |
|-------|---------------|
| [app-store-optimization](skills/app-store-optimization/) | ASO keyword strategy, description writing, screenshot optimization, Custom Product Pages, A/B testing |
| [app-store-review](skills/app-store-review/) | App Review guidelines, rejection prevention, privacy manifests, ATT, HIG compliance |
| [authentication](skills/authentication/) | Sign in with Apple, ASAuthorizationController, passkeys, biometric auth (LAContext), credential management |
| [background-processing](skills/background-processing/) | BGTaskScheduler, background refresh, URLSession background transfers |
| [cryptokit](skills/cryptokit/) | SHA-2/SHA-3, HMAC, AES-GCM, ChaChaPoly, HPKE, ML-KEM/ML-DSA, P256/Curve25519 signing, ECDH, Secure Enclave |
| [debugging-instruments](skills/debugging-instruments/) | Xcode debugger, Instruments, os_signpost, MetricKit, crash symbolication |
| [device-integrity](skills/device-integrity/) | DeviceCheck (DCDevice per-device bits), App Attest (DCAppAttestService attestation and assertion flows) |
| [ios-accessibility](skills/ios-accessibility/) | VoiceOver, Dynamic Type, custom rotors, accessibility focus, assistive-technology support |
| [ios-ettrace-performance](skills/ios-ettrace-performance/) | ETTrace launch/runtime capture, exact-build dSYM matching, processed flamegraph JSON, comparable verification |
| [ios-localization](skills/ios-localization/) | String Catalogs, pluralization, FormatStyle, right-to-left layout |
| [ios-memgraph-analysis](skills/ios-memgraph-analysis/) | Simulator memgraph capture, leak ownership paths, reachable heap growth, raw evidence preservation |
| [ios-networking](skills/ios-networking/) | URLSession async/await, REST APIs, downloads/uploads, WebSockets, pagination, retry, caching |
| [swift-security](skills/swift-security/) | Keychain Services, CryptoKit symmetric/asymmetric, biometric authentication, Secure Enclave, certificate trust, credential storage, OWASP compliance · *Based on [ivan-magda/swift-security-skill](https://github.com/ivan-magda/swift-security-skill)* |
| [ios-simulator](skills/ios-simulator/) | xcrun simctl commands, device lifecycle, push/location/privacy simulation, log streaming, simulator limitations |
| [metrickit](skills/metrickit/) | MetricManager async reports, hang/crash diagnostics, production performance telemetry |
| [swiftlint](skills/swiftlint/) | SwiftLint setup, .swiftlint.yml, build tool plugin, rule selection, baselines, suppressions, CI integration |

### Hardware & Device Integration

| Skill | What it covers |
|-------|---------------|
| [accessorysetupkit](skills/accessorysetupkit/) | Privacy-preserving BLE/Wi-Fi accessory discovery, ASAccessorySession, picker UI |
| [core-bluetooth](skills/core-bluetooth/) | CBCentralManager, CBPeripheral, BLE scanning/connecting, services, characteristics, background modes |
| [core-motion](skills/core-motion/) | CMMotionManager, CMPedometer, accelerometer, gyroscope, activity recognition, altitude |
| [core-nfc](skills/core-nfc/) | NFCNDEFReaderSession, NFCTagReaderSession, NDEF reading/writing, background tag reading |
| [dockkit](skills/dockkit/) | DockAccessoryManager, camera subject tracking, motor control, framing |
| [pencilkit](skills/pencilkit/) | PKCanvasView, PKDrawing, PKToolPicker, Apple Pencil drawing and annotation |
| [realitykit](skills/realitykit/) | RealityView, entities, anchors, ARKit world tracking, raycasting, scene understanding |
| [sensorkit](skills/sensorkit/) | Research-grade sensor data, ambient light, keyboard metrics, device usage (approved studies) |

### Platform Integration

| Skill | What it covers |
|-------|---------------|
| [appmigrationkit](skills/appmigrationkit/) | Cross-platform data transfer, AppMigrationExtension export/import (iOS 26) |
| [audioaccessorykit](skills/audioaccessorykit/) | Audio accessory features, automatic switching, device placement (iOS 26.4) |
| [browserenginekit](skills/browserenginekit/) | Alternative browser engines (EU), process management, web content extensions |
| [callkit](skills/callkit/) | CXProvider, CXCallController, PushKit VoIP registration, call directory extensions |
| [cryptotokenkit](skills/cryptotokenkit/) | TKTokenDriver, TKSmartCard, iOS 26 NFC smart cards, certificate-based auth |
| [energykit](skills/energykit/) | ElectricityGuidance, EnergyVenue, grid forecasts, load event submission, electricity insights |
| [homekit](skills/homekit/) | HMHomeManager, accessories, rooms, actions, triggers, MatterSupport commissioning |
| [permissionkit](skills/permissionkit/) | AskCenter, PermissionQuestion, child communication safety, CommunicationLimits |
| [relevancekit](skills/relevancekit/) | Widget relevance signals, time/location-based relevance providers (watchOS 26) |
| [shareplay-activities](skills/shareplay-activities/) | GroupActivity, GroupSession, GroupSessionMessenger, coordinated media playback |

### Gaming

| Skill | What it covers |
|-------|---------------|
| [gamekit](skills/gamekit/) | Game Center, GKLocalPlayer, leaderboards, achievements, real-time and turn-based multiplayer |
| [scenekit](skills/scenekit/) | SCNView, SCNScene, 3D geometry, materials, lighting, physics, SceneView |
| [spritekit](skills/spritekit/) | SKScene, SKSpriteNode, SKAction, physics simulation, particle effects, SpriteView |
| [tabletopkit](skills/tabletopkit/) | Multiplayer spatial board games, pieces, cards, dice, Group Activities (visionOS) |

## Yapı

Her becerinin ana yönergeleri `skills/<beceri>/SKILL.md` dosyasında, ayrıntılı örnekleri ise gerektiğinde `references/` altında bulunur. Beceri dosyaları [Agent Skills](https://agentskills.io) standardını kullanır.

## Lisans

[PolyForm Perimeter 1.0.0](LICENSE). Bu fork, özgün projenin telif ve lisans bildirimlerini korur. Apple API’leri, belgeleri ve markaları Apple’a aittir; bu proje Apple ile bağlantılı değildir.
