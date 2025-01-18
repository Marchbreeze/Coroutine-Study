## 1. CoroutineDispatcher의 역할

- CoroutineDispatcher
    - 코루틴이 사용할 수 있는 스레드나 스래드풀을 결정 → 스레드로 보내 실행시키는 역할

- 동작
    - ex. 2개의 스레드로 구성된 스레드풀 & CoroutineDispatcher 객체의 경우
        
        ![2025-01-19_01-39-44.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/dfb429d2-812a-4940-90f5-294309ac76dd/2025-01-19_01-39-44.jpg)
        
        1. CoroutineDispatcher 객체는 실행 요청받은 코루틴을 작업 대기열에 적재
        2. 자신이 사용할 수 있는 스레드가 있는지 확인
        3. 사용 가능한 스레드가 있으면 - 적재된 코루틴을 스레드로 보내서 실행시킴
        4. 사용 가능한 스레드가 없으면 - 작업 대기열에서 대기 & 기존 스레드가 자유로워지면 실행

- CoroutineDispatcher의 종류 : 사용할 수 있는 스레드 or 스레드풀의 제한 유무
    1. 제한된 디스패처 (Confined Dispatcher)
        - 일반적으로는 CoroutineDispatcher 객체별로 어떤 작업을 처리할지 미리 역할을 부여하고 실행을 요청하는 것이 효율적
        - 대부분 제한된 디스패처 활용
        - ex. 입출력 작업(I/O), CPU 연산 작업 용 CoroutineDispatcher
    2. 무제한 디스패처 (Unconfined Dispatcher)
        - 실행 요청된 코루틴이 이전 코드가 실행되던 스레드에서 계속해서 실행되도록 설정
        - 특정 스레드로 제한 X

## 2. 제한된 디스패처 생성

- 단일 스레드 디스패처 (Single-Thread Dispatcher)
    - 사용할 수 있는 스레드가 하나인 CoroutineDispatcher 객체
    - name 파라미터 : 디스패처에서 고나리하는 스레드명
        
        ```kotlin
        val dispatcher: CoroutineDispatcher = newSingleThreadContext(
        	name = "SingleThread"
        )
        ```
        

- 멀티 스레드 디스패처 (Multi-Thread Dispatcher)
    - 사용할 수 있는 스레드가 2개 이상인 CoroutineDispatcher 객체
    - 파라미터에서 nThreads 값으로 개수 조정
    - 만들어지는 스레드들은 인자로 받은 name값 뒤에 -1, -2, …
        
        ```kotlin
        val multiThreadDispatcher: CoroutineDispatcher = newFixedThreadPoolContext(
          nThreads = 2,
          name = "MultiThread"
        )
        ```
        

- 내부 코드
    - ExecutorService를 생성하는 함수를 통해 스레드풀 생성
    - JVM 종료 시 애플리케이션이 블로킹되지 않도록 하기 위해서 모두 데몬 스레드로 구성

## 3. 코루틴 실행

### (1) Launch의 파라미터

1. 단일 스레드 디스패처
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val dispatcher = newSingleThreadContext(
    		name = "SingleThread"
      )
      launch(context = dispatcher) {
        println("[${Thread.currentThread().name}] 실행")
      }
    }
    /*
    // 결과:
    [SingleThread @coroutine#2] 실행
    */
    ```
    
2. 멀티 스레드 디스패처
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val multiThreadDispatcher = newFixedThreadPoolContext(
        nThreads = 2,
        name = "MultiThread"
      )
      launch(context = multiThreadDispatcher) {
        println("[${Thread.currentThread().name}] 실행")
      }
      launch(context = multiThreadDispatcher) {
        println("[${Thread.currentThread().name}] 실행")
      }
    }
    /*
    // 결과:
    [MultiThread-1 @coroutine#2] 실행
    [MultiThread-2 @coroutine#3] 실행
    */
    ```
    

### (2) 부모 코루틴의 디스패처

- 코루틴은 구조화를 제공해 코루틴 내부에서 새로운 코루틴 실행 가능
- 부모 코루틴 (바깥 코루틴) & 자식 코루틴 (내부에서 생성되는 새로운 코루틴)
- 부모 코루틴의 실행 환경을 자식 코루틴에 전달도 가능
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val multiThreadDispatcher = newFixedThreadPoolContext(
        nThreads = 2,
        name = "MultiThread"
      )
      launch(multiThreadDispatcher) { // 부모 Coroutine
        println("[${Thread.currentThread().name}] 부모 코루틴 실행")
        launch { // 자식 코루틴 실행
          println("[${Thread.currentThread().name}] 자식 코루틴 실행")
        }
        launch { // 자식 코루틴 실행
          println("[${Thread.currentThread().name}] 자식 코루틴 실행")
        }
      }
    }
    /*
    // 결과:
    [MultiThread-1 @coroutine#2] 부모 코루틴 실행
    [MultiThread-2 @coroutine#3] 자식 코루틴 실행
    [MultiThread-1 @coroutine#4] 자식 코루틴 실행
    */
    ```
    
    - 자식 코루틴에서 별도의 CoroutineDispatcher 객체가 설정되어 있지 않으므로 부모 코루틴에서 설정했던 CoroutineDispatcher 객체 사용

## 4. 사전 정의 CoroutineDispatcher

- 앞서 소개한 newFixedThreadPoolContext 함수를 사용해서 객체 생성 시 경고 출력됨
    - 비효율적일 가능성이 높기 때문
        - 특정 CoroutineDispatcher 객체에서만 사용되는 스레드풀 생성
            
            → 스레드 풀 내 스레드 개수의 비효율 가능성 & 스레드 생성 비용
            
    - 사전에 정의된 CoroutineDispatcher 활용을 권장함

### (1) Dispatchers.IO

- 네트워크 요청이나 파일 입출력 등의 입출력(I/O) 작업 용도
- 스레드 수의 제한 : JVM에서 사용이 가능한 프로세서 수 or 64 중 큰 수
    
    → 여러 입출력 작업을 동시에 수행 가능
    
- 싱글톤 인스턴스 - launch 함수의 인지로 활용
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(Dispatchers.IO) {
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
    }
    /*
    // 결과:
    [DefaultDispatcher-worker-1 @coroutine#2] 코루틴 실행
    */
    ```
    
    - 공유 스레드풀의 스레드 사용 → worker-1 스레드에 할당

### (2) Dispatchers.Default

- CPU 바운드 작업 (CPU를 많이 사용하는 연산 작업) 용도
- 싱글톤 인스턴스 - launch 함수의 인지로 활용
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(Dispatchers.Default){
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
    }
    /*
    // 결과:
    [DefaultDispatcher-worker-1 @coroutine#2] 코루틴 실행
    */
    ```
    

- 입출력 작업과의 차이 : 작업이 실행되었을 때 스레드를 지속적으로 사용해야 함
- 스레드를 지속적으로 사용 → CPU 바운드 작업의 경우 스레드와 처리 속도에서 큰 차이 없음
    
    ![2025-01-19_02-47-33.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/f7c38410-7a7c-42c3-b283-dc867d958662/2025-01-19_02-47-33.jpg)
    

- limitedParallelism 함수
    - Dispatchers.Default를 사용할 때 무거운 연산 처리 시 모든 스레드를 독점할 수 있는 가능성 존재
    - 해당 연산에 할당될 스레드 수를 제한할 수 있음
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          launch(Dispatchers.Default.limitedParallelism(2)){
            repeat(10) {
              launch {
                println("[${Thread.currentThread().name}] 코루틴 실행")
              }
            }
          }
        }
        /*
        // 결과:
        [DefaultDispatcher-worker-2 @coroutine#3] 코루틴 실행
        [DefaultDispatcher-worker-1 @coroutine#4] 코루틴 실행
        [DefaultDispatcher-worker-2 @coroutine#5] 코루틴 실행
        ...
        [DefaultDispatcher-worker-1 @coroutine#10] 코루틴 실행
        [DefaultDispatcher-worker-2 @coroutine#11] 코루틴 실행
        [DefaultDispatcher-worker-2 @coroutine#12] 코루틴 실행
        */
        ```
        

- 공유 스레드풀
    - 스레드의 생성과 관리를 효율적으로 할 수 있도록 애플리케이션 레벨의 공유 스레드풀 제공
    - 스레드 무제한 생성 가능
    - 코루틴 라이브러리는 공유 스레드풀에 스레드를 생성하고 사용할 수 있도록 하는 API 제공
        
        ![2025-01-19_03-03-44.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/f73c10c6-46e9-42f6-8ce9-3796dccaf17c/2025-01-19_03-03-44.jpg)
        

### (3) Dispatchers.Main

- 메인 스레드 사용하기 위한 용도
- 일반적으로 UI가 있는 애플리케이션에서 메인 스레드의 사용을 위해 사용되는 특별한 디스패처
- 별도의 라이브러리 필요 (kotlinx-coroutines-android)
