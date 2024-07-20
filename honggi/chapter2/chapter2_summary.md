책을 읽으며 의문점이 생기는 부분들은 chapter2_study.md 파일에 정리합니다.

## 시퀀스 빌더

코틀린에서 제너레이터 대신 시퀀스를 생성할 때 사용하는 시퀀스 빌더를 제공하고 있습니다.
코틀린의 시퀀스는 List나 Set과 같은 컬렉션이랑 비슷한 개념이지만, 필요할 때마다 값을 하나씩 계산하는 지연(Lazy)처리를 합니다. 시퀀스의 특징은 다음과 같습니다.

- 요구되는 연산을 최소한으로 수행합니다.
- 무한이 될 수 있습니다.
- 메모리 사용이 효율적입니다.

이러한 특징 때문에 값을 순차적으로 계산하여 필요할 때 반환하는 빌더를 정의하는 것이 좋습니다. 시퀀스는 ```sequence```라는 
함수를 이용해 정의합니다. 시퀀스의 람다 표현식 내부에서는 ```yield```함수를 호출하여 시퀀스의 다음 값을 생성합니다.

```
val seq = sequence {
    yield(1)
    yield(2)
    yield(3)
}

fun main() {
  for (num in seq) {
      print(num)
  }
}
```

🧐 의문점
sequence는 어떻게 위 코드처럼 for문에서 사용할 수 있을까? iterator를 구현하는 건가?, 내부 구조를 확인해보자

위 코드에서 사용한 sequence 함수는 짧은 DSL 코드입니다. 인자는 수신 객체 지정 람다 함수입니다 (suspend SequenceScope<T>() -> Unit). 람다 내부에서 수신 객체인 this는
SequenceScope<T>를 가리킵니다. 이 객체는 yield 함수를 자기고 있습니다. this가 암시적으로 사용되므로 yeild(1)을 호출하면 this.yield(1)을 호출하는 것과 동일합니다.

여기서 반드시 알야하 하는 것은 각 숫자가 미리 생성되는 대신, 필요할 때마다 생성된다는 점입니다. 시퀀스 빌더 내부 그리고 시퀀스를 사용하는 곳에서 메시지를 출력하면 이러한 작동 방식을 쉽게 확인할 수 있습니다.

```
val seq = sequence {
    println("Generating first")
    yield(1)
    println("Generating second")
    yield(2)
    println("Generating thrid")
    yield(3)
    println("Done")
}

fun main() {
  for (num in seq) {
      print("The next number is $num")
  }
}
// Generating first
// The next number is 1
// Generating second
// The next number is 2
// Generating Third
// The next number is 3
// Done
```
시퀀스의 작동 방식에 대해 알아봅시다. 우선 첫 번째 수를 요청하면 빌더 내부로 진입합니다. "Generating first"를 출력한 뒤, 숫자 1을 반환합니다. 
이후 반복문에서 반환된 값을 받은 뒤, "Next number is 1"을 출력합니다. 여기서 반복문과 다른 결정적인 차이가 보입니다. 이전에 다른 숫자를 찾기 위해 멈췄던 지점에서
다시 실행이 됩니다. 중단 체제가 없으면 함수가 중간에 멈췄다가, 나중에 중단된 지점에서 다시 실행되는 건 불가능합니다. 중단이 가능하기 때문에 main 함수와 시퀀스 제너레이터가 번갈아가면서 실행됩니다.

```
val seq = sequence {
    yield(1)
    println("Generating second")
    yield(2)
    println("Generating third")
    yield(3)
    println("Done")
}

fun main() {
    val iterator = seq.iterator()
    println("Starting")
    val first = itrator.next()
    println("First: $first")
    val second = iterator.next()
    println("Second: $second")
    // ...
}
```
이터레이터를 통해 다음 값을 얻을 수 있습니다. 코루틴 없이 이런 게 가능할까요? 스레드가 이 일을 대신할 수도 있습니다. 하지만 중단을 지원하는 스레드로 처리하려면 유지하고 관리하는데 막대한 비용이 듭니다. 코루틴을 사용하면 더 빠르고 간단하게 중단이 가능합니다. 




