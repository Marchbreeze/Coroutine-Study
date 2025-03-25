## 1. 서브루틴과 코루틴

### (1) 루틴과 서브루틴

- 루틴 : 특정한 일을 처리하기 위한 일련의 명령 (함수, 메서드)
- 서브루틴 : 함수 내에서 함수가 호출되는 경우
    
    ```kotlin
    fun routine() {
      routineA() // routineA는 routine의 서브루틴이다.
      routineB() // routineB는 routine의 서브루틴이다.
    }
    ```
    

- 서브루틴은 한 번 호출되면 끝까지 실행됨
    
    → 루틴을 실행하던 스레드는 서브루틴을 실행하는 데 사용되어 다른 작업을 할 수 없음
    
    ![2025-03-26_04-10-14.jpg](attachment:3bfe555c-9fba-4bec-aad6-58aba5544431:2025-03-26_04-10-14.jpg)
    

### (2) 서브루틴과 코루틴의 차이

- 코루틴 : 함께(Co) 실행되는 루틴 → 서로 간에 스레드 사용을 양보하며 함께 실행됨
    
    ![2025-03-26_04-17-22.jpg](attachment:d8dabac8-1bca-4f58-b835-9530e86d7798:2025-03-26_04-17-22.jpg)
    

- 코루틴의 협력적 동작을 위해, 코루틴이 작업하지 않는 시점에 스레드 사용 권한을 양보하고 일시 중단해야 함

## 2. 코루틴의 스레드 양보

- 스레드 양보의 주체 = 코루틴
    - 스레드의 코루틴을 할당해 실행되도록 만드는 주체는 CoroutineDispatcher 객체이지만,
    - CoroutineDispatcher는 코루틴이 스레드를 양보하도록 강제할 수 없음

- 코루틴이 스레드를 양보하려면 직접 양보를 위한 함수를 호출해야 함 (그렇지 않으면 스레드를 점유)

### (1) delay - 일시 중단 함수를 통해 알아보는 스레드 양보

- 작업을 일정 시간동안 일시 중단하는 경우, delat 일시 중단 함수 사용 가능
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      launch {
        delay(1000L) // 1초 동안 코루틴 일시 중단
        println("[${getElapsedTime(startTime)}] 코루틴 실행 완료") // 지난 시간과 함께 "코루틴 실행 완료" 출력
      }
    }
    /*
    // 결과:
    [지난 시간: 1012ms] 코루틴 실행 완료
    */
    ```
    

- 코루틴 스레드 양보의 강력함은 코루틴이 여러 개 실행되는 환경에서 드러남
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      repeat(10) { repeatTime ->
        launch {
          delay(1000L) // 1초 동안 코루틴 일시 중단
          println("[${getElapsedTime(startTime)}] 코루틴${repeatTime} 실행 완료") // 지난 시간과 함께 "코루틴 실행 완료" 출력
        }
      }
    }
    /*
    // 결과:
    [지난 시간: 1014ms] 코루틴0 실행 완료
    [지난 시간: 1015ms] 코루틴1 실행 완료
    [지난 시간: 1015ms] 코루틴2 실행 완료
    [지난 시간: 1015ms] 코루틴3 실행 완료
    [지난 시간: 1015ms] 코루틴4 실행 완료
    [지난 시간: 1015ms] 코루틴5 실행 완료
    [지난 시간: 1015ms] 코루틴6 실행 완료
    [지난 시간: 1016ms] 코루틴7 실행 완료
    [지난 시간: 1016ms] 코루틴8 실행 완료
    [지난 시간: 1016ms] 코루틴9 실행 완료
    */
    ```
    
    - 각 코루틴은 메인 스레드 상에서 실행되지만, 시작하자마자 delay 함수로 메인 스레드 사용을 양보
        
        → 하나의 코루틴이 시작된 후 바로 다음 코루틴에게 양보
        

- Thread.Sleep() : 스레드를 양보하지 않는 지연 함수
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      repeat(10) { repeatTime ->
        launch {
          Thread.sleep(1000L) // 1초 동안 스레드 블로킹(코루틴의 스레드 점유 유지)
          println("[${getElapsedTime(startTime)}] 코루틴${repeatTime} 실행 완료") // 지난 시간과 함께 "코루틴 실행 완료" 출력
        }
      }
    }
    /*
    // 결과:
    [지난 시간: 1007ms] 코루틴0 실행 완료
    [지난 시간: 2014ms] 코루틴1 실행 완료
    [지난 시간: 3016ms] 코루틴2 실행 완료
    [지난 시간: 4019ms] 코루틴3 실행 완료
    [지난 시간: 5024ms] 코루틴4 실행 완료
    [지난 시간: 6030ms] 코루틴5 실행 완료
    [지난 시간: 7036ms] 코루틴6 실행 완료
    [지난 시간: 8041ms] 코루틴7 실행 완료
    [지난 시간: 9045ms] 코루틴8 실행 완료
    [지난 시간: 10051ms] 코루틴9 실행 완료
    */
    ```
    
    - 대기 시간동안 스레드를 블로킹(점유)하여 순차적으로 실행됨

### (2) join & await - 동작 방식 자세히 알아보기

- Job의 join 함수, Deferred의 await 함수 호출 시, 해당 함수를 호출한 코루틴은 스레드를 일시 중단 및 양보
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val job = launch {
        println("1. launch 코루틴 작업이 시작됐습니다")
        delay(1000L) // 1초간 대기
        println("2. launch 코루틴 작업이 완료됐습니다")
      }
      println("3. runBlocking 코루틴이 곧 일시 중단 되고 메인 스레드가 양보됩니다")
      job.join() // job 내부의 코드가 모두 실행될 때까지 메인 스레드 일시 중단
      println("4. runBlocking이 메인 스레드에 분배돼 작업이 다시 재개됩니다")
    }
    
    /*
    // 결과:
    3. runBlocking 코루틴이 곧 일시 중단 되고 메인 스레드가 양보됩니다
    1. launch 코루틴 작업이 시작됐습니다
    2. launch 코루틴 작업이 완료됐습니다
    4. runBlocking이 메인 스레드에 분배돼 작업이 다시 재개됩니다
    */
    ```
    
    ![2025-03-26_04-34-43.jpg](attachment:9a11a956-1775-424e-9b54-5404cb0cd698:2025-03-26_04-34-43.jpg)
    
    1. 처음에는 runBlocking 코루틴이 점유
    2. launch 함수 호출 이후에도 runBlocking 코루틴이 계속해서 메인 스레드를 점유하기 때문에, launch 코루틴은 실행 대기 상태에 머무름
    3. 이후 job.join()이 실행되면 메인 스레드를 양보받을 수 있음
    4. 자유로워진 메인 스레드에 launch 코루틴이 보내져 실행됨
        
        runBlocking 코루틴은 job.join()에 의해 launch 코루틴이 실행 완료될 때까지 재개되지 못함
        
    5. launch 코루틴이 실행이 완료되면 runBlocking 코루틴이 재개

### (3) yield - 함수 호출해 스레드 양보하기

- 특수한 상황에서 스레드 양보를 직접 호출해야 하는 경우 yield 사용
    - ex.
        
        ```kotlin
        //AS-IS
        fun main() = runBlocking<Unit> {
          val job = launch {
            while (this.isActive) {
              println("작업 중")
            }
          }
          delay(100L) // 100밀리초 대기(스레드 양보)
          job.cancel() // 코루틴 취소
        }
        
        /*
        // 결과:
        ...
        작업 중
        작업 중
        작업 중
        ...
        */
        ```
        
        ![2025-03-26_04-47-54.jpg](attachment:fd020866-8aee-4f38-9ec3-103bd000a01a:2025-03-26_04-47-54.jpg)
        
        ```kotlin
        //TO-BE
        fun main() = runBlocking<Unit> {
          val job = launch {
            while (this.isActive) {
              println("작업 중")
              yield() // 스레드 양보
            }
          }
          delay(100L)
          job.cancel()
        }
        /*
        // 결과:
        ...
        작업 중
        작업 중
        작업 중
        */
        ```
        
        → while 문 내부에서 yield()를 호출해 명시적으로 스레드를 양보해 job.cancel() 호출
        

## 3. 코루틴의 실행 스레드

### (1) 코루틴의 실행 스레드는 고정이 아니다

- 코루틴이 일시 중단 후 재개되면, CoroutineDispatcher 객체는 재개된 코루틴을 다시 스레드에 할당함
    
    → 코루틴이 일시 중단 전에 실행되던 스레드와 다를 수 있음
    
    - ex. delay
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val dispatcher = newFixedThreadPoolContext(2, "MyThread")
          launch(dispatcher) {
            repeat(2) {
              println("[${Thread.currentThread().name}] 코루틴 실행이 일시 중단 됩니다") // launch 코루틴을 실행 중인 스레드를 출력한다.
              delay(100L) // delay 함수를 통해 launch 코루틴을 100밀리초간 일시 중단한다.
              println("[${Thread.currentThread().name}] 코루틴 실행이 재개 됩니다") // launch 코루틴이 재개되면 코루틴을 실행 중인 스레드를 출력한다.
            }
          }
        }
        /*
        // 결과:
        [MyThread-1 @coroutine#2] 코루틴 실행이 일시 중단 됩니다
        [MyThread-2 @coroutine#2] 코루틴 실행이 재개 됩니다
        [MyThread-2 @coroutine#2] 코루틴 실행이 일시 중단 됩니다
        [MyThread-1 @coroutine#2] 코루틴 실행이 재개 됩니다
        */
        ```
        

### (2) 스레드를 양보하지 않으면 실행 스레드가 바뀌지 않는다

- 코루틴의 실행 스레드가 바뀌는 시점 = 코루틴이 재개될 때
    
    → 코루틴이 스레드 양보를 하지 않아 일시 중단될 일이 없다면, 실행 스레드가 바뀌지 않음
    
    - ex. Thread.sleep
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val dispatcher = newFixedThreadPoolContext(2, "MyThread")
          launch(dispatcher) {
            repeat(2) {
              println("[${Thread.currentThread().name}] 스레드를 점유한채로 100밀리초간 대기합니다")
              Thread.sleep(100L) // 스레드를 점유한 채로 100 밀리초 동안 대기
              println("[${Thread.currentThread().name}] 점유한 스레드에서 마저 실행됩니다")
            }
          }
        }
        /*
        // 결과:
        [MyThread-1 @coroutine#2] 스레드를 점유한채로 100밀리초간 대기합니다
        [MyThread-1 @coroutine#2] 점유한 스레드에서 마저 실행됩니다
        [MyThread-1 @coroutine#2] 스레드를 점유한채로 100밀리초간 대기합니다
        [MyThread-1 @coroutine#2] 점유한 스레드에서 마저 실행됩니다
        */
        ```
