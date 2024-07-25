## 중단은 어떻게 작동할까?

코루틴을 중단한다는 건 실행을 중간에 멈추는 것을 의미합니다. </br>
코루틴은 Continuation을 이용하여 멈췄던 곳에서 다시 코루틴을 실행합니다. </br>

여기서 코루틴은 스레드와 많이 다른 것을 알 수 있는데, 그레드는 저장이 불가능하고 멈추는 것만 가능하기 때문입니다.</br>
이러한 점에서 코루틴이 훨씬 강력한 도구라 할 수 있습니다. 

### 재개
```
suspend fun main() {
    print("Before")

    suspendCoroutine<Unit>

    println("After")
}
// Before
```

위 코드를 실행하면 "After"는 출력되지 않으며, 코드는 실행된 상태로 유지됩니다. 코루틴은 "Before" 이후에 중단됩니다. </br>
프로그램은 멈춘 뒤 재개되지 않습니다. 그러면 어떻게 다시 실행시킬 수 있을까요? 앞서 언급했던 Contiuation은 어디 있을까요?</br>
suspendCoroutine이 호출된 지점을 보면 람다 표현식 ({ })으로 끝났다는 걸 알 수 있습니다. </br>
인자로 들어간 람다 함수는 중단되기 전에 실행됩니다.</br>

```
suspend fun main() {
  println("Before")

  suspendCoroutine<Unit> { continuation ->
    println("Before too")
  }

  println("Aftrer")
}
// Before 
// Before too
```

suspendCoroutine의 람다 매개변수를 아래와 같이 변경할 수 있습니다.</br>

```
suspendCoroutine<Unit> { continuation ->
    contination.resume(Unit)
}
```

컨티뉴에이션 객체를 이용해 코루틴을 중단한 후 곧바로 실행할 수 있습니다.</br>

```
suspend fun main() {
  println("Before")

  suspendCoroutine<Unit> { continuation ->
    thread {
      println("Suspend")
      Thread.sleep(1000)
      contiuation.resume(Unit)
      println("Resumed")
    }
  }
}

// Before
// Suspended
// (1초후)
// After
// Resumed
```

위와 같은 방식으로 실행을 멈출 수 있지만, 만들어진 다음 1초 뒤에 사라지는 스레드는 불필요해 보입니다.</br>
이를 위 해 JVM이 제공하는 ScheduledExcutorService를 사용할 수 있습니다. </br>
정해진 시간이 지나면 contiuation.resume(Unit)을 호출하도록 알람을 설정할 수 있습니다.</br>

```
private val executor = 
    Executors.newSingleThreadScheduledExecutor {
        Thread(it, "scheduler").apply { isDaemon = true }
    }
  }

suspend fun main() {
  println("Before")

  suspendCoroutine<Unit> { continuation ->
    executor.schedule({
      continuation.resume(Unit)
    }, 1000, TimeUnit.MILLISECONDS)
  }

  println("After")
}
```

#### 🧐 의문점
결국 앞선 예제에서 스레드를 불필요하게 생성한다고 했는데 해당 예제에서도 불필요하게 생성하는거 아닌가?</br>

이를 delay 추출하면 아래와 같아집니다.

```
private val executor = 
  Executors.newSingleThreadScheduledExecutor {
    Executors.newSingleThreadScheduledExecutor {
        Thread(it, "scheduler").apply { isDaemon = true }
    }
  }

suspend fun delay(timeMillis: Long): Unit {
  suspendCoroutine<Unit> { continuation ->
    executor.schedule({
      continuation.resume(Unit)
    }, timeMillis, TimeUnit.MILLISECONDS)
  }
}

suspend fun main() {
  println("Before")

  delay(1000)
  
  println("After")
}

// Before
// (1초 후)
// After
```

여기서 이그제큐터는 스레드를 사용하긴 하지만 delay 함수를 사용하는 모든 코루틴의 전용 스레드입니다. 
앞에서 설명한 대기할 때마다 하나의 스레드를 블로킹하는 방법보다 훨씬 낫습니다.

위 코드는 코틀린 코루틴 라이브러리에서 delay가 구현된 방식이랑 정확히 일치합니다.

#### 🧐 의문점
delay 함수 내부와 비교해보자 

### 값으로 재개하기

```
suspendCoroutine<Int> { cont ->
  cont.resume(42)
}
```
suspendCoroutine을 호출할 때 컨티뉴에이션 객체로 반환될 값의 타입을 지정할 수 있습니다. 
resume을 통해 반환되는 값은 반드시 지정된 타입과 같은 타입이어야 합니다.




