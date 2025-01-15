## 1. JVM 프로세스와 단일 스레드

- 일반적인 메인 로직 : 실행 → JVM이 프로세스 시작 → 메인 스레드 생성 → main 함수 내부의 코드 실행

- JVM의 프로세스
    - JVM의 프로세스는 사용자 스레드가 모두 종료될 때 프로세스를 종료하며, 메인 스레드는 그 중 하나
    - 기본적으로 메인 스레드를 단일 스레드로 해서 실행
    - 예외로 인해 메인 스레드가 강제로 종료되면 프로세스도 강제 종료됨

- 단일 스레드 (Single-Thread)
    - 스레드는 하나의 작업을 수행할 때 다른 작업을 동시에 수행 불가능
    - ex. 네트워크 요청 후 대기 : UI 작업을 멈추고 입력을 전달받지 못하는 상태로 멈춤

## 2. 멀티 스레드 프로그래밍

- 멀티 스레드 프로그래밍
    - 독립적으로 분할된 작업을 서로 다른 스레드로 할당해서 처리 (병렬 처리)
        
        ![2025-01-15_02-03-49.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/b435d7c5-a359-4e26-857d-95e197888d52/2025-01-15_02-03-49.jpg)
        

### (1) Thread 클래스

- Thread 클래스의 run 함수를 오버라이드해서 작성
- ex.
    
    ```kotlin
    class ExampleThread : Thread() {
      override fun run() {
        println("[${Thread.currentThread().name}] 새로운 스레드 시작")
        Thread.sleep(2000L) // 2초 동안 대기
        println("[${Thread.currentThread().name}] 새로운 스레드 종료")
      }
    }
    
    fun main() {
      println("[${Thread.currentThread().name}] 메인 스레드 시작")
      ExampleThread().start()
      Thread.sleep(1000L) // 1초 동안 대기
      println("[${Thread.currentThread().name}] 메인 스레드 종료")
    }
    ```
    
    ![2025-01-15_02-38-14.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/e5a2b7df-dcfc-44aa-aa4b-cd246d49ce61/2025-01-15_02-38-14.jpg)
    
    - 스레드는 각각 하나의 작업을 진행할 수 있으므로 동시에 진행
    - 두 스레드는 모두 사용자 스레드 → 메인이 아닌 새로운 스레드가 종료될 때 JVM 프로세스 종료

- 사용자 스레드 vs 데몬 스레드
    1. 사용자 스레드 (User Thread)
        - 기본적으로 애플리케이션이 실행될 때 생성되는 일반적인 스레드
        - 메인 프로그램이 종료되더라도 자신이 맡은 작업이 끝날 때까지 실행을 유지
        - 우선도가 높은 스레드 → 사용자 스레드가 하나라도 실행 중이라면 JVM은 종료되지 않음
    2. 데몬 스레드 (Daemon Thread)
        - 백그라운드에서 실행되는 스레드로, 주로 사용자 스레드가 필요로 하는 서비스를 제공하기 위해 존재
        - 우선도가 낮은 스레드 → 모든 사용자 스레드가 종료되면 자동으로 종료
        - ex. 주로 가비지 컬렉터(Garbage Collector), JVM 메모리 관리 또는 백그라운드 작업에서 사용
        - 설정 방법
            
            ```kotlin
            ExampleThread().apply {
              isDaemon = true
            }
            ```
            

- thread 클래스를 따로 생성하지 않아도 제공되는 함수 사용 가능
    
    ```kotlin
    fun main() {
      thread(isDaemon = false) {
        Thread.sleep(2000L)
      }
      Thread.sleep(1000L) 
    }
    ```
    

- Thread 클래스의 한계
    1.  Thread 클래스를 상속한 클래스를 인스턴스화해 실행할 때마다 매번 새로운 스레드가 생성
        - 스레드는 생성 비용이 비싸기 때문에 성능적으로 좋지 않음
    2. 스레드 생성과 관리에 대한 책임이 개발자에게 있음
        - 프로그램의 복잡성이 증가하며, 실수로 인해 오류나 메모리 누수를 발생시킬 가능성 존재

### (2) Executor 프레임워크

- 스레드풀 (Thread Pool)
    - 스레드의 집합 - Thread 객체로 직접 스레드를 생성하지 않고도 작업을 스레드 풀에서 실행

- Executor 프레임워크
    - 스레드풀을 관리하고 사용자로부터 요청받은 작업을 각 스레드에 할당하는 시스템
    - 스레드풀에 속한 스레드의 생성과 관리 및 작업 분배에 대한 책임을 Executor 프레임워크가 담담
        
        →  개발자 : 스레드풀에 속한 스레드 개수를 설정하고, 해당 스레드풀을 관리하는 서비스에 작업을 제출
        

1. 스레드풀을 관리하는 객체 반환
    
    ```kotlin
    val executorService: ExecutorService = Executors.newFixedThreadPool(2)
    ```
    

1. ExecutorService 객체에서 제공하는 submit 함수를 통해 스레드풀에 작업을 제출
    
    ```kotlin
    fun main() {
      val executorService: ExecutorService = Executors.newFixedThreadPool(2)
    
      // 작업1 제출
      executorService.submit {
        Thread.sleep(1000L) 
      }
      // 작업2 제출
      executorService.submit {
        Thread.sleep(1000L) 
      }
      // 작업3 제출
      executorService.submit {
        Thread.sleep(1000L) 
      }
    
      // ExecutorService 종료
      executorService.shutdown()
    }
    /*
    // 결과:
    [pool-1-thread-1][지난 시간: 4ms] 작업1 시작
    [pool-1-thread-2][지난 시간: 4ms] 작업2 시작
    [pool-1-thread-1][지난 시간: 1009ms] 작업1 완료
    [pool-1-thread-2][지난 시간: 1011ms] 작업2 완료
    [pool-1-thread-1][지난 시간: 1012ms] 작업3 시작
    [pool-1-thread-1][지난 시간: 2016ms] 작업3 완료
    */
    ```
    
    - 스레드풀에 2개의 스레드만 생성 → 작업 1,2가 병렬적으로 처리된 후 종료되면 작업3 할당됨
    - ExecutorService 객체 구성 : 스레드풀 & 작업 대기열 (할당받은 작업을 적재)
        
        ![2025-01-15_16-49-35.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/205afbd9-ed65-49f5-a28f-922459011127/2025-01-15_16-49-35.jpg)
        
- 한계 = 스레드 블로킹
    - 스레드 블로킹 : 스레드가 아무것도 하지 못하고 사용될 수 없는 상태에 있는 것
    - ex.
        - 여러 스레드가 동기화 블록에 동시에 접근하는 경우 - 하나의 스레드만 동기화 블록에 접근이 허용
        - 뮤텍스나 세마포어로 인해 공유되는 자원에 접근할 수 있는 스레드가 제한되는 경우
        - ExecutorService 객체에 제출한 작업에서 결과를 전달받을 때는 언젠가 올지 모르는 값을 기다리는데 Future 객체를 사용 → 결과값이 반환될 때까지 블로킹됨
            
            ```kotlin
            fun main() {
              val executorService: ExecutorService = Executors.newFixedThreadPool(2)
              val future: Future<String> = executorService.submit<String> {
                Thread.sleep(2000)
                return@submit "작업 1완료"
              }
            
              val result = future.get() // 메인 스레드가 블로킹 됨
              println(result)
              executorService.shutdown()
            }
            ```
            
            ![2025-01-15_17-08-21.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/639045a5-8d3b-460a-95d8-9691a5ead407/2025-01-15_17-08-21.jpg)
            

- 간단한 작업에서는 콜백을 사용하거나 체이닝 함수를 사용해서 스레드 블로킹을 피할 수 있음
    
    ```kotlin
    fun main() {
      val startTime = System.currentTimeMillis()
      val executor = Executors.newFixedThreadPool(2)
    
      // CompletableFuture 생성 및 비동기 작업 실행
      val completableFuture = CompletableFuture.supplyAsync({
        Thread.sleep(1000L) // 1초간 대기
        return@supplyAsync "결과" // 결과 반환
      }, executor)
    
      // 비동기 작업 완료 후 결과 처리를 위한 체이닝 함수 등록
      completableFuture.thenAccept { result ->
        println("[${getElapsedTime(startTime)}] $result 처리") // 결과 처리 출력
      }
    
      // 비동기 작업 실행 도중 다른 작업 실행
      println("[${getElapsedTime(startTime)}] 다른 작업 실행")
    
      executor.shutdown()
    }
    ```
    

## 3. 코루틴

- 코루틴 : 작업 단위 코루틴을 통해 스레드 블로킹 문제 해결
    - 작업 단위 코루틴 : 스레드에서 작업 실행 도중 일시 중단할 수 있는 작업 단위
        - 작업 일시 중단 시, 더 이상 스레드 사용이 필요하지 않으므로 스레드의 사용 권한을 양보
        - 일시 중단된 코루틴을 재개 시점에 다시 스레드에 할당돼 실행

- 기존의 스레드
    
    ![2025-01-15_17-34-08.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/6e0f3bf5-9f35-490b-84d3-898aeae2ec81/2025-01-15_17-34-08.jpg)
    
- 코루틴
    
    ![2025-01-15_17-34-26.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/edfd69d1-6c01-4d0c-9269-1bae8a4e3915/d32c7cdc-1336-46f7-b35b-b120bba3de13/2025-01-15_17-34-26.jpg)
    

- 코루틴 = 경량 스레드
    - 스레드에 비해 생성과 전환 비용이 적게 들고 스레드에 자유롭게 뗐다 붙였다 할 수 있어 작업을 생성하고 전환하는 데 필요한 리소스와 시간이 매우 줄어듦
        - 1000개의 스레드 생성 및 실행 -> 약 500~2000ms
        - 1000개의 코루틴 생성 및 실행 -> 약 5~50ms
    - 구조화된 동시성을 통해 비동기 작업을 안전하게 만들고, 예외 처리를 효과적으로 처리할 수 있도록 하며,  코루틴이 실행 중인 스레드를 손쉽게 전환

- 코루틴 실행
    
    ```kotlin
    fun main() = runBlocking<Unit> { // this: CoroutineScope
      println("[${Thread.currentThread().name}] 실행")
      launch {
        println("[${Thread.currentThread().name}] 실행")
      }
      launch {
        println("[${Thread.currentThread().name}] 실행")
      }
    }
    /*
    // 실행 결과:
    [main @coroutine#1] 실행
    [main @coroutine#2] 실행
    [main @coroutine#3] 실행
    */
    ```
    

- 코루틴 이름 추가
    
    ```kotlin
    fun main() = runBlocking<Unit>(context = CoroutineName("Main")) {
      println("[${Thread.currentThread().name}] 실행")
      launch(context = CoroutineName("Coroutine1")) {
        println("[${Thread.currentThread().name}] 실행")
      }
      launch(context = CoroutineName("Coroutine2")) {
        println("[${Thread.currentThread().name}] 실행")
      }
    }
    /*
    // 실행 결과:
    [main @Main#1] 실행
    [main @Coroutine1#2] 실행
    [main @Coroutine2#3] 실행
    */
    ```
