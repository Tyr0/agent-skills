---
name: swift-testing
description: Use this skill whenever the user asks about testing in Swift, including the Swift Testing framework (@Suite, @Test, #expect, #require), parameterized tests, async test support, test organization with tags, or migrating from XCTest to Swift Testing. Also use it for questions about writing unit tests, integration tests, mocking dependencies, test coverage, or test performance in Swift. Triggers on 'how do I write a Swift test', 'what is @Test', 'how does #expect work', 'how do I test async code', 'Swift Testing vs XCTest', or any question about testing Swift code.
---

# Swift Testing Reference

A dense reference for the Swift Testing framework (Swift 5.9+, Xcode 16+) — the modern replacement for XCTest.

> **Requires:** Swift 5.9+ / Xcode 16+. Import `Testing`. XCTest remains available; the two frameworks can coexist in the same target.

---

## Core Concepts

| Concept | Swift Testing | XCTest |
|---|---|---|
| Test function | `@Test func name()` | `func testName()` |
| Test class/struct | `@Suite struct Name` | `class Name: XCTestCase` |
| Assertion | `#expect(expr)` | `XCTAssert(expr)` |
| Required assertion | `#require(expr)` | `XCTUnwrap(expr)` / `XCTAssertNotNil` |
| Fatal assertion | `#require` throws on failure | `XCTFail()` |
| Setup | `init()` | `setUp()` |
| Teardown | `deinit` | `tearDown()` |
| Skipping | `#expect throws:` or `withKnownIssue` | `XCTSkip` |

---

## Basic Test Structure

```swift
import Testing

@Suite("User authentication")
struct AuthTests {
    let service: AuthService

    init() {
        service = AuthService(database: .inMemory)
    }

    @Test("Valid credentials return a session token")
    func loginWithValidCredentials() throws {
        let token = try service.login(email: "user@example.com", password: "correct")
        #expect(token.isNotEmpty)
    }

    @Test("Invalid password throws AuthError.invalidCredentials")
    func loginWithInvalidPassword() throws {
        #expect(throws: AuthError.invalidCredentials) {
            try service.login(email: "user@example.com", password: "wrong")
        }
    }
}
```

- Suites can be `struct`, `class`, or `actor` — `struct` is idiomatic (value semantics, fresh instance per test).
- Every `@Test` function gets a **fresh instance** of the suite — no shared mutable state between tests.
- `init()` runs before each test; `deinit` runs after (use for resource cleanup).

---

## `#expect` — Non-Fatal Assertions

`#expect` records a failure but continues test execution. Use for independent assertions where you want to see all failures.

```swift
#expect(value == 42)
#expect(array.isEmpty)
#expect(string.hasPrefix("Hello"))
#expect(optionalValue != nil)

// Custom failure message
#expect(result == expected, "Expected \(expected) but got \(result)")

// Assertion on thrown errors
#expect(throws: MyError.specificCase) {
    try riskyOperation()
}

// Assert no error is thrown
#expect(throws: Never.self) {
    try safeOperation()
}
```

Failure output includes the full expression tree — no need to write custom messages for most cases:

```
Expectation failed: (value → 41) == 42
```

---

## `#require` — Fatal Assertions

`#require` throws on failure, stopping the test immediately. Use when subsequent assertions depend on this value.

```swift
@Test func tokenIsValid() throws {
    let response = try fetchToken()

    // Stop if token is nil — no point continuing
    let token = try #require(response.token)

    // These only run if token was non-nil
    #expect(token.expiresAt > Date.now)
    #expect(token.value.count == 32)
}
```

`#require` also works as an unwrapping operator for optionals:

```swift
let user = try #require(findUser(id: 42))   // throws if nil
#expect(user.name == "Alice")
```

---

## Async Tests

Mark the test `async` — Swift Testing handles the execution context automatically:

```swift
@Test("Fetches user from network")
func fetchUser() async throws {
    let user = try await userService.fetch(id: 42)
    #expect(user.name == "Alice")
}
```

Async suites with shared async state:

```swift
@Suite
actor DatabaseTests {
    var db: Database!

    init() async throws {
        db = try await Database.inMemory()
        try await db.migrate()
    }

    @Test func insertsRecord() async throws {
        try await db.insert(User(name: "Alice"))
        let count = try await db.count(User.self)
        #expect(count == 1)
    }
}
```

Use `actor` suite when multiple tests share async mutable state and need protection from data races.

---

## Parameterized Tests

Run a single test with multiple inputs. Eliminates copy-paste test duplication.

```swift
@Test("Validates email format", arguments: [
    "valid@example.com",
    "also.valid+tag@sub.domain.com",
])
func validEmailFormats(email: String) {
    #expect(EmailValidator.isValid(email))
}

@Test("Rejects malformed emails", arguments: [
    "",
    "no-at-sign",
    "@no-local-part.com",
    "spaces in@email.com",
])
func invalidEmailFormats(email: String) {
    #expect(!EmailValidator.isValid(email))
}
```

### Two-argument parameterized tests

```swift
@Test("Converts units correctly", arguments: [
    (1.0, UnitLength.meters, 100.0, UnitLength.centimeters),
    (1.0, UnitLength.kilometers, 1000.0, UnitLength.meters),
])
func unitConversion(value: Double, from: UnitLength, expected: Double, to: UnitLength) {
    let result = Measurement(value: value, unit: from).converted(to: to).value
    #expect(result == expected)
}
```

### Zip vs product

```swift
// zip: pairs are (inputs[0], outputs[0]), (inputs[1], outputs[1]) — same count required
@Test(arguments: zip(inputs, expectedOutputs))
func pairedTest(input: String, expected: String) { ... }

// product (default with two collections): all combinations
@Test(arguments: operators, operands)
func allCombinations(op: String, n: Int) { ... }
```

---

## Tags

Tags group tests across suites and enable filtering in Xcode or `swift test`.

```swift
extension Tag {
    @Tag static var networking: Self
    @Tag static var slow: Self
    @Tag static var critical: Self
}

@Suite(.tags(.networking))
struct NetworkTests {
    @Test(.tags(.slow)) func largeDownload() { ... }
    @Test(.tags(.critical)) func authFlow() { ... }
}
```

Run tagged tests:

```bash
swift test --filter .tags(.networking)
```

---

## Known Issues / Expected Failures

```swift
@Test("This known bug is being tracked in #1234")
func knownBug() {
    withKnownIssue {
        #expect(buggyFunction() == expected)  // expected failure; test passes
    }
}
```

If the issue is fixed and the test starts passing, `withKnownIssue` reports an unexpected pass — prompting you to remove it.

---

## Traits

Traits customize how tests run:

```swift
@Test(.disabled("Not implemented yet"))
func futureFeature() { }

@Test(.timeLimit(.minutes(1)))
func networkRequest() async { }

@Suite(.serialized)  // run tests in this suite serially (not in parallel)
struct OrderDependentTests { }
```

---

## Dependency Injection and Mocking

Inject dependencies through `init()` — no global mutable state needed:

```swift
protocol HTTPClient {
    func fetch(_ url: URL) async throws -> Data
}

struct MockHTTPClient: HTTPClient {
    let response: Data
    func fetch(_ url: URL) async throws -> Data { response }
}

@Suite
struct WeatherServiceTests {
    let service: WeatherService

    init() {
        service = WeatherService(client: MockHTTPClient(response: mockWeatherJSON))
    }

    @Test func parsesCurrentTemperature() async throws {
        let weather = try await service.current(for: "Boulder, CO")
        #expect(weather.temperature == 18.5)
    }
}
```

---

## XCTest Migration

You do not need to migrate all at once — both frameworks coexist in the same target.

| XCTest | Swift Testing |
|---|---|
| `class MyTests: XCTestCase` | `@Suite struct MyTests` |
| `func testFoo()` | `@Test func foo()` |
| `XCTAssertEqual(a, b)` | `#expect(a == b)` |
| `XCTAssertNil(x)` | `#expect(x == nil)` |
| `XCTAssertNotNil(x)` | `#expect(x != nil)` or `try #require(x)` |
| `XCTAssertThrowsError(try f())` | `#expect(throws: Error.self) { try f() }` |
| `XCTUnwrap(optional)` | `try #require(optional)` |
| `XCTSkip("reason")` | `try #require(Bool(false), "reason")` or `.disabled` trait |
| `setUp()` | `init()` |
| `tearDown()` | `deinit` |
| `XCTExpectFailure` | `withKnownIssue { }` |

**Do not mix XCTest and Swift Testing in the same test class/suite.** Keep them in separate files.

`@MainActor` is not needed on Swift Testing suites — the framework handles concurrency correctly without it.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Using `XCTAssert*` in `@Test` functions | XCTest assertions don't integrate with Swift Testing's failure model | Use `#expect` / `#require` exclusively in `@Test` functions |
| Shared mutable state between tests via `static var` | Tests become order-dependent; parallel execution causes races | Use `struct` suites — each test gets a fresh instance |
| `#expect` for nil-unwrapping before dependent assertions | Test continues with a nil crash instead of clean failure message | Use `try #require(optional)` to stop early on nil |
| No `@Suite` grouping | Tests are hard to filter and navigate | Group related tests into named suites |
| `@MainActor` on an entire `@Suite` | Serializes all tests on the main actor unnecessarily | Only annotate specific `@Test` functions that touch MainActor state |
| Parameterized test with one argument | Adds boilerplate for no gain | Use a regular `@Test` with one hardcoded value; parameterize when there are 3+ cases |
| Keeping dead XCTest alongside migrated Swift Testing | Two frameworks, double maintenance | Migrate file-by-file; delete XCTest file when Swift Testing equivalent is complete |
