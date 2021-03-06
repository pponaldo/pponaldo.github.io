---
title: "Coroutines in Kotlin"
date: 2018-12-12 01:26:00 -0400
categories: development
---

Coroutines in Kotlin
====================
_[Coroutine Codelab](https://codelabs.developers.google.com/codelabs/kotlin-coroutines)
을 기반으로 개인 학습을 위한 부분적인 번역 및 학습내용을 기록합니다._

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


<br>
<br>
<br>
Kotlin에서 모든 coroutines은 CoroutineScope 내에서 실행된다. Scope는 그것의 Job을 통해서 coroutines의 lifetime을 컨트롤한다. 특정 scope의 job을 취소할 때, 그것은 scope안에서 시작 된 모든 coroutines를 취소한다. 안드로이드 상에서 모든 실행 중인 coroutines을 취소하기 위해서 scope를 사용할 수 있다. 예를들어 사용자가 activity 또는 fragment에서 벗어나는 경우

또한 Scope는 default dispatcher 명시를 허용한다. dispatcher는 어떤 thread가 coroutines을 운영할지 컨트롤 할 수 있다.
```
private val viewModelJob = Job()

private val uiScope = CoroutineScope(Dispatchers.Main + viewModelJob)
```

uiScope는 안드로이드 상의 main thread인 Dispatcher.Main에서 coroutines을 시작할 것이다.
main thread에서 시작 된 coroutine은 suspended 동안 main thread를 block하지 않을 것이다.
coroutine은 시작 된 후에 언제든지 dispatcher들을 전환 할 수있다. 예를 들면 한 coroutine이 main dispatcher 위에서 시작될 수 있고 main thread의 결과인 large JSON을 parsing하기 위해서 또다른 dispatcher를 사용할 수 있다.

- scope에서 시작 된 모든 coroutines를 취소하기 위해서는 CoroutineScope에 Job을 설정해야 한다.

- CoroutineScope contructor로 생성 된 scopes는 암시적인 job을 추가하는데 이는 uiScope.coroutineContext.cancel()로 취소할 수 있다.


<br>
<br>
## Converting existing callback API with coroutines

- callback pattern
<br>
아래 코드에서 fetchNewWelcome function은 FakeNetworkCall을 리턴한다. 호출되면 network요청을 다른 thread에서 수행하고 그 결과를 addOnResultListener로 등록 된 listener에 전달한다.
```
fun refreshTitle(/* ... */) {
   val call = network.fetchNewWelcome()
   call.addOnResultListener { result ->
       // callback called when network request completes or errors
       when (result) {
           is FakeNetworkSuccess<String> -> {
               // process successful result
           }
           is FakeNetworkError -> {
               // process network error
           }
       }
   }
}
```

- coroutines
suspend function extension *await()* 만들기
<br>extension function을 만들어서 FakeNetworkCall Class 원본에는 수정없이 coroutine을 지원하는 await function을 만들수 있다.


suspendCoroutine호출은 즉각적으로 현재의 coroutine을 suspend하고 해당 coroutine을 resume할 수 있는 *continuation* object를 줄 것이다. continuation은 suspended coroutine을 continue, resume을 위해 필요로하는 모든 context를 가지고있다.

>suspendCoroutine은 cacellation을 지원하지 않아서 cancellation을 전달을 지원하기 위해서는 suspendCancellationCoroutine을 사용해야한다.
```
suspend fun <T> FakeNetworkCall<T>.await(): T {
   return suspendCoroutine { continuation ->
       addOnResultListener { result ->
           when (result) {
               is FakeNetworkSuccess<T> -> continuation.resume(result.data)
               is FakeNetworkError -> continuation.resumeWithException(result.error)
           }
       }
   }
}
```

아래와 같이 await function을 호출해서 sequential하게 코드를 작성할 수 있다.

```
// Example usage of await

suspend fun exampleAwaitUsage() {
   try {
       val call = network.fetchNewWelcome()
       // suspend until fetchNewWelcome returns a result or throws an error
       val result = call.await()
       // resume will cause await to return the network result
   } catch (error: FakeNetworkException) {
       // resumeWithException will cause await to throw the error
   }
}
```

## Testing Coroutine Directly

await function의 unit test를 진행하면 아래와 같은 compile error를 만나게 된다.
```
@Test(expected = FakeNetworkException::class)
fun whenFakeNetworkCallFailure_throws() {
    val subject = makeFailureCall(FakeNetworkException("the error"))

    subject.await() // Compiler error: Can't call outside of coroutine
}
```
await function은 suspend funtion이므로 일반적인 Kotlin에서는 호출 할 수 없기 때문이다.
따라서 테스트를 위해서는 CoroutineScope를 사용해서 coroutine을 launch해야하지만
위의 test function whenFakeNetworkCallFailure_throws은 바로 return을 하고 exception을 캐치할 수 없기 때문에 항상 fail이다.

이를 위해서 kotlin은 suspend function이 호출되는 동안 block할 수 있는 runBlocking이 존재한다. 이 호출을 통해서 suspend function을 일반적인 function처럼 block할 수 있다.

따라서 아래와 같이 runBlocking호출로 unit test를 할 수 있다.
```
@Test(expected = FakeNetworkException::class)
fun whenFakeNetworkCallFailure_throws() {
   val subject = makeFailureCall(FakeNetworkException("the error"))

   runBlocking {
       subject.await()
   }
}
```

...ing

## Using Couroutines on a worker thread


## Using coroutines in higher order functions


## Using coroutines with WorkManager
