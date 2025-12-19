# DashboardFeature – TCA Learning README

This document captures **everything learned while building the `DashboardFeature`** using **The Composable Architecture (TCA)** with SwiftUI.

The goal of this project was **not** to build UI polish, but to deeply understand **architecture, composition, debugging, and mental models** required for real-world TCA apps.

---

## 🎯 Project Overview

The Dashboard feature is a **parent feature** that composes two independent child features:

* `CounterFeature`
* `StopWatchFeature`

```
DashboardFeature
 ├── CounterFeature
 └── StopWatchFeature
```

All three features are **fully isolated, testable, and composable**.

---

## 🧠 Core TCA Concepts Learned

### 1. Unidirectional Data Flow

Every interaction follows this strict flow:

```
View → Action → Reducer → State → View
```

* Views **never mutate state directly**
* Reducers are the **only place** state changes
* Effects are **explicit and cancellable**

This guarantees predictability and debuggability.

---

### 2. Parent → Child Reducer Composition

The `DashboardFeature` owns the state of its children and routes their actions.

```swift
@Reducer
struct DashboardFeature {

    @ObservableState
    struct State: Equatable {
        var counter = CounterFeature.State()
        var stopwatch = StopWatchFeature.State()
    }

    enum Action {
        case counter(CounterFeature.Action)
        case stopwatch(StopWatchFeature.Action)
    }
}
```

#### Key rules learned:

* Parent **owns child state**
* Parent **routes child actions**
* Child reducers run **independently**
* Parent may **react to child actions**, but children never know about parents

---

### 3. `Scope` – The Most Important TCA Primitive

`Scope` wires child reducers into the parent reducer:

```swift
Scope(state: \.counter, action: \.counter) {
    CounterFeature()
}
```

What `Scope` guarantees:

* Correct state routing
* Correct action routing
* Isolation between features
* Reusability of child features

---

## 🧩 SwiftUI Integration (Correct Mental Model)

### 4. Store Scoping in Views

Views receive **only the state and actions they need**:

```swift
CounterView(
    store: store.scope(
        state: \.counter,
        action: \.counter
    )
)
```

This enforces:

* View boundaries
* Compile-time safety
* No accidental state access

---

### 5. Observation vs Passing Stores

A critical insight:

> **Passing a store ≠ observing state**

In `DashboardView`:

* The parent does NOT observe state by default
* Child views observe their own scoped stores

This prevents unnecessary parent re-renders.

---

## ⏱️ Async Effects & Cancellation (StopWatchFeature)

### 6. Long-Living Effects

The stopwatch introduced:

* Async effects using `Clock`
* Infinite timers
* Effect cancellation via IDs

```swift
for await _ in clock.timer(interval: .seconds(1)) {
    await send(.timerTicked)
}
```

### Key rule learned:

> **Every long-living effect must be cancellable**

---

### 7. Effect Cancellation

```swift
.cancellable(id: TimerID.self)
```

And cancellation triggered by another action:

```swift
return .cancel(id: TimerID.self)
```

This is **mandatory** for:

* Timers
* Streams
* Notifications
* Location updates

---

## 🧪 Testing Learnings

### 8. Tests Must Cancel Long-Living Effects

A major lesson:

> If a reducer starts a long-living effect, the test **must stop it**

Otherwise:

```
An effect returned for this action is still running
```

Correct pattern:

```swift
await store.send(.startTapped)
await clock.advance(by: .seconds(3))
await store.send(.stopTapped)
```

---

## 🪵 Debugging – The Big Learning Area

### 9. Debugging Is NOT One Thing

We learned there are **two separate debugging systems**:

| Area    | Tool                      | Purpose                 |
| ------- | ------------------------- | ----------------------- |
| Reducer | Logging / debug reducers  | Actions, state, effects |
| View    | SwiftUI `_printChanges()` | Re-render causes        |

Mixing them causes confusion.

---

### 10. Why `_printChanges()` "Did Nothing"

Key insight:

> `_printChanges()` only prints when the view **re-renders**

If a view does not observe state, it will not re-render, and nothing prints.

This is **correct SwiftUI behavior**, not a bug.

---

### 11. Why `.debug()` Was Not Available

We learned:

* `.debug()` depends on TCA version
* Macro-based reducers ≠ debug availability

This led to a **version-safe solution**.

---

## 🧰 Custom LoggingReducer (Version-Safe Debugging)

### 12. Building Our Own Tooling

Instead of relying on framework APIs, we built our own:

```swift
struct LoggingReducer<R: Reducer>: Reducer {
    let reducer: R
    let label: String

    func reduce(into state: inout R.State, action: R.Action) -> Effect<R.Action> {
        print("🔹 \(label) action:", action)
        let effect = reducer.reduce(into: &state, action: action)
        print("🔸 \(label) state:", state)
        return effect
    }
}
```

This gave:

* Full control
* Version independence
* Clear mental model

---

### 13. Reducing Verbosity with Extensions

To avoid boilerplate, we introduced:

```swift
extension Reducer {
    func debugged(_ label: String) -> some Reducer<State, Action> {
        LoggingReducer(reducer: self, label: label)
    }
}
```

Usage:

```swift
StopWatchFeature()
    .debugged("⏱️ Stopwatch")
```

This balances:

* Readability
* Power
* Maintainability

---

## 🧠 Architectural Principles Reinforced

### 14. Separation of Concerns

* Views render
* Reducers decide
* Effects do async work
* Debugging lives outside features

---

### 15. Composition Over Inheritance

* Features compose other features
* No base classes
* No shared mutable state

---

### 16. Build Tools When APIs Don’t Exist

Instead of waiting for framework support, we learned to:

* Build small, focused utilities
* Keep them optional
* Remove them easily

This is **senior-level engineering practice**.

---

## 🚀 Where This Leaves Us

After this project, we are comfortable with:

* Multi-feature TCA apps
* Parent–child coordination
* Async effects and cancellation
* Deterministic testing
* Professional debugging strategies

This is the foundation used in **large production TCA apps**.

---

## 🔜 Recommended Next Steps

1. Navigation with `StackState` and `PresentationState`
2. API-driven features (loading, error, retry)
3. App-wide dependency layering
4. Performance tuning & observation control
5. Full-feature Tic-Tac-Toe or real app clone

---

> **This README represents architectural understanding, not just code.**
> Keep it close — it will save you months later.
