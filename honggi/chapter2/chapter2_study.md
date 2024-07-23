## sequence 왜 효율적일까?

sequence는 최종 연산이 실행되기 전까지 즉시 실행되지 않습니다. </br>
즉, toList(), take() 같은 함수를 호출해 결과를 산출할 때만 연산 작업이 실행됩니다.

list, sequence를 만든 다음, map 중간연산자를 통해 실행해보았습니다.

```
listOf(1, 2, 3)
    .map { println(it) }

listOf(4, 5, 6)
    .asSequence()
    .map { println(it) }

// result
// 123
```

결과는 list 아이템들만 출력이 되었습니다. </br></br>

기존 리스트의 중간 연산자 map으로 어떠한 연산을 하게되면 그 즉시 실행이 되는데, </br>
sequence는 map 연산을 할 때 실행되는게 아니라 최종 연산(toList(), take(), onEach() 등)을 실행할 때 연산이 됩니다.

### Effective Kotlin 에선 아래와 같이 sequence 특징을 말합니다.

#### 특징1. 각 단계에서 컬렉션을 만들지 않습니다.
그렇기 때문에 불필요하게 중간 과정마다 컬렉션을 만들어 메모리를 차지 하는 것을 막을 수 있습니다. </br>

Iterable은 각 단계에서 결과를 저장하기 위해 새로운 컬렉션을 만듭니다.

```
listOf(1, 2, 3)
    .filter { ... }   // 새로운 컬렉션 생성 
    .map { println(it) }  // 새로운 컬렉션 생성 

listOf(4, 5, 6)
    .asSequence()
    .filter { ... }
    .map { println(it) }
```

#### 특징2. 최소 연산만 실행합니다.
중간 처리 단계를 모든 리스트의 아이템들에 적용할 필요 없는 경우에 시퀀스를 사용하여 연산량을 줄일 수 있습니다.

```
listOf(1, 2, 3)
    .map { println(it) } 
    .first()

// 123

listOf(4, 5, 6)
    .asSequence()
    .map { println(it) }
    .first()

// 4
```

#### 특징3. 무한 시퀀스로 생성 가능합니다.

한번 제가 만들어보겠습니다.

todo 
```

```


