## 코루틴의 실제 구현 

해당 장에선 코루틴의 내부 구현을 주로 설명합니다.

- 중단 함수는 함수가 시작할 때와 중단 함수가 호출되었을 때 상태를 가진다는 점에서 상태 머신과 비슷합니다.
- 컨티뉴에이션(continuation) 객체는 상태를 나타내는 숫자와 로컬 데이터를 가지고 있습니다.
- 함수의 컨티뉴에이션 객체가 이 함수를 부르는 다른 함수의 컨티뉴에이션 객체를 장식(decorate)합니다. 그 결과,
컨티뉴에이션 객체는 실행을 재개하거나 재개된 함수를 완료할 때 사용되는 콜 스택으로 사용됩니다.

코루틴은 중단 함수 구현을 위해 컨티뉴에이션 전달 방식(contiuation-passing style)을 사용합니다. 컨티뉴에이션은 함수에서 함수로
인자를 통해 전달됩니다.

```
suspend fun setUser(user: User)
// 자세히 들여다 보면
fun setUser(user: User, contiuation: Contiuation<*>): Any
```

중단 함수 내부를 들여다 보면 원래 선언했던 형태와 반환 타입이 달라졌다는걸 알 수 있습니다. 이는 중단 함수를 실행하는 도중에 중단되면 선언된
타입의 값을 반환하지 않을수있기 때문입니다. </br>
이때 중단 함수는 COROUTINE_SUSPENDED를 반환합니다. 지금은 getUser 함수가 User? 또는 COROUTINE_SUSPENDED를 반환할 수 있기 때문에 결과 타입이 User?와 Any의 가장 가까운 슈퍼타입인 Any?로 지정되었다는 것만 확인하면 됩니다.

### 아주 간단한 함수

```
suspend fun myFunction(){
  println("Before")
  delay(1000) // 중단 함수
  println("After")
}
// 디컴파일 시
fun myFunction(contiuation: Contiuation<*>): Any

이 함수는 상태를 저장하기 위해 자신만의 컨티뉴에이션 객체가 필요하다는 것입니다. 이를 MyFunctioinContinuation이라 하겠스빈다. 
본체가 시작될 때 MyFunction은 파라미터인 continuation을 자산만의 컨티뉴에이션인 MyFunctionContiuation으로 
포장합니다.

val continuation = contiuation as ? MyFunctionContiuation
      ?: MyFunctionContiuation(contiuation)

suspend fun myFunction(){
  println("Before")
  delay(1000)
  println("After")
}
```

함수가 시작되는 지점은 함수의 시작점(함수가 처음 호출될 때)과 중단 이후 재개 지점(컨티뉴에이션이 resume을 호출할 때)
e두 곳입니다. 현재 상태를 저장라혀면 label이라는 필드를 사용합니다. 함수가 처음 시작될 때 이 값은 0으로 설정됩니다.
이후에는 중단되기 전에 다음 상태로 설정되어 코루틴이 재개될 시점을 알 수 있게 도와줍니다.

```
// myFuction의 세부구현을 간단하게 표현하면 다음과 같습니다.
fun myFunction(contiuation: Contiuation<Unit>): Any { 
  val continuation = contiuation as? MyFunctionContiuation
      ?: MyFunctionContiuation(contiuation)

  if (continuation.label == 0){
    println("Before")
    continuation.label = 1
    if(delay(1000, continuation) == COROUTINE_SUSPENDED){
        return COROUTINE_SUSPENDED
    }
  }
  if (continuation.label == 1) {
    println("After")
    return Unit
  }
}
```


delay에 의해 중단된 경우 COROUTINE_SUSPENDED가 반환되며, myFunction은 COROUTINE_SUSPENDED를 
반환합니다. myFunction을 호출한 함수부터 시작해 콜 스택에 있는 모든 함수도 똑같습니다. 따라서 중단이 일어나면
콜 스택에 있는 모든 함수가 종료되며, 중단된 코루틴을 실행하던 스레드를 (다른 종류의 코루틴을 포함해) 실행 가능한 코드가
사용할 수 있게 됩니다.

#### 지금까지 설계한 함수를 간략화한 최종 모습

```
fun myFunction(contiuation: Contiuation<Unit>): Any { 
  val continuation = contiuation as? MyFunctionContiuation
      ?: MyFunctionContiuation(contiuation)

  if (continuation.label == 0){
    println("Before")
    continuation.label = 1
    if(delay(1000, continuation) == COROUTINE_SUSPENDED){
        return COROUTINE_SUSPENDED
    }
  }
  if (continuation.label == 1) {
    println("After")
    return Unit
  }
  error("Impossible")
}

class MyFunctionContiuation(
  val completion: Continuation<Unit>
) : Continuation<Unit> {
  override val context: CoroutineContext
    get() = competion.context

  var label = 0
  var result: Result<Any>? = null

  override fun resumeWith(result: Result<Unit>) {
    this.result = result
    val res = try {
      val r = myFunction(this)
      if (r == COROUTINE_SUSPENDED) return
      Result.success(r as Unit)
    } catch (e: Throwable) {
      Result.failure(e)
    }
    completion.resumeWith(res)
  }
}
```

### 📌 짚고 넘어가야할 점
코루틴의 동작 원리를 확실히 이해하기 위해, contiuation을 통한 중단과 재개를 코드로 직접 구현해보자 

### 상태를 가진 함수

함수가 중단된 후에 다시 상숑할 지역 변수나 파라미터와 같은 상태를 가지고 있다면, 함수의 컨티뉴에이션 객체에 
상태를 저장해야 합니다. 

지역변수나 파라미터 같이 함수 내에서 사용되던 값들을 중단되기 직전에 저장되고, 이후 함수가 재개될 때 복구됩니다.


## 콜 스택

함수 a가 함수 b를 호출하면 가상머신은 a의 상태와 b가 끝나면 실행이 될 지점을 어딘가에 저장해야 합니다. 이런 정보들은 모두 콜 스택(call stack)이라는
자료 구조에 저장됩니다. 코루틴을 중단하면 스레드를 반환해 콜 스택에 있는 정보가 사라질 것입니다. 그래서 컨티뉴에이션 객체가 콜 스택 역할을 대신합니다.
컨티뉴에이션 객체는 중단되었을 때의 상태(label)와 함수의 지역 변수와 파라미터(필드)를 가지고 있습니다.

컨티뉴에이션 객체가 재개될 때 각 컨티뉴에이션 객체는 자신이 담당하는 함수를 먼저 호출합니다. 함수의 실행이 끝나면 자신을 호출한 함수의 컨티뉴에이션을 재개합니다. 재개된 컨티뉴에이션 객체 또한 담당하는 함수를 호출하며, 이 과정은 스택의 끝에 다다를 때까지 반복됩니다.

함수 a가 함수 b를 호출하고, 함수 b는 함수 c를 호출하며, 함수 c에서 중단된 상황을 예로 들어봅시다. 실행이 재개되면 c의 컨티뉴에이션 객체는 c함수를 먼저 재개합니다. 함수가 완료되면 c 컨티뉴에이션 객체는 b 함수를 호출하는 b 컨티뉴에이션 객체를 재개합니다. b 함수가 완료되면 b 컨티뉴에이션은 a 컨티뉴에이션을 재개하고 a 함수가 호출되게 됩니다.

### 🧐 의문점
콜스택은 함수에 대한 정보를 어떻게 저장할까?

### 예외

처리되지 못한 예외가 resumewith에서 잡히면 Result.failure(e)로 래핑되며, 예외를 던진 함수를 호출한 함수는 포장된 결과를 받게 됩니다.


