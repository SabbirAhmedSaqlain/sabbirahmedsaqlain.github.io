# iOS Developer Interview Questions and Answers: A Complete Beginner-to-Senior Guide

iOS interviews test more than Swift syntax. Strong candidates explain tradeoffs, connect language features to production behavior, reason about memory and concurrency, design testable systems, and describe how they diagnose real problems.

This guide progresses from fundamentals to senior-level architecture and leadership questions. The answers begin with an interview-ready response and then add the reasoning, examples, and common follow-up points.

## Short Answer: What Should an iOS Developer Prepare?

Prepare these core areas:

- Swift value and reference semantics, optionals, protocols, generics, closures, and error handling;
- ARC, ownership, retain cycles, `weak`, and `unowned` references;
- Swift concurrency, tasks, actors, `MainActor`, `Sendable`, cancellation, and data-race safety;
- UIKit and SwiftUI lifecycle, state management, layout, navigation, and interoperability;
- architecture, dependency injection, modularity, and testability;
- `URLSession`, Codable, caching, retries, authentication, and offline behavior;
- Core Data, SwiftData, files, UserDefaults, and Keychain;
- unit, integration, UI, asynchronous, and performance testing;
- Instruments, launch time, scrolling, memory, energy, and production diagnostics;
- system design, security, release strategy, communication, and ownership.

For most questions, use this answer structure:

```text
1. Define the concept clearly.
2. Explain why it matters.
3. Give a small practical example.
4. Name the main tradeoff or failure mode.
5. Connect it to something you built or debugged.
```

## Part 1: Swift Fundamentals

### 1. What is the difference between `let` and `var`?

**Short answer:** `let` declares an immutable binding; `var` declares a mutable binding. Prefer `let` unless mutation is required because immutability reduces accidental state changes and makes code easier to reason about.

For a value type, a `let` binding prevents mutation of its stored properties. For a class instance, `let` prevents replacing the reference, but mutable properties of the referenced object can still change.

```swift
struct Profile { var name: String }

let valueProfile = Profile(name: "Sabbir")
// valueProfile.name = "Ahmed" // Not allowed

final class Session { var token = "A" }
let session = Session()
session.token = "B" // Allowed; the reference itself did not change
```

### 2. What is the difference between a `struct` and a `class`?

**Short answer:** Structures are value types and are copied on assignment; classes are reference types and share identity. Classes also support inheritance, deinitializers, and reference counting.

Use a structure when independent values and predictable local mutation fit the model. Use a class when identity, shared mutable state, Objective-C interoperability, or lifecycle ownership matters. Do not choose classes merely because the object is large; Swift collections and many standard value types use copy-on-write to avoid unnecessary copying.

### 3. What are value semantics and reference semantics?

**Short answer:** With value semantics, each variable behaves as an independent value. With reference semantics, multiple variables can point to the same instance.

```swift
struct CounterValue { var count = 0 }
final class CounterReference { var count = 0 }

var a = CounterValue()
var b = a
b.count = 1       // a.count remains 0

let x = CounterReference()
let y = x
y.count = 1       // x.count is also 1
```

Value semantics are useful for models, configuration, and state snapshots. Reference semantics are appropriate when shared identity is intentional.

### 4. What is an optional?

**Short answer:** An optional represents either a wrapped value or the absence of a value. Swift uses `Optional<Wrapped>`, with `.some(value)` and `.none` cases.

Optionals make absence explicit in the type system. Handle them with optional binding, `guard`, optional chaining, `map`, `flatMap`, nil coalescing, or pattern matching. Force unwrap only when the invariant is guaranteed and a crash is genuinely the correct response to a programming error.

### 5. When would you use `if let`, `guard let`, and `??`?

**Short answer:** Use `if let` for conditional work within a branch, `guard let` for required preconditions and early exit, and `??` to provide a default value.

```swift
func display(user: User?) {
    guard let user else { return }
    let title = user.displayName ?? "Anonymous"

    if let avatar = user.avatarURL {
        loadImage(from: avatar)
    }

    titleLabel.text = title
}
```

`guard` keeps the successful path less nested and is especially useful in parsing, validation, and delegate callbacks.

### 6. What is a protocol in Swift?

**Short answer:** A protocol defines a contract of properties, methods, initializers, or associated types that conforming types implement.

Protocols support abstraction, composition, dependency injection, and test doubles. Protocol extensions can provide default behavior, but dispatch rules matter: requirements declared in the protocol participate in dynamic dispatch through an existential; methods added only in an extension are selected statically based on the visible type.

### 7. What are protocol extensions used for?

**Short answer:** Protocol extensions add shared implementations or helper behavior to conforming types.

```swift
protocol IdentifiableRequest {
    var requestID: UUID { get }
}

extension IdentifiableRequest {
    var requestHeader: String { requestID.uuidString }
}
```

Use default implementations to remove meaningful duplication, but avoid placing unrelated behavior into broad protocols. Small, capability-focused protocols are easier to understand and test.

### 8. What are generics?

**Short answer:** Generics let functions and types operate on placeholder types while preserving compile-time type safety.

```swift
func first<T>(in values: [T]) -> T? {
    values.first
}
```

Generics avoid duplication and usually permit specialization by the compiler. Constraints such as `T: Decodable` express required capabilities. Prefer generics when the concrete type can remain known at compile time; use existential types when heterogeneous runtime values are needed.

### 9. What is the difference between `some Protocol` and `any Protocol`?

**Short answer:** `some Protocol` is an opaque type: one specific conforming type is hidden from the caller. `any Protocol` is an existential container that can hold different conforming types at runtime.

Opaque types preserve more static type information and are common in SwiftUI's `some View`. Existentials provide runtime flexibility but may introduce boxing, dynamic dispatch, and limitations involving associated types. Choose based on whether identity of the concrete type should remain fixed or vary.

### 10. What is an associated type?

**Short answer:** An associated type is a placeholder type that a protocol's conforming type supplies.

```swift
protocol Repository {
    associatedtype Entity
    func load() async throws -> [Entity]
}
```

Associated types make protocols expressive for strongly typed relationships. If callers need to store different repositories behind one uniform runtime type, type erasure or a redesign around a concrete generic boundary may be needed.

### 11. What is the difference between escaping and nonescaping closures?

**Short answer:** A nonescaping closure must finish before the function returns. An `@escaping` closure can be stored or called later.

Completion handlers are often escaping. Escaping closures require explicit `self` in many contexts, making capture semantics visible. With modern APIs, `async` functions often replace completion-handler pyramids, but callbacks remain common for delegates, event streams, and interoperability.

### 12. How do capture lists work?

**Short answer:** A capture list controls how a closure captures values or object references.

```swift
service.load { [weak self] result in
    guard let self else { return }
    self.render(result)
}
```

Use `[weak self]` when the closure may outlive the owner and should not keep it alive. Do not add weak captures mechanically: a short-lived nonescaping closure usually does not create a cycle, and silently dropping required work can be incorrect.

### 13. What do `map`, `compactMap`, and `flatMap` do?

**Short answer:** `map` transforms each element, `compactMap` transforms and removes nil results, and `flatMap` transforms then flattens one level of nested sequence.

For optionals, `map` transforms a wrapped value while preserving optionality; `flatMap` is useful when the transform itself returns an optional and nested optionals are not desired.

### 14. How does Swift error handling work?

**Short answer:** A throwing function uses `throws`; callers handle errors with `try`, `do/catch`, propagation, optional conversion with `try?`, or forced execution with `try!` when failure is impossible by invariant.

Model errors with meaningful types:

```swift
enum APIError: Error {
    case invalidResponse
    case unauthorized
    case server(statusCode: Int)
    case decoding(underlying: Error)
}
```

Preserve enough context for diagnostics while mapping internal details to safe, useful user messages.

### 15. What is `Result` and when is it useful?

**Short answer:** `Result<Success, Failure>` stores either a success value or a typed failure. It is useful in callback APIs, cached outcomes, and boundaries where an operation's result must be passed as data.

With `async throws`, direct return and throw are usually clearer for a single asynchronous operation. `Result` remains valuable for completion handlers, batch results, and event streams.

### 16. What are property wrappers?

**Short answer:** A property wrapper encapsulates storage and access behavior that can be reused through an attribute such as `@State` or a custom wrapper.

Wrappers can implement validation, dependency access, persistence, or observation, but they should not hide surprising side effects. Understand the wrapped value, projected value, and ownership semantics before using one in shared architecture.

## Part 2: ARC and Memory Management

### 17. What is ARC?

**Short answer:** Automatic Reference Counting tracks strong references to class instances and deinitializes an instance when its strong-reference count reaches zero.

ARC is deterministic reference management, not tracing garbage collection. It manages class instances, while value types follow their own storage semantics. ARC cannot automatically break strong reference cycles.

### 18. What is a strong reference cycle?

**Short answer:** A cycle occurs when objects keep strong references to one another, preventing their counts from reaching zero.

```swift
final class Parent { var child: Child? }
final class Child { weak var parent: Parent? }
```

The relationship that does not own the other object should usually be `weak` or, when lifetime is guaranteed, `unowned`.

### 19. What is the difference between `weak` and `unowned`?

**Short answer:** A weak reference is optional and becomes `nil` when the object deallocates. An unowned reference is expected to remain valid for every access and traps if that assumption is violated.

Use `weak` for delegates and relationships where the referenced object may disappear first. Use `unowned` only when the lifetime relationship is certain, such as a closure owned by an object that cannot outlive that object under the design.

### 20. How can a closure create a retain cycle?

**Short answer:** If an object strongly owns a closure and the closure strongly captures the object, they keep each other alive.

Common sources include stored callbacks, timers, display links, Combine subscriptions, notification observers, and long-lived tasks. Fix the ownership design, invalidate registrations, cancel work, or use an appropriate weak capture.

### 21. How do you investigate a memory leak?

**Short answer:** Reproduce a lifecycle that should deallocate, verify `deinit`, then use Xcode's memory graph and Instruments Allocations or Leaks to inspect retain paths and growth.

Look for closure captures, delegates that should be weak, observers, timers, tasks, caches, and framework objects with retained delegates. Compare repeated navigation or workload cycles, not only one snapshot.

## Part 3: Swift Concurrency

### 22. What do `async` and `await` mean?

**Short answer:** `async` marks a function that can suspend. `await` marks a call that may suspend the current task until progress can continue.

Suspension does not block a thread. After suspension, the task may resume on a different thread unless actor isolation constrains where the code runs. `await` is also a reasoning boundary because actor state may have changed before execution resumes.

### 23. What is structured concurrency?

**Short answer:** Structured concurrency keeps child tasks within a lexical parent scope so lifetimes, cancellation, priority, and errors form a predictable hierarchy.

`async let` and task groups are structured. They make it clear when concurrent work finishes and prevent background tasks from accidentally outliving the operation that created them.

### 24. What is the difference between `async let` and a task group?

**Short answer:** Use `async let` for a fixed, small number of known child operations. Use a task group for a dynamic collection of tasks.

```swift
async let profile = api.profile()
async let settings = api.settings()
return try await (profile, settings)
```

Task groups can add work dynamically and consume results as children complete. Both propagate cancellation according to structured-concurrency rules.

### 25. What is an actor?

**Short answer:** An actor is a reference type that protects its mutable isolated state by allowing actor-isolated code to access that state in a serialized manner.

Calls across actor boundaries usually require `await`. Actors address data races, but they do not automatically make multi-step business logic atomic across suspension points.

### 26. What is `@MainActor`?

**Short answer:** `@MainActor` isolates declarations to the main actor, which is used for UI-related state and operations.

Annotate view models or methods that own UI-facing mutable state when appropriate. Do not run expensive parsing, image processing, or ML inference on the main actor. Perform that work in a suitable isolation domain and return only the result needed by the UI.

### 27. What is `Sendable`?

**Short answer:** `Sendable` marks data that can safely cross concurrency-domain boundaries.

Value types composed of sendable values are often naturally sendable. Mutable reference types need isolation, synchronization, or redesign. `@unchecked Sendable` tells the compiler to trust a manual guarantee and should be rare, documented, and audited.

### 28. What is actor reentrancy?

**Short answer:** When an actor-isolated method suspends at `await`, other work may run on that actor before the method resumes.

Therefore, validate actor state again after suspension when correctness depends on it:

```swift
actor TokenStore {
    private var generation = 0

    func refresh() async throws {
        let startingGeneration = generation
        let token = try await fetchToken()
        guard generation == startingGeneration else { return }
        save(token)
    }
}
```

Actors prevent simultaneous unsynchronized access, not logical races caused by stale assumptions.

### 29. How does task cancellation work?

**Short answer:** Cancellation is cooperative. A task receives a cancellation signal, but code must observe it and stop appropriately.

Use `Task.checkCancellation()`, `Task.isCancelled`, cancellation-aware APIs, and cleanup handlers when needed. Network and child tasks should not continue consuming resources after the associated screen or operation is gone.

### 30. What is the difference between `Task` and `Task.detached`?

**Short answer:** A normal `Task` inherits relevant actor context, priority, and task-local values. A detached task starts outside that structured context.

Prefer structured child tasks or normal tasks. Use detached tasks only when the operation is intentionally independent and all captured data is safe to cross isolation boundaries.

### 31. How do you bridge a completion-handler API to `async`?

**Short answer:** Use checked continuations when no native async API exists.

```swift
func load() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        legacyLoad { result in
            continuation.resume(with: result)
        }
    }
}
```

Resume exactly once on every path. A missing resume hangs the task; multiple resumes are a correctness error. Also bridge cancellation when the underlying operation supports it.

### 32. What is the difference between concurrency and parallelism?

**Short answer:** Concurrency is structuring multiple tasks that can make progress over overlapping time. Parallelism is simultaneous execution on multiple cores.

An async network task is concurrent even while no CPU work runs. Image transformations may run in parallel. Good iOS design cares about responsiveness, isolation, cancellation, and bounded resource use rather than manually choosing threads for every operation.

### 33. When would you use GCD instead of Swift concurrency?

**Short answer:** Prefer Swift concurrency for new asynchronous application logic. Use GCD for interoperability, low-level queue-specific APIs, or existing code that cannot yet migrate safely.

Avoid mixing models casually. Repeatedly wrapping async functions in dispatch queues can obscure actor isolation and create unnecessary thread hops.

## Part 4: UIKit and App Lifecycle

### 34. Explain the iOS application lifecycle.

**Short answer:** An app moves through launch, inactive, active, background, and suspended behavior. Scene-based UIKit apps handle per-interface lifecycle events through `UISceneDelegate`, while the app delegate handles shared application events.

At transitions, pause timers and capture, save appropriate state, protect sensitive snapshots, finish only permitted background work, and release rebuildable resources under memory pressure. Do not assume termination callbacks always run.

### 35. What is the difference between `AppDelegate` and `SceneDelegate`?

**Short answer:** `AppDelegate` manages process-wide application behavior. `SceneDelegate` manages one UI scene's lifecycle, window, state, and incoming activities in scene-based UIKit apps.

An application may have multiple scenes, each in a different state. Keep shared service initialization separate from scene-specific UI creation.

### 36. Explain the `UIViewController` lifecycle.

**Short answer:** Important methods include `loadView`, `viewDidLoad`, `viewWillAppear`, `viewDidAppear`, `viewWillDisappear`, `viewDidDisappear`, layout callbacks, and `deinit`.

Create or load the view hierarchy in `loadView`; perform one-time controller setup in `viewDidLoad`; refresh appearance-dependent data in `viewWillAppear`; start work that requires visibility in `viewDidAppear`; and stop or persist appropriate work while disappearing. Repeated appearance calls must remain safe.

### 37. What is the difference between `viewDidLoad` and `viewWillAppear`?

**Short answer:** `viewDidLoad` normally runs once for a controller's loaded view. `viewWillAppear` runs before each appearance.

Use `viewDidLoad` for stable setup such as view binding and static layout. Use `viewWillAppear` for state that may have changed while another screen was presented. Avoid starting duplicate subscriptions or requests on every appearance without cancellation.

### 38. How does Auto Layout work?

**Short answer:** Auto Layout solves a system of constraints describing relationships among view positions and sizes.

Constraints need enough information to produce an unambiguous layout without required conflicts. Use safe areas, layout guides, readable content guides, and Dynamic Type. Diagnose warnings by finding the mutually incompatible required constraints rather than hiding the log.

### 39. What are content hugging and compression resistance?

**Short answer:** Hugging controls how strongly a view resists growing beyond its intrinsic size. Compression resistance controls how strongly it resists shrinking below that size.

When labels and controls compete for space, priorities tell Auto Layout which view may stretch or compress. Test long localized strings and accessibility text sizes, not only short English content.

### 40. How does cell reuse work in table and collection views?

**Short answer:** Reusable cells are recycled as content scrolls, reducing allocation and memory use.

Configuration must set every visible state because a cell may contain values from a previous item. Cancel or identify asynchronous image requests, reset transient state in `prepareForReuse`, and update UI only if the cell still represents the expected model.

### 41. What is a diffable data source?

**Short answer:** A diffable data source applies snapshots of stable section and item identifiers and calculates UI changes safely.

Identifiers must represent identity, not changing display values. Build snapshots from the source of truth, apply them on the expected isolation context, and avoid duplicate identifiers.

### 42. What is the responder chain?

**Short answer:** The responder chain is the ordered route through which UIKit sends events and actions when a specific target is not fixed.

Views, view controllers, windows, and the application can participate. It enables target-action behavior, menu commands, keyboard handling, and event forwarding without tightly coupling the sender to a receiver.

### 43. Why must UI updates happen on the main actor?

**Short answer:** UIKit and SwiftUI UI state is main-actor isolated because interface frameworks require serialized interaction with their objects.

Keep heavy work away from the main actor, but return to main-actor isolation for UI-facing mutations. Do not assume a completion callback arrives on the main thread unless the API guarantees it.

### 44. How would you implement navigation in a large UIKit app?

**Short answer:** Keep navigation decisions outside individual view controllers using a coordinator, router, or feature-level navigation owner.

View controllers emit user intents; the navigation layer composes screens and dependencies. This reduces coupling, improves deep-link handling, and makes flows easier to test. Avoid a single global coordinator that knows every feature detail.

### 45. How do you handle deep links safely?

**Short answer:** Parse a deep link into a typed route, validate scheme, host, path, parameters, authentication state, and authorization, then navigate through the application's routing system.

A deep link may select a screen but must never grant access to protected data. Object-level permission remains a server responsibility.

## Part 5: SwiftUI

### 46. How is SwiftUI different from UIKit?

**Short answer:** SwiftUI is declarative: views describe the UI for current state. UIKit is primarily imperative: code creates and updates view objects directly.

SwiftUI offers state-driven composition and cross-platform reuse. UIKit provides mature low-level control and is common in established applications. Production apps can mix both through representable and hosting-controller bridges.

### 47. What do `@State` and `@Binding` mean?

**Short answer:** `@State` stores transient source-of-truth state owned by a view. `@Binding` provides read-write access to state owned elsewhere.

Do not use `@State` merely to silence mutation errors. Decide who owns the data. A binding should represent intentional shared mutation, not an invisible global dependency.

### 48. What is the difference among `@StateObject`, `@ObservedObject`, and `@EnvironmentObject`?

**Short answer:** In the `ObservableObject` model, `@StateObject` creates and owns an observable reference for the view's lifetime, `@ObservedObject` observes an instance owned elsewhere, and `@EnvironmentObject` retrieves a shared instance from the environment.

With the newer Observation model, an `@Observable` reference can be held in `@State`, passed normally, accessed from `@Environment`, and exposed as bindings through `@Bindable`. Know the deployment-target requirements before choosing a model.

### 49. What causes a SwiftUI view to update?

**Short answer:** A view update is triggered when state or observed data that the view depends on changes. SwiftUI reevaluates `body` and reconciles the resulting view description with existing UI identity.

`body` may run often, so keep it free of side effects and expensive work. Start asynchronous work with lifecycle-aware APIs such as `.task`, and move computation into models or services.

### 50. Why is identity important in SwiftUI?

**Short answer:** Identity lets SwiftUI associate a new view description with existing state and rendered content.

Unstable IDs can reset local state, break animation, or cause incorrect list updates. Use durable model identifiers rather than a new UUID generated during every render.

### 51. What is the difference between `.task` and `.onAppear`?

**Short answer:** `.task` starts asynchronous work tied to the view's lifecycle and supports automatic cancellation. `.onAppear` runs a synchronous closure when the view appears.

Both may execute multiple times as views enter and leave the hierarchy. Make loading idempotent and model explicit states such as idle, loading, loaded, empty, and failed.

### 52. How do you improve SwiftUI performance?

**Short answer:** Keep state local, use stable identity, avoid expensive work in `body`, reduce unnecessary observation, use lazy containers for large content, and profile before optimizing.

Large observable objects can invalidate too much UI. Split state by responsibility, derive inexpensive values, cache costly transformations outside rendering, and inspect actual updates with Instruments or SwiftUI diagnostics.

### 53. How do UIKit and SwiftUI interoperate?

**Short answer:** Embed SwiftUI in UIKit with `UIHostingController`. Wrap UIKit views or controllers for SwiftUI with `UIViewRepresentable` or `UIViewControllerRepresentable`.

Define a clear ownership boundary for navigation, state, delegates, and lifecycle. Wrappers should translate between frameworks rather than become another application architecture.

## Part 6: Architecture and Dependency Design

### 54. Explain MVC, MVVM, Coordinator, and Clean Architecture.

**Short answer:** MVC separates model, view, and controller; MVVM introduces a presentation model between UI and domain behavior; Coordinator moves navigation composition out of screens; Clean Architecture organizes dependencies around domain rules and boundaries.

No pattern is automatically correct. Choose the smallest architecture that creates clear ownership, testability, and change isolation. A small feature may need only a view controller and service. A large regulated app may benefit from modules, use cases, repositories, and explicit navigation ownership.

### 55. What makes a view model useful?

**Short answer:** A view model converts domain state and operations into presentation-ready state and user intents without depending unnecessarily on concrete views.

A useful view model owns loading, empty, content, and failure states; invokes injected dependencies; and remains testable without a device. It should not become a container for navigation, networking, persistence, analytics, and every shared service.

### 56. What is dependency injection?

**Short answer:** Dependency injection provides collaborators from outside a type rather than constructing hidden concrete dependencies inside it.

```swift
protocol UserLoading {
    func loadUser() async throws -> User
}

final class ProfileViewModel {
    private let loader: UserLoading

    init(loader: UserLoading) {
        self.loader = loader
    }
}
```

Initializer injection makes required dependencies explicit and supports test doubles. Avoid turning a service locator into invisible global state.

### 57. When should you create a protocol?

**Short answer:** Create a protocol at a meaningful substitution or ownership boundary, not automatically for every concrete type.

Good candidates include external services, repositories, clocks, random generators, file access, analytics, and feature interfaces. A one-method protocol created only to satisfy a pattern can increase indirection without reducing risk.

### 58. What is a repository pattern?

**Short answer:** A repository provides a domain-oriented interface over one or more data sources such as network, memory cache, and local persistence.

The repository decides where data comes from and how sources synchronize. It should not leak transport DTOs or persistence-specific objects into unrelated UI code when domain models are more appropriate.

### 59. How would you modularize a large iOS application?

**Short answer:** Create modules around durable features or platform capabilities with clear public interfaces and controlled dependencies.

Typical modules include feature UI, domain contracts, networking, persistence, design system, analytics, authentication, and shared test support. Avoid a giant `Common` module. Enforce an acyclic dependency graph and keep feature internals hidden.

### 60. How do you decide between a singleton and an injected service?

**Short answer:** Prefer injected services for explicit ownership, configuration, and testing. Use process-wide shared instances only when the resource is genuinely singular and its global lifetime is intentional.

Even a shared service can be hidden behind an injected protocol. Global mutable singletons create test coupling, unclear initialization order, and concurrency risk.

### 61. How do you model UI state?

**Short answer:** Represent mutually exclusive states explicitly instead of combining unrelated booleans.

```swift
enum ScreenState {
    case idle
    case loading
    case content([Item])
    case empty
    case failed(UserFacingError)
}
```

An enum prevents impossible combinations such as loading and failed at the same time. Separate persistent domain state from transient presentation state.

## Part 7: Networking

### 62. How does `URLSession` work?

**Short answer:** `URLSession` coordinates data, upload, download, WebSocket, and streaming tasks according to a configuration and optional delegate.

Use ephemeral or default sessions according to caching and cookie requirements. Validate the HTTP response and status code before decoding. Custom sessions strongly retain their delegates until invalidated, so lifecycle design matters for long-lived delegated sessions.

### 63. How would you build a type-safe API client?

**Short answer:** Define requests with method, path, headers, body, and response type; centralize transport and validation; decode into boundary models; and expose domain-friendly errors.

```swift
protocol APIRequest {
    associatedtype Response: Decodable
    var path: String { get }
    var method: HTTPMethod { get }
}

func send<R: APIRequest>(_ request: R) async throws -> R.Response
```

Keep authentication refresh, request IDs, logging redaction, and environment configuration in appropriate layers rather than duplicating them in every screen.

### 64. How do you use `Codable` safely?

**Short answer:** Define decoding models that match the external contract, configure key and date strategies deliberately, and convert to domain models when the API shape should not spread through the app.

Do not make every field optional simply to avoid decoding failures. Optionality should represent a valid absence. Capture decoding context in internal diagnostics without exposing raw sensitive payloads.

### 65. How should an app handle HTTP errors?

**Short answer:** Separate transport failures, invalid responses, HTTP status failures, decoding failures, authentication failures, cancellation, and domain errors.

Use status-specific behavior: refresh or sign out according to policy for authorization failures, respect rate-limit guidance, retry only suitable transient failures, and map errors to actionable user states. A `200` response is not enough if the body violates the contract.

### 66. How do you implement retries?

**Short answer:** Retry only transient, safe operations using bounded attempts, exponential backoff, jitter, cancellation, and server guidance such as `Retry-After`.

Do not blindly retry non-idempotent requests. Use idempotency keys or server-designed operation identifiers for uploads, payments, and submissions that must not execute twice.

### 67. How would you refresh an expired access token?

**Short answer:** Coordinate refresh so multiple failed requests await one in-flight refresh operation, then retry eligible requests once with the new token.

Protect token state with actor isolation or another correct synchronization model. If refresh fails definitively, clear the session through one controlled path. Prevent infinite retry loops and avoid logging token values.

### 68. How do you cache network data?

**Short answer:** Choose a cache based on freshness, size, sensitivity, offline needs, and server semantics.

Use HTTP caching when headers support it, memory caches for rebuildable hot data, and persistence for offline or durable state. Define invalidation and versioning before adding a cache. Sensitive responses may require no-store behavior and protected local storage.

### 69. What is a background `URLSession`?

**Short answer:** A background session lets the system perform eligible uploads and downloads in a separate process and relaunch or wake the app to deliver completion events.

It is appropriate for large file transfers that should continue when the app is suspended. Persist enough task-to-domain mapping to reconstruct state after relaunch, and handle discretionary scheduling, connectivity, cancellation, and user-visible progress.

### 70. How do you secure network communication?

**Short answer:** Use HTTPS with correct certificate and hostname validation, App Transport Security, secure authentication, minimal sensitive payloads, and server-side authorization.

Certificate pinning may add defense in depth for controlled high-risk endpoints, but it requires rotation and recovery planning and can be bypassed on a compromised client. Never embed server secrets in the application.

## Part 8: Persistence and Security

### 71. When should you use UserDefaults, Keychain, files, Core Data, or SwiftData?

**Short answer:** Use UserDefaults for small preferences, Keychain for suitable credentials and secrets, files for document or media data, and Core Data or SwiftData for structured persistent object graphs and queries.

Do not store passwords, long-lived tokens, or identity data in UserDefaults. Choose storage based on sensitivity, query needs, migrations, synchronization, volume, and deployment target.

### 72. What is Core Data?

**Short answer:** Core Data is an object-graph management and persistence framework with identity, relationships, change tracking, validation, migrations, undo support, and multiple store options.

It is not simply a thin SQLite wrapper. Managed objects belong to managed object contexts, and context concurrency rules must be respected. Use stable identifiers or object IDs when passing identity across contexts.

### 73. What is SwiftData?

**Short answer:** SwiftData provides a Swift-native persistence model integrated with Swift macros and SwiftUI while building on Apple's persistence infrastructure.

It can reduce boilerplate for supported deployment targets, but architecture still needs explicit model ownership, migrations, testing, background work, error handling, and boundaries between persistence models and business rules.

### 74. How do you handle database migrations?

**Short answer:** Version the schema, define compatible lightweight changes when possible, test migrations using real previous stores, and create explicit mapping or staged migrations for complex changes.

Back up or recover safely according to data criticality. Test upgrades across every supported previous application version, including interrupted migrations and low-storage conditions.

### 75. What should be stored in Keychain?

**Short answer:** Keychain is appropriate for small sensitive values such as credentials, tokens, and cryptographic keys, with an accessibility policy matching the security and background-access requirements.

Keychain items may survive app reinstall depending on configuration and platform behavior, so do not assume reinstall clears identity. Define logout, account-switching, device-lock, backup, synchronization, and biometric-access behavior explicitly.

### 76. How should an iOS app protect sensitive data?

**Short answer:** Minimize collection, use Keychain and data-protection classes appropriately, encrypt or avoid retained sensitive files, redact logs, protect background snapshots, secure network traffic, and enforce authorization on the server.

For NID, face, healthcare, or financial data, also control temporary image retention, backups, analytics payloads, clipboard behavior, screenshots, and crash attachments. The mobile client should never be the authoritative source of KYC, role, or payment status.

### 77. How do biometrics fit into application security?

**Short answer:** Use LocalAuthentication to request platform authentication and, for high-value local secrets, bind access to Keychain or Secure Enclave-backed key operations where appropriate.

A successful Face ID prompt should not merely set an editable Boolean that unlocks server operations forever. Define fallback, enrollment-change, cancellation, lockout, fresh-authentication, and transaction-binding policies.

## Part 9: Testing

### 78. What is the difference among unit, integration, and UI tests?

**Short answer:** Unit tests verify a small component in isolation; integration tests verify components or boundaries together; UI tests drive the application through accessibility interfaces and validate user flows.

Use many fast deterministic unit tests, targeted integration tests for risky boundaries, and fewer high-value UI tests for critical journeys. Test behavior and contracts rather than private implementation details.

### 79. How do you test asynchronous code?

**Short answer:** Mark XCTest or Swift Testing functions `async` or `async throws` and await the operation directly. Use expectations or confirmations for callback- and event-driven behavior.

```swift
func testLoadReturnsProfile() async throws {
    let profile = try await service.loadProfile()
    XCTAssertEqual(profile.name, "Sabbir")
}
```

Avoid arbitrary sleeps. Inject clocks, schedulers, clients, and deterministic fakes so timing behavior is controllable.

### 80. What is the difference between a mock, stub, fake, and spy?

**Short answer:** A stub returns configured data; a spy records interactions; a mock verifies expected interactions; a fake provides a lightweight working implementation.

Terminology varies, so explain the behavior you need. Prefer state-based assertions where possible. Over-mocking couples tests to call order and implementation instead of outcomes.

### 81. How would you test a view model?

**Short answer:** Inject deterministic dependencies, invoke a user intent, observe resulting state or output, and assert success, empty, failure, cancellation, and retry paths.

Do not require a real network, database, clock, or main run-loop delay for a unit test. If the view model is `@MainActor`, keep test isolation consistent and await asynchronous transitions.

### 82. What makes UI tests reliable?

**Short answer:** Stable accessibility identifiers, controlled data, deterministic launch arguments, explicit waiting for observable conditions, isolated accounts, and cleanup make UI tests reliable.

Avoid fixed sleeps and coordinate-based taps when semantic elements are available. Keep a small suite focused on login, purchase or submission, critical navigation, accessibility, and major regressions.

### 83. What are performance tests?

**Short answer:** Performance tests repeatedly measure code or application behavior and detect regressions in duration, memory, CPU, storage, or launch metrics.

Use representative data, control the environment, establish baselines carefully, and investigate variance. A microbenchmark does not replace end-to-end profiling on physical devices.

## Part 10: Performance and Debugging

### 84. Which Instruments templates should an iOS developer know?

**Short answer:** Time Profiler, Allocations, Leaks, Network, Energy Log, Core Animation, Points of Interest, and app-launch or hang diagnostics cover many common problems.

Start from a reproducible symptom and measure before changing code. Signposts and points of interest make user journeys visible across subsystems.

### 85. How do you improve application launch time?

**Short answer:** Minimize work before the first responsive frame, defer nonessential initialization, avoid synchronous disk and network work, reduce dynamic loading cost, and measure cold and warm launch separately.

Initialize only critical dependencies, move migrations and heavy parsing to controlled asynchronous phases where safe, and use launch metrics on real devices. Prewarming means code should not assume launch always immediately produces visible UI.

### 86. How do you improve scrolling performance?

**Short answer:** Keep main-thread cell configuration small, reuse views correctly, precompute expensive formatting, resize images to display needs, cancel offscreen work, and reduce unnecessary layout or rendering passes.

Profile frame hitches instead of guessing. Common causes include image decoding, text measurement, Auto Layout churn, shadow rendering, synchronous I/O, and large SwiftUI invalidation regions.

### 87. How do you design an image-loading pipeline?

**Short answer:** Use request deduplication, memory and disk caching, correct downsampling, cancellation, priority, placeholder/error states, and identity checks before applying results.

Decode and resize away from the main actor, but assign the final image through UI isolation. Include the transformation size and content variant in the cache key.

### 88. How do you respond to memory pressure?

**Short answer:** Release rebuildable caches and large temporary resources, stop unnecessary work, and reduce future memory demand without discarding required user state.

Investigate the normal memory curve first. Repeated growth may indicate a leak; a stable high watermark may indicate oversized images, model buffers, or unbounded caches.

### 89. How do you debug a crash that you cannot reproduce?

**Short answer:** Symbolicate the crash, group by signature, inspect exception and thread state, correlate app version, device, OS, breadcrumbs, network state, and recent releases, then form and test hypotheses.

Preserve privacy by redacting sensitive data. Add targeted observability, not blanket logging. Validate the fix against the failing code path and monitor the crash-free metric after release.

### 90. What causes application hangs?

**Short answer:** Common causes include synchronous I/O on the main thread, lock contention, deadlocks, expensive layout, large parsing work, semaphore waits, and blocking calls inside actor-isolated UI code.

Use hang diagnostics, main-thread stacks, signposts, and Time Profiler. Replace blocking waits with asynchronous composition and reduce critical-section duration.

## Part 11: System Design Questions

### 91. How would you design an offline-first mobile feature?

**Short answer:** Make local durable state the immediate source for UI, queue user operations, synchronize with the server, and define conflict, retry, ordering, and deletion semantics.

Track operation identity and sync status. Design for process death, partial uploads, duplicate delivery, schema migration, clock differences, and account changes. Offline-first is a consistency model, not only a cache.

### 92. How would you design a paginated feed?

**Short answer:** Use stable item identity, cursor-based pagination when supported, deduplication, cancellation, cache policy, refresh semantics, and explicit loading/error states.

Prevent simultaneous duplicate page requests. Merge updates deterministically, preserve scroll position, and decide how inserts or deletions affect existing cursors.

### 93. How would you design a secure image-upload flow?

**Short answer:** Validate the image locally for UX, create a server-owned upload operation, compress and normalize safely, upload with cancellation and idempotency, and let the backend validate decoded content, authorization, and workflow state.

For NID or face images, protect temporary storage, strip unnecessary metadata, prevent duplicate submissions, report progress, handle poor networks, and delete retained media according to policy.

### 94. How would you integrate an on-device ML model?

**Short answer:** Define the model's exact input/output contract, preprocess consistently, run inference away from UI isolation, bound frame rate and memory, smooth noisy results, and expose typed domain outcomes.

Measure latency, energy, thermal behavior, false positives, false negatives, and device compatibility. Version the model and labels together, preserve privacy, and design a safe fallback for unsupported or low-confidence results.

### 95. How would you migrate a UIKit application toward SwiftUI?

**Short answer:** Migrate incrementally at feature boundaries, establish shared state and navigation rules, host SwiftUI from UIKit or wrap UIKit where necessary, and measure behavior rather than rewriting everything.

Start with isolated screens or reusable components. Keep domain and service layers UI-framework independent. Avoid running two competing navigation and dependency systems without a documented boundary.

### 96. How do you design observability for a mobile application?

**Short answer:** Collect privacy-safe crashes, hangs, performance metrics, structured events, network outcomes, and feature success signals with build, OS, and device context.

Use request correlation IDs to connect mobile and backend behavior. Redact tokens, identity data, and raw payloads. Define alerts and dashboards around user impact, not merely log volume.

### 97. How do you roll out a risky feature safely?

**Short answer:** Use staged environments, automated tests, feature flags, internal and beta distribution, percentage rollout, observability, rollback or kill-switch capability, and explicit success and stop conditions.

Schema and API compatibility must survive mixed application versions because users do not update simultaneously. Security controls should fail safely and should not depend solely on a client-side feature flag.

## Part 12: Senior and Behavioral Questions

### 98. Tell me about a difficult production problem you solved.

**Strong answer structure:** Describe the user impact, evidence, constraints, hypotheses, diagnosis, decision, implementation, verification, and prevention.

Use a real example. Explain what you personally did and what the team did. Include measurements such as crash rate, latency, failed sessions, or support volume. Finish with the system improvement, not only the emergency patch.

### 99. How do you estimate an uncertain project?

**Short answer:** Break work into outcomes and risks, identify unknowns, run short discovery spikes, expose dependencies, estimate ranges, and update confidence as evidence changes.

Separate target dates from evidence-based forecasts. Present scope, time, quality, and staffing tradeoffs rather than hiding uncertainty in one precise number.

### 100. How do you handle disagreement in a technical design review?

**Short answer:** Align on the problem and constraints, make tradeoffs explicit, examine evidence, invite affected expertise, and use the agreed decision owner when consensus is not necessary.

Document the decision and review conditions. Disagree respectfully, then support the selected direction unless new evidence or an unacceptable risk emerges.

### 101. What do you look for in code review?

**Short answer:** Correctness, clarity, tests, security, concurrency safety, performance where relevant, API design, maintainability, and consistency with architecture.

Review the risk, not personal style. Ask explanatory questions, identify blocking issues clearly, and automate formatting and mechanical checks. A review should improve both code and shared understanding.

### 102. How do you balance technical debt and feature delivery?

**Short answer:** Express debt as measurable delivery, reliability, security, or operational risk and prioritize it alongside product work.

Fix small debt continuously, reserve capacity for recurring health work, and propose larger investments with evidence and milestones. Avoid calling every disliked implementation technical debt.

### 103. How do you mentor another iOS developer?

**Short answer:** Understand the person's goals, give progressively broader ownership, pair on reasoning, provide timely specific feedback, and create opportunities to teach or lead.

Do not solve every problem for them. Adjust support as competence grows and measure development through autonomy, decision quality, communication, and impact.

### 104. How do you respond when a release deadline is at risk?

**Short answer:** Surface the risk early, explain evidence and impact, identify the critical path, reduce scope or sequence safely, obtain missing decisions, and preserve essential quality and security.

Do not create false confidence or silently require sustained overtime. After delivery, examine why the risk surfaced late and improve planning, ownership, or technical foundations.

### 105. What makes someone a senior iOS engineer?

**Short answer:** A senior engineer independently owns meaningful outcomes, makes sound technical tradeoffs, delivers reliable systems, improves team capability, communicates across functions, and handles ambiguity.

Seniority is not the number of frameworks memorized. It appears in judgment: choosing appropriate complexity, preventing recurring problems, mentoring others, understanding product impact, and taking responsibility from discovery through production operation.

### 106. How should you present a project in an interview?

Use this structure:

```text
Problem and users
Your role and team
Constraints
Architecture and key decisions
Hardest technical problem
Tradeoffs and alternatives
Testing and observability
Measured outcome
What failed or changed
What you would improve now
```

Be specific about your contribution without erasing the team. Interviewers learn more from one deeply understood project than from a list of technologies.

## Practical Coding Questions to Practice

Prepare to implement and discuss:

- a generic thread-safe or actor-isolated cache;
- a debounced search feature with cancellation;
- paginated loading with duplicate-request prevention;
- a type-safe API client;
- token refresh with one in-flight operation;
- an image loader with caching and reuse safety;
- a view model with loading, empty, content, and error states;
- decoding polymorphic or imperfect JSON;
- a coordinator or typed SwiftUI route;
- an LRU cache;
- a producer-consumer flow using `AsyncSequence`;
- unit tests for asynchronous success, failure, and cancellation.

During coding, explain complexity, ownership, edge cases, cancellation, error behavior, and how you would test the solution.

## Common Interview Mistakes

- Giving definitions without a practical example.
- Calling every reference `weak` without explaining ownership.
- Saying async code always runs on a background thread.
- Treating actors as locks that make an entire workflow atomic.
- Doing networking or parsing directly inside a view.
- Naming architecture patterns without explaining why they help.
- Storing tokens in UserDefaults.
- Retrying every failed request, including non-idempotent operations.
- Using fixed sleeps in asynchronous tests.
- Optimizing performance without measurement.
- Claiming individual ownership of team results.
- Hiding a failed decision instead of explaining what changed afterward.

## Final Preparation Checklist

- Explain Swift value and reference semantics with code.
- Diagnose and fix a retain cycle.
- Explain task hierarchy, actor isolation, reentrancy, and cancellation.
- Describe UIKit and scene lifecycle behavior.
- Choose SwiftUI state ownership correctly.
- Design a testable feature using dependency injection.
- Build a secure, cancellable API request flow.
- Explain persistence and Keychain choices.
- Write deterministic async tests.
- Use Instruments to diagnose a measured problem.
- Design one offline or media-heavy feature end to end.
- Prepare two production-debugging stories.
- Prepare one disagreement, mentoring, and deadline-risk story.
- Present one project deeply, including tradeoffs and measured results.

The best iOS interview answers connect language rules to product behavior. Explain not only what an API does, but how ownership, lifecycle, concurrency, persistence, networking, testing, security, and user experience interact in a production application.

## Official References

- [The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/)
- [Swift Concurrency](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)
- [Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- [UIKit app lifecycle](https://developer.apple.com/documentation/uikit/managing-your-app-s-life-cycle)
- [UIKit development overview](https://developer.apple.com/documentation/uikit/about-app-development-with-uikit)
- [SwiftUI model data](https://developer.apple.com/documentation/swiftui/model-data)
- [URLSession](https://developer.apple.com/documentation/foundation/urlsession)
- [XCTest](https://developer.apple.com/documentation/xctest)
- [Testing asynchronous code](https://developer.apple.com/documentation/testing/testing-asynchronous-code)
