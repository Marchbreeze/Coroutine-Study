## 0. 구조화된 동시성

- 구조화된 동시성 (Structured Concurrency)
    - 비동기 작업을 구조화함으로써 비동기 프로그래밍을 보다 안정적이고 예측할 수 있게 만드는 원칙

- 코루틴 : 비동기 작업인 코루틴을 부모-자식 관계로 구조화함으로써 안전하게 관리 & 제어
- 특징 :
    1. 부모 코루틴의 실행 환경이 자식 코루틴에게 상속
    2. 작업을 제어하는 데 사용
    3. 부모 코루틴이 취소되면 자식 코루틴도 취소
    4. 부모 코루틴은 자식 코루틴이 완료될 때까지 대기
    5. CoroutineScope를 사용해 코루틴이 실행되는 범위 제한

## 1. 실행 환경 상속

### (1) 부모 코루틴의 실행 환경 상속

- 부모 코루틴이 자식 코루틴을 생성하면, 부모 코루틴의 CoroutineContext가 자식 코루틴에게 전달됨
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val coroutineContext = newSingleThreadContext("MyThread") + CoroutineName("CoroutineA")
      launch(coroutineContext){ // 부모 코루틴 생성
        println("[${Thread.currentThread().name}] 부모 코루틴 실행")
        launch {  // 자식 코루틴 생성
          println("[${Thread.currentThread().name}] 자식 코루틴 실행")
        }
      }
    }
    /*
    // 결과:
    [MyThread @CoroutineA#2] 부모 코루틴 실행
    [MyThread @CoroutineA#3] 자식 코루틴 실행
    */
    ```
    
    - CoroutineContext가 설정되지 않은 자식 코루틴에서도 동일한 코루틴 이름을 확인할 수 있음

### (2) 실행 환경 덮어씌우기

- 자식 코루틴을 생성하는 코루틴 빌더 함수로 새로운 CoroutineContext 객체가 전달되면 덮어씌울 수 있음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val coroutineContext = newSingleThreadContext("MyThread") + CoroutineName("ParentCoroutine")
      launch(coroutineContext){ // 부모 코루틴 생성
        println("[${Thread.currentThread().name}] 부모 코루틴 실행")
        launch(CoroutineName("ChildCoroutine")) {  // 자식 코루틴 생성
          println("[${Thread.currentThread().name}] 자식 코루틴 실행")
        }
      }
    }
    /*
    // 결과:
    [MyThread @ParentCoroutine#2] 부모 코루틴 실행
    [MyThread @ChildCoroutine#3] 자식 코루틴 실행
    */
    ```
    
    ![2025-02-27_16-48-09.jpg](attachment:c0e6e15a-9bfd-4fd3-acb9-0cf8b9182f50:2025-02-27_16-48-09.jpg)
    

### (3) 상속되지 않는 Job

- 다른 CoroutineContext 구성 요소들과 다르게, Job 객체는 상속되지 않고 코루틴 빌더 함수 호출시 새롭게 생성됨
    - launch나 async를 포함한 모든 코루틴 빌더 함수는 호출 때마다 코루틴 추상체인 Job 객체를 새로 생성
    - 상속을 받게 되면 개별 코루틴의 제어가 어려워지기 때문
    
    ```kotlin
    fun main() = runBlocking<Unit> { // 부모 코루틴 생성
      val runBlockingJob = coroutineContext[Job] // 부모 코루틴의 CoroutineContext로부터 부모 코루틴의 Job 추출
      launch { // 자식 코루틴 생성
        val launchJob = coroutineContext[Job] // 자식 코루틴의 CoroutineContext로부터 자식 코루틴의 Job 추출
        if (runBlockingJob === launchJob) {
          println("runBlocking으로 생성된 Job과 launch로 생성된 Job이 동일합니다")
        } else {
          println("runBlocking으로 생성된 Job과 launch로 생성된 Job이 다릅니다")
        }
      }
    }
    /*
    // 결과:
    runBlocking으로 생성된 Job과 launch로 생성된 Job이 다릅니다
    */
    ```
    

### (4) 구조화에 사용되는 Job

- 자식 코루틴이 부모 코루틴으로부터 전달받은 Job 객체는 코루틴 구조화에 활용됨
    
    ![2025-02-27_16-58-46.jpg](attachment:d11c4c3e-3681-4557-9c67-1f2b0c1ae886:2025-02-27_16-58-46.jpg)
    
    - 생성된 자식 Job 객체
        - 내부에 정의된 parent 프로퍼티를 통해 부모 코루틴의 Job 객체에 대한 참조를 가짐
    - 부모 코루틴의 Job 객체
        - Sequence 타입의 children 프로퍼티를 통해 자식 코루틴의 Job에 대한 참조를 가짐
    - 양방향 참조
        
        ![2025-02-27_17-02-42.jpg](attachment:0e6a81cb-9bc8-4ee6-964c-d6a8bee1b3f7:2025-02-27_17-02-42.jpg)
        
        - 부모 코루틴이 없는 (parent null) 코루틴 = 루트 코루틴

- parent 프로퍼티와 children 프로퍼티의 참조
    
    ```kotlin
    fun main() = runBlocking<Unit> { // 부모 코루틴
      val parentJob = coroutineContext[Job] // 부모 코루틴의 CoroutineContext로부터 부모 코루틴의 Job 추출
      launch { // 자식 코루틴
        val childJob = coroutineContext[Job] // 자식 코루틴의 CoroutineContext로부터 자식 코루틴의 Job 추출
        println("1. 부모 코루틴과 자식 코루틴의 Job은 같은가? ${parentJob === childJob}")
        println("2. 자식 코루틴의 Job이 가지고 있는 parent는 부모 코루틴의 Job인가? ${childJob?.parent === parentJob}")
        println("3. 부모 코루틴의 Job은 자식 코루틴의 Job을 참조를 가지는가? ${parentJob?.children?.contains(childJob)}")
      }
    }
    /*
    // 결과:
    1. 부모 코루틴과 자식 코루틴의 Job은 같은가? false
    2. 자식 코루틴의 Job이 가지고 있는 parent는 부모 코루틴의 Job인가? true
    3. 부모 코루틴의 Job은 자식 코루틴의 Job을 참조를 가지는가? true
    */
    ```
    

## 2. 코루틴의 구조화와 작업 제어

- 코루틴 구조화 : 하나의 큰 비동기 작업을 작은 비동기 작업으로 나눌 때 발생 → 코루틴 관리 및 제어 안정성
- ex.
    
    ![2025-02-27_17-05-55.jpg](attachment:2ccdd427-0a89-439d-99c5-bd12b0841be4:2025-02-27_17-05-55.jpg)
    

### (1) 취소의 전파

- 특정 코루틴이 취소가 되면 하위의 모든 코루틴이 취소됨
- 취소는 자식 코루틴 방향으로만 전파되며, 부모 코루틴으로는 전파되지 않음
    
    ![2025-02-27_17-07-27.jpg](attachment:7a849a66-b8bb-45a3-b08c-6af64c8c9f03:2025-02-27_17-07-27.jpg)
    
- 작업 중간에 부모 코루틴이 취소되면, 자식 코루틴의 작업이 더 이상 진행될 필요가 없어짐 (리소스 낭비)
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val parentJob = launch(Dispatchers.IO){ // 부모 코루틴 생성
        val dbResultsDeferred: List<Deferred<String>> = listOf("db1","db2","db3").map {
          async { // 자식 코루틴 생성
            delay(1000L) // DB로부터 데이터를 가져오는데 걸리는 시간
            println("${it}으로부터 데이터를 가져오는데 성공했습니다")
            return@async "[${it}]data"
          }
        }
        val dbResults: List<String> = dbResultsDeferred.awaitAll() // 모든 코루틴이 완료될 때까지 대기
    
        println(dbResults) // 화면에 표시
      }
      parentJob.cancel() // 부모 코루틴에 취소 요청
    }
    /*
    // 결과: 출력되지 않음
    */
    ```
    

### (2) 완료 의존성

- 부모 코루틴 : 모든 자식 코루틴이 실행 완료되야 완료될 수 있음 = 자식 코루틴에 대해 완료 의존성을 가짐
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      val parentJob = launch { // 부모 코루틴 실행
        launch { // 자식 코루틴 실행
          delay(1000L) // 1초간 대기
          println("[${getElapsedTime(startTime)}] 자식 코루틴 실행 완료")
        }
        println("[${getElapsedTime(startTime)}] 부모 코루틴이 실행하는 마지막 코드")
      }
      parentJob.invokeOnCompletion { // 부모 코루틴이 종료될 시 호출되는 콜백 등록
        println("[${getElapsedTime(startTime)}] 부모 코루틴 실행 완료")
      }
    }
    /*
    // 결과:
    [지난 시간: 3ms] 부모 코루틴이 실행하는 마지막 코드
    [지난 시간: 1019ms] 자식 코루틴 실행 완료
    [지난 시간: 1019ms] 부모 코루틴 실행 완료
    */
    ```
    
    - invokeOnCompletion 함수 : 실행 완료 혹은 취소 완료 시점에서 동작

- 부모 코루틴의 시점 : 자식 코루틴 완료 전까지 “실행 완료 중” 상태
    
    ![2025-02-27_17-38-49.jpg](attachment:d950e3a7-4910-4ec3-941a-804a034f818a:2025-02-27_17-38-49.jpg)
    
    - 실행 완료 중
        - isActive = true
        - isCancelled = false
        - isCompleted = false
    - 실행 완료 중 상태는 실행 중 상태와 완전히 같은 Job 상태값은 가짐 (구분 어려움)

## 3. CoroutineScope 사용한 코루틴 관리

- CoroutineScope 객체 :
    - 자신의 범위 내에서 생성된 코루틴들에게 실행 환경을 제공하고, 이들의 실행 범위를 관리하는 역할

### (1) CoroutineScope 생성

- 방법 1. CoroutineScope Interface
    - 코루틴의 실행 환경인 CoroutineContext를 가진 단순한 인터페이스
        
        ```kotlin
        public interface CoroutineScope {
        		public val coroutineContext: CoroutineContext
        }
        ```
        
    - 해당 인터페이스를 구현한 구체적인 클래스로 활용
        
        ```kotlin
        class CustomCoroutineScope : CoroutineScope {
          override val coroutineContext: CoroutineContext = Job() + newSingleThreadContext("CustomScopeThread")
        }
        
        fun main() {
          val coroutineScope = CustomCoroutineScope() // CustomCoroutineScope 인스턴스화
          coroutineScope.launch {
            delay(100L) // 100밀리초 대기
            println("[${Thread.currentThread().name}] 코루틴 실행 완료")
          }
          Thread.sleep(1000L) // 코드 종료 방지
        }
        
        /*
        // 결과:
        [CustomScopeThread @coroutine#1] 코루틴 실행 완료
        */
        ```
        

- 방법 2. CoroutineScope 함수
    - CoroutineScope 함수 : CoroutineContext를 인자로 입력받아 CoroutineScope 객체를 생성
        
        ```kotlin
        public fun CoroutineScope(context: CoroutineContext): CoroutineScope = 
        		ContextScope(if (context[Job] != null) context else context + Job())
        ```
        
        - 인자로 입력된 CoroutineContext에 Job 객체가 포함되어 있지 않으면 새로운 Job 객체 생성
    - 활용
        
        ```kotlin
        fun main() {
          val coroutineScope = CoroutineScope(Dispatchers.IO)
          coroutineScope.launch {
            delay(100L) // 100밀리초 대기
            println("[${Thread.currentThread().name}] 코루틴 실행 완료")
          }
          Thread.sleep(1000L)
        }
        /*
        // 결과:
        [DefaultDispatcher-worker-1 @coroutine#1] 코루틴 실행 완료
        */
        ```
        

### (2) 코루틴에게 실행 환경 제공

- CoroutineScope가 코루틴에게 실행 환경을 제공하는 방식
    - launch 함수
        
        ```kotlin
        public fun CoroutineScope.launch(
        	context: CoroutineContext = EmptyCoroutineContext, 
        	start: CoroutineStart = CoroutineStart.DEFAULT, 
        	block: suspend CoroutineScope.() -> Unit
        ): Job {
          //...
        }
        ```
        
        1. 수신 객체인 CoroutineScope로부터 CoroutineContext 객체를 제공받음
        2. 제공받은 CoroutineContext 객체에 launch 함수의 context 인자로 넘어온 CoroutineContext를 더함
        3. 생성된 CoroutineContext에 코루틴 빌더 함수가 호출되어 새로 생성되는 Job을 더함
            
            (CoroutineContext를 통해 전달되는 Job 객체는 새로 생성되는 Job 객체의 부모)
            

- CoroutineScope로부터 실행 환경 상속
    - CoroutineScope 수신 객체를 람다식 내부에서 this를 통해 접근할 수 있음
        
        ```kotlin
        fun main() {
          val newScope = CoroutineScope(CoroutineName("MyCoroutine") + Dispatchers.IO)
          newScope.launch(CoroutineName("LaunchCoroutine")) { // this: CoroutineScope
            this.coroutineContext // LaunchCoroutine의 실행 환경을 CoroutineScope을 통해 접근
            this.launch { // CoroutineScope으로부터 LaunchCoroutine의 실행 환경을 제공 받아 코루틴 실행
              // 작업 실행
            }
          }
          Thread.sleep(1000L)
        }
        ```
        
        - this는 생략할 수 있음

### (3) CoroutineScope에 속한 코루틴의 범위

- 코루틴 빌더 람다식에서 수신 객체로 제공되는 CoroutineScope 객체
    
    코루틴 빌더로 생성된 코루틴 + 람다식 내에서 CoroutineScope 객체를 사용해 실행되는 모든 코루틴 포함
    
    ![2025-02-27_17-57-33.jpg](attachment:60680d41-9360-424c-945e-2f7e12f2df06:2025-02-27_17-57-33.jpg)
    

- CoroutineScope를 새로 생성해 기존 CoroutineScope 범위에서 벗어나기
    
    ![2025-02-27_17-58-56.jpg](attachment:960b60ff-6cde-4f1b-b9b2-d55e564cb658:2025-02-27_17-58-56.jpg)
    
    - CoroutineScope 함수가 호출되면 새로운 Job 객체가 생성 → 기존의 계층 구조를 따르지 않음
        
        ⇒ 코루틴의 구조화를 깨기 때문에 권장하지 않음
        

### (4) CoroutineScope 취소

- CoroutineScope 인터페이스는 확장 함수로 cancel 함수를 지원함
    - CoroutineScope 객체의 범위에 속한 모든 코루틴을 취소
    - 내부 구현
        
        ```kotlin
        public fun CoroutineScope.cancel(cause: CancellationException? = null) {
        	val job = coroutineContext[Job] ? : error("Scope cannot be cancelled because it does not have a job: $this")
        	job.cancel(cause)
        }
        ```
        
        - coroutineContext 프로퍼티를 통해 Job 객체에 접근한 후 cancel 호출
    - ex.
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          launch(CoroutineName("Coroutine1")) {
            launch(CoroutineName("Coroutine3")) {
              delay(100L)
              println("[${Thread.currentThread().name}] 코루틴 실행 완료")
            }
            launch(CoroutineName("Coroutine4")) {
              delay(100L)
              println("[${Thread.currentThread().name}] 코루틴 실행 완료")
            }
            this.cancel() // Coroutine1의 CoroutineScope에 cancel 요청
          }
        
          launch(CoroutineName("Coroutine2")) {
            delay(100L)
            println("[${Thread.currentThread().name}] 코루틴 실행 완료")
          }
        }
        /*
        // 결과:
        [main @Coroutine2#3] 코루틴 실행 완료
        */
        ```
        
        - CoroutineScope 객체의 범위에 속한 1, 3, 4는 실행 도중 취소 & 속하지 않는 2는 진행

### (5) CoroutineScope 활성화 상태 확인

- CoroutineScope 활성화 상태 확인하는 isActive 확장 프로퍼티
    - 내부 구현
        
        ```kotlin
        	public val CoroutineScope.isActive: Boolean
        		get() = coroutineContext[Job]?.isActive ?: true
        ```
        
        - coroutineContext 프로퍼티를 통해 Job 객체에 접근한 후 isActive 확인

## 4. 구조화와 Job

### (1) runBlocking과 루트 Job

- runBlocking 코루틴 하위에 코루틴이 생성되면 runBlocking 코루틴을 부모로 하는 자식 코루틴이 생성됨
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(CoroutineName("Coroutine1")) { // Coroutine1 실행
        launch(CoroutineName("Coroutine3")) { // Coroutine3 실행
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine4")) { // Coroutine4 실행
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      launch(CoroutineName("Coroutine2")) { // Coroutine2 실행
        launch(CoroutineName("Coroutine5")) { // Coroutine5 실행
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      delay(1000L)
    }
    ```
    
    ![2025-02-27_22-32-07.jpg](attachment:187250f0-4ceb-4192-8dbf-52cc7a1f8052:2025-02-27_22-32-07.jpg)
    

### (2) Job 구조화 깨기

- CoroutineScope 사용해 구조화 깨기
    
    ```kotlin
    fun main() = runBlocking<Unit> { // 루트 Job 생성
      val newScope = CoroutineScope(Dispatchers.IO) // 새로운 루트 Job 생성
      newScope.launch(CoroutineName("Coroutine1")) { // Coroutine1 실행
        launch(CoroutineName("Coroutine3")) { // Coroutine3 실행
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine4")) { // Coroutine4 실행
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      newScope.launch(CoroutineName("Coroutine2")) { // Coroutine2 실행
        launch(CoroutineName("Coroutine5")) { // Coroutine5 실행
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
    }
    /*
    // 결과:
    Process finished with exit code 0
    */
    ```
    
    - 루트 Job이지만, runBlocking이 아닌 newScope를 통해 구조화 진행
    - runBlocking이 다른 코루틴들의 완료를 기다리지 않고 메인 스레드 사용을 종료해서 프로세스 종료
        
        → 출력 X
        

- Job 사용해 구조화 깨기
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val newRootJob = Job() // 루트 Job 생성
      launch(CoroutineName("Coroutine1") + newRootJob) {
        launch(CoroutineName("Coroutine3")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine4")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      launch(CoroutineName("Coroutine2") + newRootJob) {
        launch(CoroutineName("Coroutine5")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      delay(1000L)
    }
    /*
    // 결과:
    [main @Coroutine3#4] 코루틴 실행
    [main @Coroutine4#5] 코루틴 실행
    [main @Coroutine5#6] 코루틴 실행
    */
    ```
    
    - runBlocking 코루틴이 아닌 newRootJob을 부모로 하위 코루틴이 생성됨

### (3) Job으로 일부 코루틴만 취소되지 않게 만들기

- ex. Coroutine5만 계층 구조를 끊어 취소되지 않도록 만들기
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val newRootJob = Job() // 새로운 루트 Job 생성
      launch(CoroutineName("Coroutine1") + newRootJob) {
        launch(CoroutineName("Coroutine3")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
        launch(CoroutineName("Coroutine4")) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      launch(CoroutineName("Coroutine2") + newRootJob) {
        launch(CoroutineName("Coroutine5") + Job()) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
      delay(50L) // 모든 코루틴이 생성될 때까지 대기
      newRootJob.cancel() // 새로운 루트 Job 취소
      delay(1000L)
    }
    /*
    // 결과:
    [main @Coroutine5#6] 코루틴 실행
    */
    ```
    
    ![2025-02-27_22-41-02.jpg](attachment:f3c399f5-be34-4c51-a741-829168a2c8fc:2025-02-27_22-41-02.jpg)
    

### (4) 생성된 Job의 부모를 명시적으로 설정하기

- Job 생성 함수
    
    ```kotlin
    public fun Job(parent: Job? = null): CompletableJob = JobImpl(parent)
    ```
    
    - Job()을 통해 객체를 생성하는 경우, parent 프로퍼티가 null이 되어 루트 Job이 생성됨
    - parent 인자에 Job 객체를 넘기면 새로운 자식 Job을 생성할 수 있음
        - 다만 이렇게 Job 객체를 생성하는 경우, 문제 발생 가능

### (5) 생성된 Job은 자동으로 실행 완료되지 않음

- launch 함수를 통해 생성된 Job 객체
    - 더 이상 실행할 코드가 없고, 모든 자식 코루틴이 실행 완료되면 자동으로 실행을 종료함
- Job 생성 함수를 통해 생성된 Job 객체
    - 자동으로 실행 완료되지 않고, 명시적으로 complete를 호출해야 함

- ex.
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(CoroutineName("Coroutine1")) {
        val coroutine1Job = this.coroutineContext[Job] // Coroutine1의 Job
        val newJob = Job(parent = coroutine1Job)
        launch(CoroutineName("Coroutine2") + newJob) {
          delay(100L)
          println("[${Thread.currentThread().name}] 코루틴 실행")
        }
      }
    }
    /*
    // 결과:
    [main @Coroutine2#3] 코루틴 실행
    // 프로세스 종료 로그가 출력되지 않는다.
    */
    ```
    
    - newJob이 자동 종료되지 않음 → 부모 코루틴들이 모두 ‘실행 완료 중’ 상태에서 대기
