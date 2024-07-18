## 1장 코루틴을 배워야하는 이유

### 1. 안드로이드에서의 코루틴 사용

```
fun onCreate(){
  val news = getNewsFromApi()
  val sortedNews = news
    .sortedByDescending { it.publishedAt }
  view.showNews(sortedNews)
}
```

안드로이드에서는 앞의 예제처럼 간단하게 구현할 수 없습니다. 안드로이드에서는 하나의 앱에서 뷰를 다루는 스레드가 하나 (메인 스레드) 만 존재합니다. 이 스레드는 앱에서 가장 중요한 스레드라 블로킹되면 안 되기 때문에, 이런 방법으로 구현할 수 없습니다.
결국, 위의 코드는 메인 스레드에서 실행되고, 해당 스레드가 블로킹 된다면 앱 크래시가 발생합니다.

### 2. 스레드 전환
```
fun onCreate(){
  thread {
    val news = getNewsFromApi()
    val sortedNews = news
        .sortedByDescending { it.publishedAt }
    runOnUiThread {
      view.showNews(sortedNews)
    }
  }
}
```

스레드 전환이 위에서 말한 문제를 푸는 가장 직관적인 방법입니다. 
몇몇 애플리케이션에서는 위와 같은 스레드 전환 방식을 아직도 사용하고 있지만, 다음과 같은 문제가 있습니다.
- 스레드가 실행되었을 때 멈출 수 있는 방법이 없어 메모리 누수로 이어질 수 있습니다.
- 스레드를 많이 생성하면 비용이 많이 듭니다.
- 스레드를 자주 전환하면 복잡도를 증가시키며 관리하기도 어렵습니다.
- 코드가 길어지고 이해하기 어려워집니다.

책을 읽고, 아래와 같은 개인적인 의문이 들었습니다.

#### 🧐 의문점1 
스레드를 멈출 수 있는 방법이 없을까? </br>
-> flag 또는 interrupt()를 통해 스레드를 종료 시키는 방법이 있는데, 왜 멈출 수 있는 방법이 없다고 나와있는지 모르겠다.. 

#### 🧐 의문점2 
스레드를 멈추지 못하면 왜 메모리 누수로 이어질까? </br>
-> 스레드를 멈추지 못한다면, 스레드가 참조하는 객체들이 메모리 상에 여전히 남아있고 이는 메모리 누수 현상이 발생시킵니다.

#### 🧐 의문점3 
스레드를 만드는 비용이 큰 이유가 뭘까? </br>
-> 자바에서 스레드를 생성할 땐 커널 스레드와 1:1 매핑되어 생성됩니다. 커널 스레드가 생성될 때 스레드를 위한 메모리 할당, 스레드 스택 초기화 및 OS 스레드 등록을 위한 os call을 하게되기 때문에 비용이 큽니다. (chapter1_study.md 파일에 추가 정리)

### 3. 콜백

콜백은 앞에서 주어진 문제를 해결하는 또 다른 패턴입니다. 콜백의 기본적인 방법은 함수를 논블로킹으로 만들고 함수의 작업이 끝났을 때 호출될 콜백 함수를 넘겨주는 것입니다.
```
fun showNews(){
  getCoifigFromApi { config ->
    getNewsFromApi(config) { news ->
      getUserFromApi { user ->
        view.showNews(user, news)
      }
    }    
  }
}
```

위 코드는 다음과 같은 이유 때문에 완벽한 해결책이 될 수 없습니다.
- 뉴스를 얻어오는 작업과 사용자 데이터를 얻어오는 작업은 병렬로 처리할 수 있지만, 현재의 콜백 구조로는 두 작업을 동시에 처리할 수 없습니다.
- 취소할 수 있도록 구현하려면 많은 노력이 필요합니다.
- 들여쓰기가 많아질수록 코드는 읽기가 어려워집니다.

#### 🧐 짚고 넘어가야할 점 
비동기, 논블로킹이란 차이는 무엇일까? 
-> 비동기는 A함수가 B함수를 실행하고, B함수의 결과를 기다리지 않는 것이고, 논블로킹은 A함수가 B함수를 실행할 때, B함수에게 제어권을 넘겨주지 않는 것입니다.

### 4. 코틀린 코루틴의 사용
코틀린 코루틴이 도입한 핵심 기능은 코루틴을 특정 지점에서 멈추고 이후에 재개할 수 있다는 것입니다. 
코루틴을 사용하면 우리가 짠 코드를 메인 스레드에서 실행하고 API에서 데이터를 얻어올 때 잠깐 중단시킬 수도 있습니다. 
코루틴을 중단시켰을 때 스레드는 블로킹되지 않으며 뷰를 바꾸거나 다른 코루틴을 실행하는 등의 또 다른 작업이 가능해집니다.
데이터가 준비되면 코루틴은 메인 스레드에서 대기하고 있다가 메인스레드가 준비되면 멈춘 지점에서 다시 작업을 수행합니다.

뉴스를 별도로 처리하는 작업을 다음과 같이 구현할 수 있습니다.

```
fun onCreate(){
  viewModelScope.launch {
    val news = getNewsFromApi()
    val sortedNews = news
        .sortedByDescending { it.publishedAt }
    view.showNews(sortedNews)
  }
}
```
여기서 코드는 메인 스레드에서 실행되지만, 스레드를 블로킹하지는 않습니다. 코루틴의 중단은 데이터가 오는 걸 기다릴 때 (스레드를 블로킹하는 대신) 
코루틴을 잠시 멈추는 방식으로 작동합니다. 코루틴이 멈춰 있는 동안, 메인 스레드는 프로그레스 바를 그리는 등의 다른 작업을 할 수 있습니다. 
데이터가 준비되면 코루틴은 다시 메인 스레드를 할당받아 이전에 멈춘 지점부터 다시 시작됩니다.

세 개의 엔드포인트를 호출해야 하는 문제는 어떻게 할까요?

```
fun onCreate(){
  viewModelScope.launch {
    val config = getConfigFromApi()
    val news = getNewsFromApi(config)
    val user = getUserFromApi()
    view.showNews(user, news)
  }
}
```

위 코드는 효율적이지 않습니다. 호출은 순차적으로 일어나기 때문에 각 호출이 1초씩 걸린다면 전체 함수는 3초가 걸립니다.
만약 병렬로 호출했다면 3초대신 2초만에 작업을 끝낼 수 있습니다. 

```
fun onCreate(){
  viewModelScope.launch {
    val config = async { getConfigFromApi() }
    val news = async { getNewsFromApi(config) } 
    val user = async { getUserFromApi() }
    view.showNews(user.await(), news.await())
  }
}
```

코틀린 코루틴은 다양한 상황에서 쉽게 적용 가능하며, 코틀린의 다른 기능 또한 활용 할 수 있다는 장점이 있습니다.
예를 들면 for-루프나 컬렉션을 처리하는 함수를 사용할 때 블로킹 없이 구현 가능합니다. 아래 예제에서 각 페이지를 병렬로 다운로드하거나 한 장씩 보여주는 방법을 확인할 수 있습니다.

```
// 모든 페이지를 동시에 받아옵니다.
fun showAllNews(){
  viewModelScope.launch {
    val allNews = (0 until getNumberOfPages())
        .map { page -> async { getNewsFromApi(page) }}
        .flatMap { it.await() }
    view.showAllNews(allNews)
  }
}
```

```
// 페이지별로 순차적으로 받아옵니다.
fun showAllNews(){
  viewModelScope.launch {
      for(page in 0 until getNumberOfPages()){
        val news = getNewsFromApi(page)
        view.showNextPage(news)
    }
  }
}
```

#### 🧐 의문점 
viewModelScope에서 여러개의 코루틴들을 동시에 실행한다면, 병렬적으로 실행되는 것이 아닌 순차적으로 실행되지 않을까? </br>
-> chapter1_study.md 파일 정리




