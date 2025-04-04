# 1. 공유 상태를 사용하는 코루틴의 문제와 데이터 동기화

## (1) 가변 변수를 사용할 때의 문제점

- 코드의 안정성을 높이기 위해서는 가변 변수 대신 불변 변수를 사용해야 함
- 그러나, 스레드 또는 코루틴 간 데이터를 전달하거나 공유된 자원을 사용하는 경우
    
    → 가변 변수를 사용해 상태를 공유하고 업데이트해야 함
    

- 멀티 스레드 환경에서 복수의 코루틴이 가변 변수를 공유하고 변경하는 경우
    
    ```kotlin
    var count = 0
    
    fun main() = runBlocking<Unit> {
      withContext(Dispatchers.Default) {
        repeat(10_000) {
          launch {
            count += 1
          }
        }
      }
      println("count = ${count}")
    }
    
    /*
    // 결과1:
    count = 9062 // 매번 다른 값이 나온다.
    // 결과2:
    count = 9019 // 매번 다른 값이 나온다.
    // 결과3:
    count = 8644 // 매번 다른 값이 나온다.
    */
    ```
    
    → 문제 발생
    

1. 메모리 가시성 문제 (Memory Visibility)
    - 스레드가 변수를 읽는 메모리 공간에 관한 문제 (하드웨어의 메모리 구조와 연관)
    - 스레드가 변수를 변경시킬 때 메인 메모리가 아닌 CPU Cache를 사용할 경우
        
        → CPU Cache의 값이 메인 메모리에 전파되는 데 약간의 시간이 걸려 데이터 불일치 문제 발생
        
        → 다른 스레드에서 해당 변수를 읽을 때 변수가 변경됨을 확인하지 못하고 이전 값 활용
        

1. 경쟁 상태 문제 (Race Condition)
    - 2개의 스레드가 동시에 값을 읽고 업데이트시키면 같은 연산이 두 번 발생
        
        → 하나의 연산이 손실됨
        

## (2) JVM의 메모리 공간이 하드웨어 메모리 구조와 연결되는 방식

- JVM의 메모리 공간
    
    ![2025-04-04_02-15-57.jpg](attachment:05c5452a-55ed-483d-9f37-a0ed8b76977a:2025-04-04_02-15-57.jpg)
    
    - 스택 영역 (Stack Area)
        - 스레드마다 각자의 메모리 공간을 가지고 있음
        - 원시 타입의 데이터가 저장되거나, 힙 영역에 저장된 객체에 대한 참조 (주소 값)이 저장됨
    - 힙 영역 (Heap Area)
        - JVM 스레드에서 공통으로 사용되는 메모리 공간
        - 객체나 배열과 같은, 크고 복잡한 데이터가 저장됨

- 하드웨어의 메모리 구조
    
    ![2025-04-04_02-17-46.jpg](attachment:08bd9660-5cb9-4bcf-adf1-e9bdebb8d7f2:2025-04-04_02-17-46.jpg)
    
    - CPU 레지스터, CPU 캐시 메모리, 메인 메모리 영역으로 구성됨
    - 데이터 조회 시 공통 영역인 메인 메모리까지 가지 않고 각 CPU 캐시 메모리에서 데이터를 조회
        
        → 메모리 액세스 속도 향상
        

- JVM의 메모리 공간이 하드웨어 메모리 구조와 연결되는 방식
    
    ![2025-04-04_02-19-43.jpg](attachment:cea564cd-6c86-49d9-8439-e53808b11bcc:2025-04-04_02-19-43.jpg)
    
    - 하드웨어 메모리 구조는 JVM의 스택 / 힙 영역 구분 X
        
        → 멀티 스레드 환경에서 공유 상태를 사용할 때 메모리 가시성 & 경쟁 상태 문제 발생
        

## (3) 메모리 가시성 문제 해결

- 공유 상태에 대한 메모리 가시성 문제
    - 하나의 스레드가 다른 스레드가 변경된 상태를 확인하지 못하는 상황
    - 서로 다른 CPU에서 실행되는 스레드에서 공유 상태를 조회하고 업데이트할 때 생기는 문제

- 발생하는 로직
    1. 공유 상태에 초기에 메인 메모리에 저장
    2. 하나의 스레드가 이 공유 상태를 읽어오면, 해당 CPU는 공유 상태를 CPU 캐시 메모리에 저장
    3. 공유 상태를 변경해도, CPU 캐시 메모리에 저장된 값은 플러시가 일어나기 전까지 메인 메모리로 전파되지 않음
    4. 다른 CPU에서 실행되는 스레드에서 메인 메모리의 공유 상태 접근 시 변경되지 않은 값을 활용하게 됨

### @Volatile

- @Volatile 어노테이션이 설정된 변수를 읽고 쓸 때는 CPU 캐시 메모리를 사용하지 않도록 설정
    
    ```kotlin
    @Volatile
    var count = 0
    
    fun main() = runBlocking<Unit> {
      withContext(Dispatchers.Default) {
        repeat(10_000) {
          launch {
            count += 1
          }
        }
      }
      println("count = ${count}")
    }
    /*
    // 결과:
    count = 9122
    */
    ```
    

## (4) 경쟁 상태 문제 해결

- 변수가 메인 메모리에만 저장되더라도, 여전히 여러 스레드가 동시에 메인 메모리의 값에 접근할 수 있음
    - 각 launch 코루틴이 Dispatchers.Default객체를 사용해 실행됨 → 중복된 연산 발생

### Mutex

- 공유 변수의 변경 가능 지점을 임계 영역(Critical Section)으로 만들어 동시 접근 제한

- Mutex 객체 : 코틀린에서 코루틴에 대한 임계 영역 생성
    - lock 일시 중단 함수가 호출되면 락이 획득
    - 이후 해당 Mutex 객체에 대해 unlock이 호출돼 해제될 때까지 다른 코루틴이 임계 영역 접근 불가
    
    ```kotlin
    var count = 0
    val mutex = Mutex()
    
    fun main() = runBlocking<Unit> {
      withContext(Dispatchers.Default) {
        repeat(10_000) {
          launch {
            mutex.lock() // 임계 영역 시작 지점
            count += 1
            mutex.unlock() // 임계 영역 종료 지점
          }
        }
      }
      println("count = ${count}")
    }
    /*
    // 결과:
    count = 10000
    */
    ```
    
- 우려점
    - 락이 해제되지 않으면 해당 임계 영역은 다른 스레드에서 접근이 불가능해져 문제 발생 가능
    - lock-unlock 쌍 대신 withLock 일시 중단 함수를 사용해서 안전하게 설정
    
    ```kotlin
    var count = 0
    val mutex = Mutex()
    
    fun main() = runBlocking<Unit> {
      withContext(Dispatchers.Default) {
        repeat(10_000) {
          launch {
            mutex.withLock {
              count += 1
            }
          }
        }
      }
      println("count = ${count}")
    }
    ```
    

### ReetrantLock

- 코루틴에서 사용할 때, 같은 뮤텍스 기능을 하는 ReetrantLock 객체 대신 Mutex 객체를 사용하는 이유
    1. Mutex
        - 코루틴이 lock 함수를 호출했는데 이미 Mutex 객체에 lock이 걸려있으면
            
            → 코루틴은 기존의 락이 해제될 때까지 스레드를 양보하고 즉시 중단
            
        - 코루틴이 일시 중단되는 동안 스레드가 블로킹되지 않도록 해서 스레드에 다른 작업이 진행되도록 함
        - 이후 기존의 락이 해제되면 코루틴이 재개돼 락을 획득
    2. Reentrant
        - 코루틴이 lock 함수를 호출했는데 이미 Reentrant 객체에 lock이 걸려있으면
            
            → 코루틴은 기존의 락이 해제될 때까지 스레드를 블로킹하고 기다림 (양보 X)
            

- 안전한 임계 영역을 만들 수 있는 것은 동일함
    
    ```kotlin
    var count = 0
    val reentrantLock = ReentrantLock()
    
    fun main() = runBlocking<Unit> {
      withContext(Dispatchers.Default) {
        repeat(10_000) {
          launch {
            reentrantLock.lock() // 스레드를 블록하고 기존의 락이 해제될 때까지 기다림
            count += 1
            reentrantLock.unlock()
          }
        }
      }
      println("count = ${count}")
    }
    ```
    

### 전용 스레드 사용

- 경쟁 상태 : 복수의 스레드가 공유 상태에 동시에 접근할 수 있기 때문에 발생
    
    → 공유 상태에 접근할 때 하나의 전용 스레드만 사용하도록 강제해서 동시 접근 문제 해결
    

- newSingleThreadContext 함수를 사용해 단일 스레드로 구성된 CoroutineDispatcher 객체 생성 및 활용
    
    ```kotlin
    var count = 0
    val countChangeDispatcher = newSingleThreadContext("CountChangeThread")
    
    fun main() = runBlocking<Unit> {
      withContext(Dispatchers.Default) {
        repeat(10_000) {
          launch { // count 값을 변경 시킬 때만 사용
            increaseCount()
          }
        }
      }
      println("count = ${count}")
    }
    
    suspend fun increaseCount() = coroutineScope {
      withContext(countChangeDispatcher) {
        count += 1
      }
    }
    ```
    
    - launch 코루틴이 Dispatchers.Default를 통해 백그라운드 스레드에서 실행되더라도, 일시 중단 함수가 호출되면 launch 코루틴의 실행 스레드가 전환되어 동시 접근을 방지함

### Atomic Variable

- 아토믹 변수
    - 동기화 없이도 원자적(atomic)으로 값을 읽고 수정할 수 있도록 설계된 변수
    - 변수에 대한 모든 읽기 및 쓰기 연산이 하나의 원자적 단위로 수행되기 때문에, 다른 스레드가 중간에 개입할 수 없게 됨
    - 원자적 연산은 불가분 연산으로, 한 스레드의 연산이 완료되기 전에 다른 스레드가 연산에 개입할 수 없음

- 간단한 타입의 경우
    
    ```kotlin
    var count = AtomicInteger(0)
    
    fun main() = runBlocking<Unit> {
      withContext(Dispatchers.Default) {
        repeat(10_000) {
          launch {
            count.getAndUpdate {
              it + 1 // count값 1 더하기
            }
            // 또는 count.incrementAndGet()
          }
        }
      }
      println("count = ${count}")
    }
    ```
    

- 복잡한 객체의 경우 : AtomicReference
    
    ```kotlin
    data class Counter(val name: String, val count: Int)
    val atomicCounter = AtomicReference(Counter("MyCounter", 0)) // 원자성 있는 Counter 만들기
    
    fun main() = runBlocking<Unit> {
      withContext(Dispatchers.Default) {
        repeat(10_000) {
          launch {
            atomicCounter.getAndUpdate {
              it.copy(count = it.count + 1) // MyCounter의 count값 1 더하기
            }
          }
        }
      }
      println(atomicCounter.get())
    }
    /*
    // 결과:
    Counter(name=MyCounter, count=10000)
    */
    ```
    

- 한계
    - ReentrantLock 객체와 같이, 이미 연산을 실행 중인 경우 스레드를 블로킹하고 대기

# 2. CoroutineStart의 다양한 옵션

- 코루틴에 실행 옵션을 주기 위해 코루틴 빌더 함수의 start 인자로 CoroutineStart 옵션을 전달할 수 있음
    
    ```kotlin
    public fun CoroutineScope. launch (
    		context: CoroutineContext = EmptyCoroutineContext, 
    		start: CoroutineStart = CoroutineStart.DEFAULT, 
    		block: suspend CoroutineScope.() -> Unit
    ): Job
    ```
    
    - 지연 시작을 위한 CoroutineStart.LAZY 외에도 다양한 옵션 존재

### (1) CoroutineStart.Default

- CoroutineStart.Default : 기본값 (start 인자로 아무런 값이 전달되지 않은 경우)
- 코루틴 빌더 함수를 호출하면 :
    
    → 즉시 생성된 코루틴의 실행을 CoroutineDispatcher 객체에 예약하고, 호출한 코루틴은 계속해서 실행됨
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch {
        println("작업1")
      }
      println("작업2")
    }
    /*
    // 결과:
    작업2
    작업1
    */
    ```
    
    - launch 코루틴의 실행이 즉시 예약됐지만, runBlocking 코루틴이 메인 스레드를 양보하지 않고 계속해서 실행되므로 작업2가 우선적으로 실행된 후 launch가 실행됨

### (2) CoroutineStart.ATOMIC

- CoroutineStart.ATOMIC : 코루틴의 실행 대기 상태에서 취소를 방지하기 위한 옵션
    - 일반적인 코루틴 : 실행되기 전에 취소되면 실행되지 않고 종료됨
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val job = launch {
            println("작업1")
          }
          job.cancel() // 실행 대기 상태의 코루틴에 취소 요청
          println("작업2")
        }
        /*
        // 결과:
        작업2
        */
        ```
        
    - start 인자로 CoroutineStart.ATOMIC 옵션 적용 시, 실행 대기 상태에서 취소되지 않음
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val job = launch(start = CoroutineStart.ATOMIC) {
            println("작업1")
          }
          job.cancel() // 실행 대기 상태의 코루틴에 취소 요청
          println("작업2")
        }
        /*
        // 결과:
        작업2
        작업1
        */
        ```
        

### (3) CoroutineStart.UNDISPATCHED

- CoroutineStart.UNDISPATCHED : CoroutineDispatcher 객체의 작업 대기열을 거치지 않고 즉시 실행
    - 적용 시
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          launch(start = CoroutineStart.UNDISPATCHED) {
            println("작업1")
          }
          println("작업2")
        }
        /*
        // 결과:
        작업1
        작업2
        */
        ```
        
        - launch 코루틴이 즉시 호출자의 스레드인 메인 스레드에 할당돼 실행
            
            → launch 코루틴이 즉시 메인 스레드를 점유해 작업1이 먼저 실행되고 이후 작업2가 실행됨
            
            ![2025-04-04_13-03-55.jpg](attachment:c54edd23-0a17-41ef-8a50-c924946d8727:2025-04-04_13-03-55.jpg)
            
- 처음 코루틴 빌더가 실행되었을 때에만 CoroutineDispatcher 객체를 거치지 않음
    
    → 코루틴 내부에서 일시 중단 후 재개되는 경우에는 작업 대기열을 거치게 됨
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(start = CoroutineStart.UNDISPATCHED) {
        println("일시 중단 전에는 CoroutineDispatcher을 거치지 않고 즉시 실행된다")
        delay(100L)
        println("일시 중단 후에는 CoroutineDispatcher을 거쳐 실행된다")
      }
    }
    ```
    

# 3. 무제한 디스패처

### (1) Unconfiend Dispatcher

- 무제한 디스패처 (Unconfiend Dispatcher)
    - 코루틴을 자신을 실행시킨 스레드에서 즉시 실행하도록 만드는 디스패처
    - 호출된 스레드가 무엇이든지 상관없기 때문에 실행 스레드가 제한되지 않음

- 동작 원리
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      launch(Dispatchers.Unconfined) { // Dispatchers.Unconfined를 사용해 실행되는 코루틴
        println("launch 코루틴 실행 스레드: ${Thread.currentThread().name}") // launch 코루틴이 실행되는 스레드 출력
      }
    }
    /*
    // 결과:
    launch 코루틴 실행 스레드: main @coroutine#2
    */
    ```
    
    - launch 코루틴은 자신을 실행시킨 runBlocking 코루틴이 있는 메인 스레드에서 실행됨

### (2) 무제한 디스패처의 특징

1. 코루틴이 자신을 생성한 스레드에서 즉시 실행
    - CoroutineStart.UNDISPATCHED 옵션의 동작과 유사
        
        ```kotlin
        fun main() = runBlocking<Unit>(Dispatchers.IO) {
          println("runBlocking 코루틴 실행 스레드: ${Thread.currentThread().name}") // runBlocking 코루틴이 실행되는 스레드 출력
          launch(Dispatchers.Unconfined) { // Dispatchers.Unconfined를 사용해 실행되는 코루틴
            println("launch 코루틴 실행 스레드: ${Thread.currentThread().name}") // launch 코루틴이 실행되는 스레드 출력
          }
        }
        /*
        // 결과:
        runBlocking 코루틴 실행 스레드: DefaultDispatcher-worker-1 @coroutine#1
        launch 코루틴 실행 스레드: DefaultDispatcher-worker-1 @coroutine#2
        */
        ```
        
        - runBlocking 코루틴은 DIspatchers.IO의 공유 스레드풀의 스레드 중 하나 사용
        - 내부의 launch 코루틴은 runBlocking 코루틴이 사용하던 스레드를 그대로 사용
        
    - 무제한 디스패처를 사용하는 코루틴은 현재 자신을 실행한 스레드를 즉시 점유해 실행
        
        ![2025-04-04_14-47-43.jpg](attachment:ee640fa9-2330-404a-8845-3e2e75863b77:2025-04-04_14-47-43.jpg)
        
    - 무제한 디스패처를 사용한 코루틴은 스레드 스위칭 없이 실행됨
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          println("작업1")
          launch(Dispatchers.Unconfined) { // Dispatchers.Unconfined를 사용해 실행되는 코루틴
            println("작업2")
          }
          println("작업3")
        }
        /*
        // 결과:
        작업1
        작업2
        작업3
        */
        ```
        

1. 중단 시점 이후의 재개는 코루틴을 재개하는 스레드에서 실행
    - 일시 중단 전까지만 자신을 실행시킨 스레드에서 실행됨
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          launch(Dispatchers.Unconfined) {
            println("일시 중단 전 실행 스레드: ${Thread.currentThread().name}")
            delay(100L)
            println("일시 중단 후 실행 스레드: ${Thread.currentThread().name}")
          }
        }
        /*
        // 결과:
        일시 중단 전 실행 스레드: main
        일시 중단 후 실행 스레드: kotlinx.coroutines.DefaultExecutor
        */
        ```
        
    - 어떤 스레드가 코루틴을 재개시키는지 예측하기는 어렵기 때문에, 일반적인 상황에서 무제한 디스패처를 사용하면 비동기 작업이 불안정해짐 → 테스트 등의 특수 상황에서만 사용하도록 권장
    
    - CoroutineStart.UNDISPATCHED의 경우 :
        - 일시 중단 후 재개 시 자신이 실행되는 CoroutineDispatcher 객체를 사용해서 재개됨
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          println("runBlocking 코루틴 실행 스레드: ${Thread.currentThread().name}") // runBlocking 코루틴이 실행되는 스레드 출력
          launch(start = CoroutineStart.UNDISPATCHED) { // CoroutineStart.UNDISPATCHED가 적용된 코루틴
            println("[CoroutineStart.UNDISPATCHED] 코루틴이 시작 시 사용하는 스레드: ${Thread.currentThread().name}")
            delay(100L)
            println("[CoroutineStart.UNDISPATCHED] 코루틴이 재개 시 사용하는 스레드: ${Thread.currentThread().name}")
          }.join()
          launch(context = Dispatchers.Unconfined) { // Dispatchers.Unconfined를 사용해 실행되는 코루틴
            println("[Dispatchers.Unconfined] 코루틴이 시작 시 사용하는 스레드: ${Thread.currentThread().name}")
            delay(100L)
            println("[Dispatchers.Unconfined] 코루틴이 재개 시 사용하는 스레드: ${Thread.currentThread().name}")
          }.join()
        }
        /*
        // 결과:
        runBlocking 코루틴 실행 스레드: main @coroutine#1
        [CoroutineStart.UNDISPATCHED] 코루틴이 시작 시 사용하는 스레드: main @coroutine#2
        [CoroutineStart.UNDISPATCHED] 코루틴이 재개 시 사용하는 스레드: main @coroutine#2
        [Dispatchers.Unconfined] 코루틴이 시작 시 사용하는 스레드: main @coroutine#3
        [Dispatchers.Unconfined] 코루틴이 재개 시 사용하는 스레드: kotlinx.coroutines.DefaultExecutor @coroutine#3
        */
        ```
        

# 4. 코루틴의 동작 방식

### (1) Continuation Passing Style

- 코루틴은 코드를 실행하는 도중 일시 중단하고 다른 작업으로 전환한 후, 필요한 시점에 재개하는 기능 지원
    
    → 코루틴이 일시 중단을 하고 재개하기 위해서는 코루틴의 실행 정보가 저장되어 전달되어야 함
    

- CPS (Continuation Passing Style)
    - 코틀린이 코루틴의 실행 정보를 저장하고 전달하는 프로그래밍 방식
    - Continuation(이어서 실행해야 하는 작업)을 전달하는 스타일

- Continuation 객체
    - 코루틴의 일시 중단 시점에 코루틴의 실행 상태를 저장하며, 다음에 실행해야 할 작업 정보가 포함됨
    - 코루틴 재개 시 코루틴의 상태를 복원하고 이어서 작업을 진행할 수 있도록 함
        
        → 코루틴 라이브러리의 고수준 API : Continuation 객체를 캡슐화
        

### (2) Continuation과 코루틴의 일시 중단 및 재개

- 코루틴에서 일시 중단 발생 시 Continuation 객체에 실행 정보가 저장
- 일시 중단된 코루틴은 Continuation 객체에 대해 resume 함수가 호출되어야 재개됨

- Continuation 객체를 CancellableContinuation 타입으로 제공하는 suspendCancellableCoroutine 함수로 확인
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      println("runBlocking 코루틴 일시 중단 호출")
      suspendCancellableCoroutine<Unit> { continuation: CancellableContinuation<Unit> ->
        println("일시 중단 시점의 runBlocking 코루틴 실행 정보: ${continuation.context}")
      }
      println("일시 중단된 코루틴이 재개되지 않아 실행되지 않는 코드")
    }
    /*
    // 결과:
    runBlocking 코루틴 일시 중단 호출
    일시 중단 시점의 runBlocking 코루틴 실행 정보: [CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@51cdd8a, BlockingEventLoop@d44fc21]
    
    프로세스가 종료되지 않는다.
    */
    ```
    
    - Continuation 객체에 대해 재개가 호출되지 않아 runBlocking 코루틴이 재개되지 못함
- 해결 방법
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      println("runBlocking 코루틴 일시 중단 호출")
      suspendCancellableCoroutine<Unit> { continuation: CancellableContinuation<Unit> ->
        println("일시 중단 시점의 runBlocking 코루틴 실행 정보: ${continuation.context}")
        continuation.resume(Unit) // 코루틴 재개 호출
      }
      println("runBlocking 코루틴 재개 후 실행되는 코드")
    }
    /*
    // 결과:
    runBlocking 코루틴 일시 중단 호출
    일시 중단 시점의 runBlocking 코루틴 실행 정보: [BlockingCoroutine{Active}@551aa95a, BlockingEventLoop@35d176f7]
    runBlocking 코루틴 재개 후 실행되는 코드
    
    Process finished with exit code 0
    */
    ```
    

- delay 일시 중단 함수의 내부에서도 활용됨
    
    ```kotlin
    public suspend fun delay(timeMillis: Long) {
    	if (timeMillis <= 0) return // don't delay 
    	return suspendCancellableCoroutine @cs { cont:
    		CancellableContinuation<Unit> ->
    			// if timeMillis == LongM. AX_VALUE then just wait forever like awaitCancellation, don't schedule.
    			if (timeMillis < Long.MAX_VALUE) {
    				cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont) // timeMillis 이후에 Continuation을 재개시키기
    			}
    		}
    }
    
    ```
    

### (3) 다른 작업으로부터 결과 수신해 코루틴 재개

- 코루틴 재개 시 다른 작업으로부터 결과를 수신받아야하는 경우
    
    → suspendCancellableCoroutine 함수의 타입 인자에 결과로 반환되는 타입을 입력
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val result = suspendCancellableCoroutine<String> { continuation: CancellableContinuation<String> -> // runBlocking 코루틴 일시 중단 시작
          thread { // 새로운 스레드 생성
            Thread.sleep(1000L) // 1초간 대기
            continuation.resume("실행 결과") // runBlocking 코루틴 재개
          }
        }
      println(result) // 코루틴 재개 시 반환 받은 결과 출력
    }
    /*
    // 결과:
    실행 결과
    
    Process finished with exit code 0
    */
    ```
