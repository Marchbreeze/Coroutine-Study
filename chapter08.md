## 1. 코루틴의 예외 전파

### (1) 코루틴에서 예외가 전파되는 방식

- 코루틴 실행 도중 예외 발생 시 취소 후 부모 코루틴으로 예외 전파
    
    → 부모 코루틴에서도 예외가 적절히 처리되지 않으면 취소 후 다시 상위 코루틴으로 예외 전파
    
    → 모두 예외가 적절히 처리되지 않아 루트 코루틴까지 예외 전파 및 취소
    
    → 하위의 모든 코루틴에 취소가 전파
    
    ![2025-03-08_23-45-39.jpg](attachment:9d427bf8-0878-4495-830b-96791e10f0f0:2025-03-08_23-45-39.jpg)
    

### (2) 예제로 알아보는 예외 전파

- 예외 발생
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(CoroutineName("Coroutine1")) {
        launch(CoroutineName("Coroutine3")) {
          throw Exception("예외 발생")
        }
        delay(100L)
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
      launch(CoroutineName("Coroutine2")) {
        delay(100L)
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
      delay(1000L)
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: 예외 발생
    */
    ```
    
    ![2025-03-08_23-53-40.jpg](attachment:1529ceb6-e61b-4823-a293-09039d52c29a:2025-03-08_23-53-40.jpg)
    
    - Coroutine2가 실행되어 printIn이 실행되지 못하고, 모든 코루틴이 취소됨

- 코루틴의 구조화는 큰 작업을 연관된 작은 작업으로 나누는 방식
    
    → 작은 작업에서 발생한 예외로 인해 큰 작업이 취소되면 안정성에서 문제가 생기게 됨
    
    ⇒ 예외 전파를 제한할 필요성 !!
    

## 2. 예외 전파 제한

### (1) Job 객체를 사용한 예외 전파 제한

- 새로운 Job 객체를 생성해 코루틴의 구조화를 깨는 방법으로 예외 전파 제한
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(CoroutineName("Parent Coroutine")) {
        launch(CoroutineName("Coroutine1") + Job()) { // 새로운 Job 객체를 만들어 Coroutine1에 연결
          launch(CoroutineName("Coroutine3")) {
            throw Exception("예외 발생")
          }
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine2")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      delay(1000L)
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: 예외 발
    [main @Coroutine2#4] 코루틴 실행
    */
    ```
    
    ![2025-03-09_00-12-24.jpg](attachment:65bdc717-3a55-434c-bef9-14d12c9fb727:2025-03-09_00-12-24.jpg)
    

- Job 객체를 생성하는 방법의 경우 : 취소 전파도 제한됨
    
    → 큰 작업에 취소가 요청되더라도 작은 작업은 취소되지 않으며 비동기 작업을 불안정하게 만들게 됨
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val parentJob = launch(CoroutineName("Parent Coroutine")) {
        launch(CoroutineName("Coroutine1") + Job()) {
          launch(CoroutineName("Coroutine3")) { // Coroutine3에서 예외 제거
            delay(100L)
            println("[${Thread.currentThread().name}] 코루틴 실행")
          }
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine2")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      delay(20L) // 코루틴들이 모두 생성될 때까지 대기
      parentJob.cancel() // Parent Coroutine에 취소 요청
      delay(1000L)
    }
    /*
    // 결과:
    [main @Coroutine1#3] 코루틴 실행
    [main @Coroutine3#5] 코루틴 실행
    */
    ```
    
    ![2025-03-09_00-13-54.jpg](attachment:cfaa1258-7a63-490d-a766-158ad65dc6eb:2025-03-09_00-13-54.jpg)
    

### (2) SupervisorJob 객체를 사용한 예외 전파 제한

- SupervisorJob : 자식 코루틴으로부터 예외를 전파받지 않는 특수한 Job 객체
    
    ```kotlin
    public fun SupervisorJob(parent: Job? = null) : CompletableJob = 
    		SupervisorJobImpl(parent)
    ```
    
    - parent 파라미터 없이 사용 시 루트 Job으로 생성 가능

- 루트 CoroutineContext에 SupervisorJob을 적용한 후 진행
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val supervisorJob = SupervisorJob()
      launch(CoroutineName("Coroutine1") + supervisorJob) {
        launch(CoroutineName("Coroutine3")) {
          throw Exception("예외 발생")
        }
        delay(100L)
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
      launch(CoroutineName("Coroutine2") + supervisorJob) {
        delay(100L)
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
      delay(1000L)
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: 예외 발생
    [main @Coroutine2#3] 코루틴 실행
    */
    ```
    
    ![2025-03-09_00-17-18.jpg](attachment:759edd54-eb42-4a8c-8608-bd0ed526c04e:2025-03-09_00-17-18.jpg)
    
    - 예외 전파 제한 성공
    - 그러나, SupervisorJob 객체가 기존 Job 객체와의 구조화를 깸

- 구조화를 깨지않고, SupervisorJob 객체의 parent 인자로 연결
    - SupervisorJob 객체는 생성된 Job 객체와 같이 자동으로 완료 처리가 되지 않음
        
        → complete() 명시적 처리 필요함
        
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      // supervisorJob의 parent로 runBlocking으로 생성된 Job 객체 설정
      val supervisorJob = SupervisorJob(parent = this.coroutineContext[Job])
      launch(CoroutineName("Coroutine1") + supervisorJob) {
        launch(CoroutineName("Coroutine3")) {
          throw Exception("예외 발생")
        }
        delay(100L)
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
      launch(CoroutineName("Coroutine2") + supervisorJob) {
        delay(100L)
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
      supervisorJob.complete() // supervisorJob 완료 처리
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: 예외 발생
    [main @Coroutine2#3] 코루틴 실행
    */
    ```
    
    ![2025-03-09_00-54-55.jpg](attachment:45af8e0c-a6c9-4930-84e8-e657f8031f6e:2025-03-09_00-54-55.jpg)
    

- SupervisorJob을 CoroutineScope와 함께 사용하는 방법
    - CoroutineScope의 CoroutineContext에 SupervisorJob 객체가 설정된다면, CoroutineScope의 자식 코루틴에서 발생하는 예외가 다른 자식 코루틴으로 전파되지 않음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val coroutineScope = CoroutineScope(SupervisorJob())
      
      coroutineScope.apply {
        launch(CoroutineName("Coroutine1")) {
          launch(CoroutineName("Coroutine3")) {
            throw Exception("예외 발생")
          }
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine2")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      delay(1000L)
    }
    /*
    // 결과:
    Exception in thread "DefaultDispatcher-worker-1" java.lang.Exception: 예외 발생
    [DefaultDispatcher-worker-1 @Coroutine2#3] 코루틴 실행
    */
    ```
    
    ![2025-03-09_01-04-07.jpg](attachment:b215edab-f022-4ccd-8c44-ead4c55bff06:2025-03-09_01-04-07.jpg)
    

- 흔히 하는 실수
    - 코루틴 빌더 함수의 context 인자로 SupervisorJob()을 넘기고, 생성된 코루틴의 하위에 자식 코루틴을 생성하는 경우
        
        → 코루틴 빌더 함수로부터 새로운 Job 객체가 생성되고, 해당 객체가 부모가 되는 구조가 생성됨
        
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(CoroutineName("Parent Coroutine") + SupervisorJob()) {
        launch(CoroutineName("Coroutine1")) {
          launch(CoroutineName("Coroutine3")) {
            throw Exception("예외 발생")
          }
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine2")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      delay(1000L)
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: 예외 발생
    */
    ```
    
    ![2025-03-09_01-06-20.jpg](attachment:849f1b98-804a-45b7-89e0-c09ce7a58a32:2025-03-09_01-06-20.jpg)
    

### (3) supervisorScope를 사용한 예외 전파 제한

- supervisorScope 함수
    - SupervisorJob 객체를 가진 CoroutineScope 객체를 생성
    - SupervisorJob 객체는 함수를 호출한 코루틴의 Job 객체를 부모로 가짐
        
        → 복잡한 설정 없이도 구조화를 깨지 않고 예외 전파를 제한할 수 있음 (자동 완료 처리)
        
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      supervisorScope {
        launch(CoroutineName("Coroutine1")) {
          launch(CoroutineName("Coroutine3")) {
            throw Exception("예외 발생")
          }
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine2")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: 예외 발생
    [main @Coroutine2#3] 코루틴 실행
    */
    ```
    
    ![2025-03-09_01-50-40.jpg](attachment:bec11d76-41fd-4a93-812c-4dcf834d762d:2025-03-09_01-50-40.jpg)
    

## 3. CoroutineExceptionHandler를 사용한 예외 처리

- 예외 전파 제한 방법이 아닌, 예외 처리하는 방법
- 코루틴 : CoroutineContext 구성 요소로 CoroutineExceptionHandler 예외 처리기 제공

### (1) CoroutineExceptionHandler 생성

- CoroutineExceptionHandler
    
    ```kotlin
    public inline fun CoroutineExceptionHandler(
    		crossinline handler : (CoroutineContext, Throwable) -> Unit
    ): CoroutineExceptionHandler
    ```
    
    - 예외를 처리하는 람다식인 handler를 매개 변수로 가짐
    - handler 람다식에 예외가 발생했을 때 어떤 동작을 할지 입력해 예외를 처리할 수 있음

### (2) CoroutineExceptionHandler 사용

- CoroutineExceptionHandler 객체를 생성한 후, CoroutineContext 객체의 구성 요소로 포함
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val exceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("[예외 발생] ${throwable}")
      }
      CoroutineScope(exceptionHandler).launch(CoroutineName("Coroutine1")) {
        throw Exception("Coroutine1에 예외가 발생했습니다")
      }
      delay(1000L)
    }
    /*
    // 결과:
    [예외 발생] java.lang.Exception: Coroutine1에 예외가 발생했습니다
    */
    ```
    

### (3) 처리되지 않은 예외만 처리하는 CoroutineExceptionHandler

- CoroutineExceptionHandler 객체는 처리되지 않은 예외만 처리함
    
    → 자식 코루틴이 부모 코루틴으로 예외를 전파하면, 자식 코루틴에서는 예외가 처리된 것으로 간주
    
    → 자식 코루틴에 설정된 CoroutineExceptionHandler 는 동작하지 않음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val exceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("[예외 발생] ${throwable}")
      }
      launch(CoroutineName("Coroutine1") + exceptionHandler) {
        throw Exception("Coroutine1에 예외가 발생했습니다")
      }
      delay(1000L)
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: Coroutine1에 예외가 발생했습니다
    */
    ```
    
    ![2025-03-09_02-17-42.jpg](attachment:3fd68f76-5b59-4c56-bc1e-daf33085e46b:2025-03-09_02-17-42.jpg)
    
    - 예외가 전파되어 Coroutine1에 설정된 exceptionHandler가 동작하지 않음

### (4) CoroutineExceptionHandler로 예외를 처리하기

- 마지막으로 예외를 전파받는 위치 (예외가 처리되는 위치)의 CoroutineExceptionHandler 객체만 작동
    
    → 공통 예외 처리기로서 동작 가능
    

1. Job과 CoroutineExceptionHandler 함께 사용
    - Job()을 호출해 새로운 루트 Job을 만들고, CoroutineExceptionHandler 객체 위치 설정
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val coroutineContext = Job() + CoroutineExceptionHandler { coroutineContext, throwable ->
            println("[예외 발생] ${throwable}")
          }
          launch(CoroutineName("Coroutine1") + coroutineContext) {
            throw Exception("Coroutine1에 예외가 발생했습니다")
          }
          delay(1000L)
        }
        /*
        // 결과:
        [예외 발생] java.lang.Exception: Coroutine1에 예외가 발생했습니다
        */
        ```
        
        ![2025-03-09_02-27-11.jpg](attachment:abf86bb0-4c0d-4b16-8699-a20e33c94484:2025-03-09_02-27-11.jpg)
        

1. SupervisorJob과 CoroutineExceptionHandler 함께 사용
    - SupervisorJob 객체 : 예외를 전파받지 않을 뿐, 예외 발생 정보를 자식 코루틴으로부터 전달받음
        
        → CoroutineExceptionHandler와 함께 설정되면 예외를 처리할 수 있음
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val exceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
            println("[예외 발생] ${throwable}")
          }
          val supervisedScope = CoroutineScope(SupervisorJob() + exceptionHandler)
          supervisedScope.apply {
            launch(CoroutineName("Coroutine1")) {
              throw Exception("Coroutine1에 예외가 발생했습니다")
            }
            launch(CoroutineName("Coroutine2")) {
              delay(100L)
              println("[${Thread.currentThread().name}] 코루틴 실행")
            }
          }
          delay(1000L)
        }
        /*
        // 결과:
        [예외 발생] java.lang.Exception: Coroutine1에 예외가 발생했습니다
        [DefaultDispatcher-worker-2] 코루틴 실행
        */
        ```
        
        ![2025-03-09_02-29-27.jpg](attachment:21138fd3-17b9-4cbf-9a56-9c28db144077:2025-03-09_02-29-27.jpg)
        

### (5) CoroutineExceptionHandler는 예외 전파를 제한하지 않음

- CoroutineExceptionHandler는 예외가 마지막으로 처리되는 위치에서 예외를 처리할 뿐, 제한하지 않음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val exceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("[예외 발생] ${throwable}")
      }
      launch(CoroutineName("Coroutine1") + exceptionHandler) {
        throw Exception("Coroutine1에 예외가 발생했습니다")
      }
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: Coroutine1에 예외가 발생했습니다
    */
    ```
    
    → 예외를 처리하지만 제한하지 않아, 프로세스가 비정상 종료됨
    

## 4. try-catch 문을 사용한 예외 처리 및 제한

### (1) try-catch 문을 사용해 코루틴 예외 처리하기

- 코루틴 예외 : 일반적으로 코틀린에서 예외를 처리하는 try-catch 문을 사용 가능
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(CoroutineName("Coroutine1")) {
        try {
          throw Exception("Coroutine1에 예외가 발생했습니다")
        } catch (e: Exception) {
          println(e.message)
        }
      }
      launch(CoroutineName("Coroutine2")) {
        delay(100L)
        println("Coroutine2 실행 완료")
      }
    }
    /*
    // 결과:
    Coroutine1에 예외가 발생했습니다
    Coroutine2 실행 완료
    */
    ```
    
    ![2025-03-09_02-39-39.jpg](attachment:98191a41-7129-4c0d-ade3-a3bf5060ccdd:2025-03-09_02-39-39.jpg)
    
    - Coroutine1에서 예외가 발생하지만, try-catch 문을 통해 처리되어 전파되지 않음

### (2) 코루틴의 예외를 잡지 못하는 코루틴 빌더 함수에 대한 try-catch 문

- 코루틴 빌더 함수에 try-catch 문 사용 시 코루틴에서 발생하는 예외를 잡을 수 없음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      try {
        launch(CoroutineName("Coroutine1")) {
          throw Exception("Coroutine1에 예외가 발생했습니다")
        }
      } catch (e: Exception) {
        println(e.message)
      }
      launch(CoroutineName("Coroutine2")) {
        delay(100L)
        println("Coroutine2 실행 완료")
      }
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: Coroutine1에 예외가 발생했습니다
    */
    ```
    
    - try-catch 문이 launch를 감싸는 형태
        
        → 람다식의 실행은 생성된 코루틴이 CoroutineDispatcher에 의해 스레드로 분배되는 시점
        

## 5. async의 예외 처리

### (1) async의 예외 노출

- async 코루틴 빌더 함수 : 결과값을 Deferred 객체로 감싸고 await 호출 시점에 결과값 노출
    
    → 코루틴 실행 도중 예외가 발생해 결과값이 없으면 await 호출 시점에서 예외가 노출됨
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      supervisorScope {
        val deferred: Deferred<String> = async(CoroutineName("Coroutine1")) {
          throw Exception("Coroutine1에 예외가 발생했습니다")
        }
        try {
          deferred.await()
        } catch (e: Exception) {
          println("[노출된 예외] ${e.message}")
        }
      }
    }
    /*
    // 결과:
    [노출된 예외] Coroutine1에 예외가 발생했습니다
    */
    ```
    
    → await 호출부에서 예외 처리가 될 수 있도록 해야 함
    

### (2) async의 예외 전파

- await 호출부에서의 예외 처리 뿐만이 아닌, 전파 처리 필요
- 전파 처리를 하지 않는 경우 :
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      async(CoroutineName("Coroutine1")) {
        throw Exception("Coroutine1에 예외가 발생했습니다")
      }
      launch(CoroutineName("Coroutine2")) {
        delay(100L)
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
    }
    /*
    // 결과:
    Exception in thread "main" java.lang.Exception: Coroutine1에 예외가 발생했습니다
    */
    ```
    
    ![2025-03-09_03-01-28.jpg](attachment:36c21fe3-9e86-4d02-9c76-efada69b0121:2025-03-09_03-01-28.jpg)
    
    - 부모 코루틴을 포함한 모든 코루틴이 취소됨

- supervisorScope를 사용해 예외 전파를 제한할 수 있음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      supervisorScope {
        async(CoroutineName("Coroutine1")) {
          throw Exception("Coroutine1에 예외가 발생했습니다")
        }
        launch(CoroutineName("Coroutine2")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
    }
    /*
    // 결과:
    [main @Coroutine2#3] 코루틴 실행
    */
    ```
    

## 6. 전파되지 않는 예외

### (1) 전파되지 않는 CancellationException

- 코루틴은 CancellationException 예외가 발생해도 부모 코루틴으로 전파되지 않음
    
    ```kotlin
    fun main() = runBlocking<Unit>(CoroutineName("runBlocking 코루틴")) {
      launch(CoroutineName("Coroutine1")) {
        launch(CoroutineName("Coroutine2")) {
          throw CancellationException()
        }
        delay(100L)
        println("[${Thread.currentThread().name}] 코루틴 실행")
      }
      delay(100L)
      println("[${Thread.currentThread().name}] 코루틴 실행")
    }
    /*
    // 결과:
    [main @runBlocking 코루틴#1] 코루틴 실행
    [main @Coroutine1#2] 코루틴 실행
    */
    ```
    
    ![2025-03-09_03-27-31.jpg](attachment:870af8db-b6ad-498a-a4a8-b9d5e5d4449a:2025-03-09_03-27-31.jpg)
    

### (2) 코루틴 취소 시 사용되는 JobCancellationException

- Job 객체에 대해 cancel 함수를 호출
    
    → CancellationException의 서브 클래스인 JobCancellationException을 발생시켜 코루틴을 취소함
    
    → 특정 코루틴만 취소하는 데 사용됨
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val job = launch {
        delay(1000L) // 1초간 지속
      }
      job.invokeOnCompletion { exception ->
        println(exception) // 발생한 예외 출력
      }
      job.cancel() // job 취소
    }
    /*
    // 결과:
    kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job=StandaloneCoroutine{Cancelled}@7494e528
    */
    ```
    

### (3)  withTimeOut 사용해 코루틴의 실행 시간 제한하기

- 코루틴 라이브러리 : 제한 시간을 두고 실행할 수 있도록 만드는 withTimeOut 함수 제공
    
    → 작업이 주어진 시간 내에 완료되지 않으면 CancellationException의 서브 클래스인 TimeoutCancellationException을 발생시킴 → 해당 코루틴만 취소
    
    ```kotlin
    fun main() = runBlocking<Unit>(CoroutineName("Parent Coroutine")) {
      launch(CoroutineName("Child Coroutine")) {
        withTimeout(1000L) { // 실행 시간을 1초로 제한
          delay(2000L) // 2초의 시간이 걸리는 작업
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      delay(2000L)
      println("[${Thread.currentThread().name}] 코루틴 실행")
    }
    /*
    // 결과:
    [main @Parent Coroutine#1] 코루틴 실행
    */
    ```
