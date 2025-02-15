## 1. async 사용해 결과값 수신하기

- launch 코루틴 빌더를 통해 생성되는 코루틴은 기본적으로 작업 수행 후 결과를 반환하지 않음
    - 결과값이 없는 코루틴 객체인 Job 반환
    - 그러나 결과를 수신해야 하는 경우 빈번 : ex. 네트워크 통신 실행 후 응답 처리
- async 코루틴 빌더를 통해 결과값 수신받을 수 있음
    - 결과값이 있는 코루틴 객체인 Deferred 반환

### (1) async 사용해 Deferred 만들기

- async 코루팀 빌더 선언부
    
    ```kotlin
    public fun ‹T> CoroutineScope.async(
    		context: CoroutineContext = EmptyCoroutineContext, 
    		start : CoroutineStart = CoroutineStart.DEFAULT, 
    		block: suspend CoroutineScope.() -> T
    ): Deferred<T>
    ```
    
    - launch 함수와 유사 (Dispatcher 설정, LAZY 지연 시작 설정, block 람다식)
    - Deferred<T> 객체 반환
        - Job과 같이 코루틴을 추상화한 객체이지만, 생성된 결과값을 감싸는 기능을 추가로 가짐

- Deferred의 제네릭 타입 지정
    - 명시적으로 타입을 설정하거나 async 블록의 반환값으로 결과값 설정
        
        ```kotlin
          val networkDeferred: Deferred<String> = async(Dispatchers.IO) {
            delay(1000L) // 네트워크 요청
            return@async "Dummy Response" // 결과값 반환
          }
        ```
        
        - String 타입의 return → networkDeferred의 타입은 Deferred<String>

### (2) await을 사용한 결과값 수신하기

- Deferred 객체는 미래의 어느 시점에 결과값이 반환될 수 있음을 표현하는 코루틴 객체
    - 언제 결과값이 반환될 지 정확히 알 수 없으며, 필요 시 수신될 때까지 대기해야 함

- Deferred 객체는 결과값 수신의 대기를 위해 await 함수를 제공
    - await 함수 : Deferred 코루틴이 실행 완료될 때까지 코루틴을 일시 중단 & 완료 시 재개 (join과 유사)
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val networkDeferred: Deferred<String> = async(Dispatchers.IO) {
            delay(1000L) // 네트워크 요청
            return@async "Dummy Response" // 결과값 반환
          }
          val result = networkDeferred.await() // networkDeferred로부터 결과값이 반환될 때까지 runBlocking 일시 중단
          println(result) // Dummy Response 출력
        }
        /*
        // 결과:
        [DefaultDispatcher-worker-1 @coroutine#2] 토큰 업데이트 시작
        [DefaultDispatcher-worker-3 @coroutine#3] 네트워크 요청
        [DefaultDispatcher-worker-1 @coroutine#2] 토큰 업데이트 완료
        */
        ```
        
        ![2025-02-15_15-32-01.jpg](attachment:3437fa9b-c794-45a8-846c-2c70f4ffd773:2025-02-15_15-32-01.jpg)
        

## 2. Deferred는 특수한 형태의 Job

- Deferred 인터페이스는 Job 인터페이스의 서브타입으로 선언 (Job 객체의 일종)
    
    ```kotlin
    public interface Deferred<out> T : Job { 
    		public suspend fun await(): T
    		...
    }
    ```
    

- Deferred 객체는 Job 객체의 모든 함수와 프로퍼티 사용할 수 있음
    - join, cancel 함수 & isActive, isCancelled, isCompleted 프로퍼티
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val networkDeferred: Deferred<String> = async(Dispatchers.IO) {
        delay(1000L) // 네트워크 요청
        "Dummy Response"
      }
      networkDeferred.join() // networkDeferred가 실행 완료될 때까지 대기
      printJobState(networkDeferred) // Job이 입력되어야 할 자리에 Deferred 입력
    }
    
    /*
    // 결과:
    Job State
    isActive >> false
    isCancelled >> false
    isCompleted >> true
    */
    ```
    

## 3. 복수의 코루틴으로부터 결과값 수신하기

### (1) await을 사용해 복수의 코루틴으로부터 결과값 수신하기

- 순차적 처리
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis() // 1. 시작 시간 기록
      val participantDeferred1: Deferred<Array<String>> = async(Dispatchers.IO) { // 2. 플랫폼1에서 등록한 관람객 목록을 가져오는 코루틴
        delay(1000L)
        return@async arrayOf("James","Jason")
      }
      val participants1 = participantDeferred1.await() // 3. 결과가 수신 될 때까지 대기
    
      val participantDeferred2: Deferred<Array<String>> = async(Dispatchers.IO) { // 4. 플랫폼2에서 등록한 관람객 목록을 가져오는 코루틴
        delay(1000L)
        return@async arrayOf("Jenny")
      }
      val participants2 = participantDeferred2.await() // 5. 결과가 수신 될 때까지 대기
    
      println("[${getElapsedTime(startTime)}] 참여자 목록: ${listOf(*participants1, *participants2)}") // 6. 지난 시간 표시 및 참여자 목록을 병합해 출력
    }
    /*
    // 결과:
    [지난 시간: 2018ms] 참여자 목록: [James, Jason, Jenny]
    */
    
    ```
    
    ![2025-02-15_15-47-02.jpg](attachment:2c8bbae0-e1ef-491f-93e2-59ecfb95967f:2025-02-15_15-47-02.jpg)
    
- 독립적인 작업을 동시에 처리
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis() // 1. 시작 시간 기록
      val participantDeferred1: Deferred<Array<String>> = async(Dispatchers.IO) { // 2. 플랫폼1에서 등록한 관람객 목록을 가져오는 코루틴
        delay(1000L)
        return@async arrayOf("James","Jason")
      }
    
      val participantDeferred2: Deferred<Array<String>> = async(Dispatchers.IO) { // 3. 플랫폼2에서 등록한 관람객 목록을 가져오는 코루틴
        delay(1000L)
        return@async arrayOf("Jenny")
      }
    
      val participants1 = participantDeferred1.await() // 4. 결과가 수신 될 때까지 대기
      val participants2 = participantDeferred2.await() // 5. 결과가 수신 될 때까지 대기
    
      println("[${getElapsedTime(startTime)}] 참여자 목록: ${listOf(*participants1, *participants2)}") // 6. 지난 시간 기록 및 참여자 목록 병합
    }
    /*
    // 결과:
    [지난 시간: 1018ms] 참여자 목록: [James, Jason, Jenny]
    */
    
    ```
    
    ![2025-02-15_15-48-54.jpg](attachment:75104353-85f9-4edb-b5a3-123b0ed87df6:2025-02-15_15-48-54.jpg)
    
    - participantDeferred1.await()가 호출되기 전에 participantDeferred2 코루틴 실행 → 동시 실행

### (2) awaitAll을 사용한 결과값 수신하기

- 2개 이상의 함수 호출 시, await 함수를 반복해서 나열되는 것은 가독성에 좋지 않음 → awaitAll 함수 제공
    
    ```kotlin
    public suspend fun ‹T› awaitAll(vararg deferreds: Deferred<T>): List‹T›
    ```
    
    - awaitAll 함수는 가변 인자로 Deferred 타입의 객체를 받아 인자로 받은 모든 Deffered 코루틴으로부터 결과가 수신될 때까지 호출부의 코루틴을 일시 중단
    - 결과가 모두 수신되면 Deferre 코루틴으로부터 수신한 결과값들을 List로 만들어서 반환 후 호출부의 코루틴 재개

- awaitAll 함수 활용
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      val participantDeferred1: Deferred<Array<String>> = async(Dispatchers.IO) {
        delay(1000L)
        arrayOf("James","Jason")
      }
    
      val participantDeferred2: Deferred<Array<String>> = async(Dispatchers.IO) {
        delay(1000L)
        arrayOf("Jenny")
      }
    
      val results: List<Array<String>> = awaitAll(participantDeferred1, participantDeferred2) // 요청이 끝날 때까지 대기
    
      println("[${getElapsedTime(startTime)}] 참여자 목록: ${listOf(*results[0], *results[1])}")
    }
    /*
    // 결과:
    [지난 시간: 1013ms] 참여자 목록: [James, Jason, Jenny]
    */
    ```
    
    ![2025-02-15_15-52-04.jpg](attachment:60d87cc4-383f-4e16-8955-9bcf70aef998:2025-02-15_15-52-04.jpg)
    

### (3) 컬렉션에 대해 awaitAll 사용하기

- awaitAll 함수를 Collection 인터페이스에 대한 확장 함수로도 제공됨
    
    ```kotlin
    public suspend fun ‹T› Collection<Deferred<T>>.awaitAll(): List<T>
    ```
    
    - Collection<Deferred<T>>에 대해 awaitAll 함수를 호출하면 컬렉션에 속한 Deferred들이 모두 완료되어 결과값을 반환할 때까지 대기

- List 인터페이스와 사용하는 예시
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis() // 시작 시간 기록
      val participantDeferred1: Deferred<Array<String>> = async(Dispatchers.IO) {
        delay(1000L) // 참여자 이름 조회 요청
        arrayOf("James", "Jason") // 참여자 반환
      }
    
      val participantDeferred2: Deferred<Array<String>> = async(Dispatchers.IO) {
        delay(1000L) // 참여자 이름 조회 요청
        arrayOf("Jenny") // 참여자 반환
      }
    
      val results: List<Array<String>> = listOf(participantDeferred1, participantDeferred2).awaitAll() // 요청이 끝날 때까지 대기
    
      println("[${getElapsedTime(startTime)}] 참여자 목록: ${listOf(*results[0], *results[1])}")
    }
    /*
    // 결과:
    [지난 시간: 1015ms] 참여자 목록: [James, Jason, Jenny]
    */
    ```
    

## 4. withContext

### (1) withContext로 async-await 대체하기

- withContext
    
    ```kotlin
    public suspend fun <T> withContext(
    		context: CoroutineContext,
    		block: suspend CoroutineScope.() -> T
    ): T
    ```
    
    - withContext 함수 호출 시, 함수의 인자로 설정된 CoroutineContext 객체를 사용해 block 람다식 싱행 후 완료되면 결과 반환
    - block 람다식을 모두 실행하면 다시 기존의 CoroutineContext 객체를 사용해 코루틴 재개

- async-await 대체
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val networkDeferred: Deferred<String> = async(Dispatchers.IO) {
        delay(1000L) // 네트워크 요청
        return@async "Dummy Response" // 문자열 반환
      }
      val result = networkDeferred.await() // networkDeferred로부터 결과값이 반환될 때까지 대기
      println(result)
    }
    /*
    // 결과:
    Dummy Response
    */
    ```
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val result: String = withContext(Dispatchers.IO) {
        delay(1000L) // 네트워크 요청
        return@withContext "Dummy Response" // 문자열 반환
      }
      println(result)
    }
    /*
    // 결과:
    Dummy Response
    */
    ```
    

### (2) withContext의 동작 방식

- 내부적으로는 async-await과 다르게 동작
- withContext : 실행 중이던 코루틴을 그대로 유지한 채로, 코루틴의 실행 환경만 변경해 작업 처리
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      println("[${Thread.currentThread().name}] runBlocking 블록 실행")
      withContext(Dispatchers.IO) {
        println("[${Thread.currentThread().name}] withContext 블록 실행")
      }
    }
    /*
    // 결과:
    [main @coroutine#1] runBlocking 블록 실행
    [DefaultDispatcher-worker-1 @coroutine#1] withContext 블록 실행
    */
    ```
    
    - 동일한 coroutine#1이지만 실행되는 스레드는 다름

- Context Switching
    - withContext 함수 호출 시 코루틴의 실행 환경이 context 인자 값으로 변경되어 실행
    - context 인자로 CoroutineDispatcher 객체가 넘어온다면, 해당 객체를 사용해 다시 실행
        
        ![2025-02-15_18-11-37.jpg](attachment:63c40b89-840f-49b5-b944-2142cc1f2605:2025-02-15_18-11-37.jpg)
        
    - withContext 함수가 block 람다식을 벗어나면 다시 원래의 CoroutineContext 객체를 사용해 실행

### (3) withContext 사용 시 주의점

- withContext 함수는 새로운 코루틴을 만들지 않기 때문에, 하나의 코루틴에서 여러번 호출되면 순차적으로 실행됨
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      val helloString = withContext(Dispatchers.IO) {
        delay(1000L)
        return@withContext "Hello"
      }
    
      val worldString = withContext(Dispatchers.IO) {
        delay(1000L)
        return@withContext "World"
      }
    
      println("[${getElapsedTime(startTime)}] ${helloString} ${worldString}")
    }
    /*
    // 결과:
    [지난 시간: 2018ms] Hello World
    */
    ```
    
    ![2025-02-15_18-14-02.jpg](attachment:539d67c1-819d-492a-88cb-c479a7edd960:2025-02-15_18-14-02.jpg)
