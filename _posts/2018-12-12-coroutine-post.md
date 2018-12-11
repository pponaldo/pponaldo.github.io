---
title: "Coroutines in Kotlin"
date: 2018-12-12 01:26:00 -0400
categories: development
---

**Coroutines in Kotlin**

오랜 시간이 소요되는 작업을 main thread에 blocking없이 수행하는 한가지 방법은 callback이다. callback 패턴으로 장시간 걸리는 작업을 background thread에서 시작할 수 있다. 해당 작업이 끝날 때 그 callback이 호출되어서 작업의 결과를 main thread에 알려준다.

```
// Slow request with callbacks
@UiThread
fun makeNetworkRequest() {
    // The slow network request runs on another thread
    slowFetch { result ->
        // When the result is ready, this callback will get the result
        show(result)
    }
    // makeNetworkRequest() exits after calling slowFetch without waiting for the result
}
```

*Callback은 좋은 패턴이지만 몇몇 단점이 있다. callback을 과도하게 사용한 코드의 경우 코드를 읽기가 어렵고 추론하기가 어려울 수 있다. 그리고 콜백은 exceptions과 같은 언어 기능의 사용을 허용하지 않는다.*

Kotlin coroutines은 callback-based 코드를 sequential코드로 변경시켜 준다. sequential한 코드는 일반적으로 읽기도 더 쉽고 exceptions같은 언어의 기능들도 사용할 수 있다.


**suspend** 키워드는 function이나 function type이 coroutines에 사용가능하도록 해주는 kotlin의 방식이다. coroutine이 suspend로 표시된 function을 호출할 때, 일반적은 function call처럼 function이 return할때까지 blocking하는 대신에 결과가 준비 될때까지 실행을 미루다가 준비되면 결과와 함게 중단 된 부분부터 다시 시작한다.
결과를 기다리며 중단되는 동안 thread를 block하지 않고 다른 function이나 coroutines을 실행할 수 있다.

```
// Slow request with coroutines
@UiThread
suspend fun makeNetworkRequest() {
    // slowFetch is another suspend function so instead of 
    // blocking the main thread  makeNetworkRequest will `suspend` until the result is 
    // ready
    val result = slowFetch()
    // continue to execute after the result is ready
    show(result)
}

suspend fun slowFetch(): SlowResult { ... }
```
