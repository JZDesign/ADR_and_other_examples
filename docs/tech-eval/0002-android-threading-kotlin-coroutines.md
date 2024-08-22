# 2. Android - Threading - Kotlin Coroutines

Date: 2020-03-25

## Status

Evaluation Started

## Problem Description

In order to keep our UI feeling responsive, we need to offload as much work as possible off of the main UI thread and onto background threads. This also helps to prevent [_Application Not Responding_ errors](https://developer.android.com/training/articles/perf-anr).

## What We Are Evaluating

[Kotlin Coroutines](https://kotlinlang.org/docs/reference/coroutines-overview.html)

* For an Android-specific introduction check out the [Android Kotlin Coroutines Codelab](https://codelabs.developers.google.com/codelabs/kotlin-coroutines/)

## Alternatives

* [RxJava](https://github.com/ReactiveX/RxJava) was previously popular on Android for managing foreground/background work.
* [AsyncTask](https://developer.android.com/topic/performance/threads) is the classic way to do things in the background.

## Evaluation Plan

### Acceptance Criteria

* Must allow us to easily shift work to background threads or foreground threads.
* Should minimize impact to the interfaces between layers. For example, we don't want every public method to have to accept or return some particular framework class.
* Should feel natural to the Android and Kotlin ecosystem.

### Duration of Evaluation

We will evaluate this throughout the _Operation: Red Pants_ epic, where we are laying the groundwork for v2.

* **Start Date:** March 25, 2020
* **Expiration Date:** December 31, 2020

### Roll-forward Plan

We will use coroutines to manage threading from the start, so rolling forward just means continuing to use it.

### Roll-backward Plan

Rolling backward or to another solution would involve reimplementing the parts that put work onto foreground or background threads. As we've been exploring this in [DEBIT-1668](https://github.com/lampo/debit-mobile-app/pull/38), we tucked that work behind a single function, so rolling to another solution would mostly involve just be reimplementing that piece, plus any similar changes required for testing code.

## Evaluation Log

### 2020-03-25

Evaluation Started

Testing coroutines does involve some added work, much of which can be partly abstracted. There's a [good video](https://www.youtube.com/watch?v=KMb0Fs8rCRs) about testing with coroutines, which covers some API design concepts also. One of the main challenges in testing with coroutines is how to make sure that they've all run to completion before running verifications and assertions.  By using `kotlinx-coroutines-test`, you can set the `Dispatchers.Main` to use a `TestCoroutineDispatcher`, which helps make sure that coroutines running on `Dispatchers.Main` will happen synchronously. But it doesn't apply to `Dispatchers.IO` or `Dispatchers.Default`.

 As we were working on DEBIT-1668, we explored a few different approaches to solving this problem.

#### 1. Injection

First, we tried injecting the a configuration holding the background thread dispatcher into the presenter. This required that all presenters extended an `AbstractPresenter` and passed along the dispatcher, like this:

```kotlin
public class GreetingPresenter(
    private val view: GreetingView,
    private val service: GreetingService,
    dispatcherConfig: DispatcherConfig
) : AbstractPresenter(dispatcherConfig) {
    // ...
}
```

And then anytime work was going onto a background thread, instead of `launch(Dispatchers.IO)`, you'd use `launch(dispatcherConfig.background)`.

This was a recommendation in the video mentioned above. But it also means all of our presenters (and other components that use coroutines for background threads) would need to change their constructor. Not a horrible impact, but does require a little more juggling.

#### 2. Using Waits for Assertions

We also tried just leaving the coroutine builders using `Dispatchers.IO`, but changing our test code to use waits - where the assertion would keep trying until the assertion was true or a timeout was reached.

This changed our test code to look like this, where we had to add the `timeout` argument.

```kotlin
verify(timeout = 5000) { view.showGreeting(expectedGreeting) }
```

Also, the assertion libraries we're using don't provide any waiting tools yet, so we'd either have to build that out, or always place our assertions after our blocking mock verifications, in order to make sure the operations completed before proceeding. And even then, depending on timing, it'd be possible for a test to fail.

Although this approach had no impact to production code, it would require some changes to the test code, and we'd have to remember to call certain versions of `verify()` or `assertThat()` when testing coroutines.

#### 3. Mutable Singleton for Dispatchers

We also tried creating our own singleton `object` to hold the dispatchers for foreground and background work, like this:

```kotlin
object CoroutineConfig {
    lateinit var Background: CoroutineDispatcher
    lateinit var Foreground: CoroutineDispatcher

    init {
        reset()
    }

    fun reset() {
        Background = Dispatchers.Default
        Foreground = Dispatchers.Main
    }
}
```

For this approach, instead of `launch(Dispatchers.IO)`, we use `launch(CoroutineConfig.Background)`. Since those two properties (`Background` and `Foreground`) are reassignable, the test code just assigns `TestCoroutineDispatcher` to both of them, causing everything to run deterministically and synchronously.

Overall, this had minimal impact to our production code (just using `CoroutineConfig` instead of `Dispatchers`), and the bodies of our test functions can look exactly like they would if it were testing normal synchronous code. The main disadvantage is that it's global mutable state, and there's no way to truly prevent a developer from switching out the dispatcher at runtime in production code.

We're going to continue trying out this third option and see how it scales as we add more coroutines to the app.
