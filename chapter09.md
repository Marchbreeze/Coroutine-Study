## 1. 일시 중단 함수와 코루틴

### (1) 일시 중단 함수

- 일시중단 함수 : suspend fun 키워드로 선언되는 함수
    - 함수 내에 일시 중단 지점을 포함할 수 있는 특별한 기능을 함
    - 주로 코루틴의 비동기 작업과 관련된 복잡한 코드들을 구조화하고 재사용할 수 있는 코드의 집합으로 사용

- 재사용이 가능한 단위
    
    ```kotlin
    //AS-IS
    fun main() = runBlocking<Unit> {
      delay(1000L)
      println("Hello World")
      delay(1000L)
      println("Hello World")
    }
    ```
    
    ```kotlin
    //TO-BE
    fun main() = runBlocking<Unit> {
      delayAndPrintHelloWorld()
      delayAndPrintHelloWorld()
    }
    
    suspend fun delayAndPrintHelloWorld() {
      delay(1000L)
      println("Hello World")
    }
    ```
    

### (2) 일시 중단 함수는 코루틴이 아님

- 일시 중단 함수는 코루틴 내부에서 실행되는 코드의 집합일 뿐, 코루틴이 아님
    
    
- 코루틴처럼 사용하고 싶으면 코루틴 빌더로 감싸야 함
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      launch {
        delayAndPrintHelloWorld()
      }
      launch {
        delayAndPrintHelloWorld()
      }
      println(getElapsedTime(startTime))
    }
    
    suspend fun delayAndPrintHelloWorld() {
      delay(1000L)
      println("Hello World")
    }
    
    /*
    // 결과:
    지난 시간: 3ms
    Hello World
    Hello World
    */
    ```
    
    1. launch 함수가 호출돼 생성된 코루틴들은 실행되자마자 delayAndPrintHelloWorld 함수의 호출로 1초 간 스레드 사용 권한을 양보
    2. 자유로워진 스레드는 다른 코루틴인 runBlocking 코루틴에 의해 사용될 수 있으므로 곧바로 마지막 줄의 getElapsedTime이 실행됨

## 2. 일시 중단 함수의 사용

### (1) 일시 중단 함수의 호출 가능 지점

- 일시 중단 함수는 일시 중단이 가능한 곳에서만 호출할 수 있음
    1. 코루틴 내부
    2. 일시 중단 함수

1. 코루틴 내부에서의 호출
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      delayAndPrint(keyword = "I'm Parent Coroutine")
      launch {
        delayAndPrint(keyword = "I'm Child Coroutine")
      }
    }
    
    suspend fun delayAndPrint(keyword: String) {
      delay(1000L)
      println(keyword)
    }
    
    /*
    // 결과:
    I'm Parent Coroutine
    I'm Child Coroutine
    */
    ```
    

1. 일시 중단 함수 내부에서의 호출
    
    ```kotlin
    suspend fun searchByKeyword(keyword: String): Array<String> {
      val dbResults = searchFromDB(keyword)
      val serverResults = searchFromServer(keyword)
      return arrayOf(*dbResults, *serverResults)
    }
    
    suspend fun searchFromDB(keyword: String): Array<String> {
      delay(1000L)
      return arrayOf("[DB]${keyword}1", "[DB]${keyword}2")
    }
    
    suspend fun searchFromServer(keyword: String): Array<String> {
      delay(1000L)
      return arrayOf("[Server]${keyword}1", "[Server]${keyword}2")
    }
    ```
    

### (2) 일시 중단 함수에서 코루틴 실행

- 일시 중단 함수에서 코루틴 빌더를 호출하는 경우, 순차적으로 실행됨
    
    → async 코루틴 빌더 함수로 감싸 서로 다른 코루틴에서 실행되도록 해결
    

- 그러나, launch와 async와 같은 코루틴 빌더 함수는 CoroutineScope의 확장 함수로 선언되어 있음
    
    → 일시 중단 함수 내부에서 일시 중단 함수를 호출한 코루틴의 CoroutineScope 객체에 접근할 수 없음
    
    → 오류 발생
    

- coroutineScope 사용해 일시 중단 함수에서 코루틴 실행하기
    - coroutineScope 일시 중단 함수를 사용하면, 내부에 새로운 CoroutineScope 객체를 생성할 수 있음
    - 구조화를 깨지 않으며, block 람다식에서 수신 객체(this)로 접근할 수 있음
        
        ```kotlin
        public suspend fun ‹R> coroutineScope(block: suspend CoroutineScope. () -> R:
        ```
        
    - ex.
        
        ```kotlin
        suspend fun searchByKeyword(keyword: String): Array<String> = coroutineScope { // this: CoroutineScope
          val dbResultsDeferred = async {
            searchFromDB(keyword)
          }
          val serverResultsDeferred = async {
            searchFromServer(keyword)
          }
        
          return@coroutineScope arrayOf(*dbResultsDeferred.await(), *serverResultsDeferred.await())
        }
        ```
        
        ```kotlin
        /*
        // 결과:
        [결과] [[DB]Keyword1, [DB]Keyword2, [Server]Keyword1, [Server]Keyword2]
        지난 시간: 1039ms
        */
        ```
        
    - 문제점
        - async에서 빌드된 코루틴에서 오류가 발생하는 경우, 부모 코루틴으로 전파되어 모두 취소시킴

- supervisorScope 일시 중단 함수를 사용하면, 예외 전파를 제한하면서 coroutineScope와 동일하게 동작
    
    ```kotlin
    public suspend fun <R> supervisorScope(block: suspend CoroutineScope. () -> R: R
    ```
    
- Deffered 객체 : await 함수 호출 시 추가로 예외를 노출 → try-catch문을 통해 제어
    
    ```kotlin
    suspend fun searchByKeyword(keyword: String): Array<String> = supervisorScope { // this: CoroutineScope
      val dbResultsDeferred = async {
        throw Exception("dbResultsDeferred에서 예외가 발생했습니다")
        searchFromDB(keyword)
      }
      val serverResultsDeferred = async {
        searchFromServer(keyword)
      }
    
      val dbResults = try {
        dbResultsDeferred.await()
      } catch (e: Exception) {
        arrayOf() // 예외 발생 시 빈 결과 반환
      }
    
      val serverResults = try {
        serverResultsDeferred.await()
      } catch (e: Exception) {
        arrayOf() // 에러 발생 시 빈 결과 반환
      }
    
      return@supervisorScope arrayOf(*dbResults, *serverResults)
    }
    ```
    
    ```kotlin
    /*
    // 결과:
    [결과] [[Server]Keyword1, [Server]Keyword2]
    */
    ```
