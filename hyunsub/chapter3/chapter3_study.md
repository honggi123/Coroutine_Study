### 코루틴의 Continuation 은 무엇인가?

**Continuation Passing Style (CPS) 개념:**

- **CPS 방식**에서는 함수의 호출 결과를 직접 호출자에게 넘기지 않고, Continuation 객체(State Machine)에 넘기면서 호출 순서를 관리한다. 하위 함수를 순차적으로 호출하여 비동기 루틴을 처리한다.

**코루틴의 동작:**

- 코틀린 코루틴은 내부적으로 CPS로 변환되어 실행된다.
- **예제 코드**에서 `suspend fun postItem(item: Item)` 함수는 비동기적으로 세 개의 하위 루틴을 호출한다.
- *State Machine (sm)**을 사용하여 코루틴의 상태와 결과를 관리한다. sm 객체는 각 호출 시 스택 프레임을 복사하고 저장하여 연속성을 유지한다.

**Continuation 인터페이스:**

```kotlin
kotlin코드 복사
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}

```

- Continuation 객체는 코루틴의 실행을 재개하고 결과를 전달하는 역할이다.

**실제 사용 예시:**

1. **클릭 리스너를 사용한 코루틴:**

    ```kotlin
    kotlin코드 복사
    suspend fun syncClick(): String = suspendCoroutine { cont ->
        btnEnter.setOnClickListener {
            cont.resumeWith(Result.success("Clicked"))
            btnEnter.setOnClickListener(null)
        }
    }
    
    lifecycleScope.launch {
        val result = syncClick()
        result // Clicked
    }
    
    ```

2. **뷰의 크기 확정 시점을 알기 위한 콜백 루틴:**

    ```kotlin
    kotlin코드 복사
    suspend fun View.awaitNextLayout() = suspendCancellableCoroutine<Unit> { cont ->
        val listener = object : View.OnLayoutChangeListener {
            override fun onLayoutChange(...) {
                view?.removeOnLayoutChangeListener(this)
                cont.resume(Unit)
            }
        }
        cont.invokeOnCancellation { removeOnLayoutChangeListener(listener) }
        addOnLayoutChangeListener(listener)
    }
    
    viewLifecycleOwner.lifecycleScope.launch {
        titleView.awaitNextLayout()
        // 뷰의 크기가 확정되면 작업을 수행...
    }
    
    ```


**결론:**

- **suspendCancellableCoroutine**을 사용하여 콜백 스타일 API를 코루틴으로 감싸면 코드가 간결해지고 관리하기 쉬워진다.
- Continuation을 명시적으로 사용하면 비동기 작업을 순차적으로 처리하는 데 많은 도움이 된다.

### 아래의 코드에서 After가 Resumed보다 먼저 출력되는 이유?

```kotlin
suspend fun main() {
	println("Before")

	suspendCoroutine<Unit> { continuation ->
		thread {
			println("Suspended")
			Thread.sleep(1000)
			continuation.resume(Unit)
			println("Resumed")
		}
	}

	println("After")

}
// Before
// Suspended
// (1초 후)
// After
// Resumed
```

- suspendCoroutine이 일시 중단된 후에도 현재 코루틴의 실행을 계속하기 때문이다. 따라서 suspendCoroutine의 람다가 완료된 후 "After"가 출력된다.

### 참고

https://ogoons.com/coroutine-continuation