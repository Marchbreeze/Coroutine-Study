## 1. join을 사용한 코루틴 순차 처리

- 코루틴 빌더 함수 : runBlocking & launch → 코루틴을 만들고, 코루틴을 추상화한 Job 객체를 생성함

- 순차 처리가 이루어지지 않는 경우
    
    ![2025-02-05_18-16-33.jpg](attachment:bbb063f1-4a11-4317-92da-34d034a3c3a1:2025-02-05_18-16-33.jpg)
    
    - 순차적으로 진행되지 않아, 토큰 업데이트가 완료되기 전에 네트워크 요청이 진행됨 (병렬 진행)

- join 함수 활용
    - JobA 코루틴이 완료된 이후에 JobB 코루틴이 실행되어야 하는 경우
        
        → JobB 코루틴이 실행되기 전에 JobA 코루틴에 join 함수 호출
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val updateTokenJob = launch(Dispatchers.IO) {
            println("[${Thread.currentThread().name}] 토큰 업데이트 시작")
            delay(100L)
            println("[${Thread.currentThread().name}] 토큰 업데이트 완료")
          }
          updateTokenJob.join() // updateTokenJob이 완료될 때까지 일시 중단
          val networkCallJob = launch(Dispatchers.IO) {
            println("[${Thread.currentThread().name}] 네트워크 요청")
          }
        }
        /*
        // 결과:
        [DefaultDispatcher-worker-1 @coroutine#2] 토큰 업데이트 시작
        [DefaultDispatcher-worker-1 @coroutine#2] 토큰 업데이트 완료
        [DefaultDispatcher-worker-1 @coroutine#3] 네트워크 요청
        */
        ```
        
    - join 함수는 join을 호출한 코루틴만 일시 중단시킴

## 2. joinAll을 사용한 코루틴 순차 처리

- joinAll : 복수의 코루틴의 실행이 모두 끝날 때까지 호출부의 코루틴 일시 중단
    
    ```kotlin
    public suspend fun joinAll(vararg jobs: Job): Unit = jobs.forEach {
    	it.join()
    }
    ```
    

- joinAll 함수 활용해 여러 코루틴의 순차 처리 예시
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val convertImageJob1: Job = launch(Dispatchers.Default) {
        Thread.sleep(1000L) // 이미지 변환 작업 실행 시간
        println("[${Thread.currentThread().name}] 이미지1 변환 완료")
      }
      val convertImageJob2: Job = launch(Dispatchers.Default) {
        Thread.sleep(1000L) // 이미지 변환 작업 실행 시간
        println("[${Thread.currentThread().name}] 이미지2 변환 완료")
      }
    
      joinAll(convertImageJob1, convertImageJob2) // 이미지1과 이미지2가 변환될 때까지 대기
    
      val uploadImageJob: Job = launch(Dispatchers.IO) {
        println("[${Thread.currentThread().name}] 이미지1,2 업로드")
      }
    }
    /*
    // 결과:
    [DefaultDispatcher-worker-1 @coroutine#2] 이미지1 변환 완료
    [DefaultDispatcher-worker-2 @coroutine#3] 이미지2 변환 완료
    [DefaultDispatcher-worker-1 @coroutine#4] 이미지1,2 업로드
    */
    ```
    
    ![2025-02-05_18-28-54.jpg](attachment:fa130f5f-6c95-44f3-9660-de548aa32c31:2025-02-05_18-28-54.jpg)
    

## 3. CoroutineStart.LAZY를 사용한 코루틴 지역 시작

- 나중에 실행돼야 할 코루틴을 미리 생성해야 하는 경우, 지역 시작 니응을 활용할 수 있음

- launch 함수의 start 인자로 CoroutineStart.LAZY를 넘겨 지연 시작 옵션 설정
    
    → Lazy Coroutine으로 생성되며 별도 실행 요청이 있기 전까지 실행되지 않음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      val lazyJob: Job = launch(start = CoroutineStart.LAZY) {
        println("[${Thread.currentThread().name}][${getElapsedTime(startTime)}] 지연 실행")
      }
      delay(1000L) // 1초간 대기
      lazyJob.start() // 코루틴 실행
    }
    /*
    // 결과:
    [main @coroutine#2][지난 시간: 1014ms] 지연 실행
    */
    ```
    

## 4. cancel을 사용해 코루틴 취소

- 코루틴이 실행될 필요가 없어졌음에도 취소하지 않고 계속해서 실행되도록 두면 → 스레드 점유 & 성능 저하

- cancel 함수로 Job 객체의 코루틴 취소하기
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val startTime = System.currentTimeMillis()
      val longJob: Job = launch(Dispatchers.Default) {
        repeat(10) { repeatTime ->
          delay(1000L) // 1000밀리초 대기
          println("[${getElapsedTime(startTime)}] 반복횟수 ${repeatTime}")
        }
      }
      delay(3500L) // 3500밀리초(3.5초)간 대기
      longJob.cancel() // 코루틴 취소
    }
    /*
    // 결과:
    [지난 시간: 1016ms] 반복횟수 0
    [지난 시간: 2021ms] 반복횟수 1
    [지난 시간: 3027ms] 반복횟수 2
    */
    ```
    

- cancelAndJoin을 사용한 순차 처리
    - 취소 요청된 시점이 아닌, 취소가 완료된 후 실행 (순차성 보장)
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val longJob: Job = launch(Dispatchers.Default) {
        // 작업 실행
      }
      longJob.cancelAndJoin() // longJob이 취소될 때까지 runBlocking 코루틴 일시 중단
      executeAfterJobCancelled()
    }
    ```
    

## 5. 코루틴의 취소 확인

- 코루틴이 취소를 확인하는 시점 = 일시 중단 시점, 실행 대기 시점
    
    
- cancel을 호출해도 취소되지 않을 수 있음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val whileJob: Job = launch(Dispatchers.Default) {
        while(true) {
          println("작업 중")
        }
      }
      delay(100L) // 100밀리초 대기
      whileJob.cancel() // 코루틴 취소
    }
    ```
    
    - 코루틴 내부에 취소를 확인할 수 있는 시점이 없기 때문에, 계속해서 반복 실행이 진행됨

1. delay를 사용한 취소 확인
    - delay 함수 : suspend fun으로 선언되어 일시 중단하도록 설정함 → 취소 확인 시점으로 활용 가능
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val whileJob: Job = launch(Dispatchers.Default) {
            while(true) {
              println("작업 중")
              delay(1L)
            }
          }
          delay(100L)
          whileJob.cancel()
        }
        ```
        
    - 그러나, 작업을 강제로 일시 중단시켜야 하기 때문에 효율적이지 않음
    
2. yield를 사용한 취소 확인
    - yield : 스레드 사용을 중단하고 양보하도록 설정 → 취소 확인 시점으로 활용 가능
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val whileJob: Job = launch(Dispatchers.Default) {
            while(true) {
              println("작업 중")
              yield()
            }
          }
          delay(100L)
          whileJob.cancel()
        }
        ```
        
        - 이 코드에서 Dispatchers.Default를 사용하는 코루틴은 whileJob밖에 없음
            
            → yield를 호출하면 일시 중단 후 곧바로 재개 & 일시 중단된 잠깐의 시점에서 취소 체크 발생
            
    - 그러나, 코루틴이 매번 일시 중단되는 것은 작업을 비효율적으로 만듬

1. CoroutineScope.isActive를 사용한 취소 확인
    - CoroutineScope는 코루틴의 활성화 여부를 Boolean으로 제공하는 isActive 함수 제공
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val whileJob: Job = launch(Dispatchers.Default) {
            while(this.isActive) {
              println("작업 중")
            }
          }
          delay(100L)
          whileJob.cancel()
        }
        ```
        

## 6. 코루틴의 상태와 Job의 상태 변수

- 코루틴 : 6가지 상태를 가짐
    
    ![2025-02-05_19-24-00.jpg](attachment:05cc0002-37d8-41e5-acc0-23a1709d0e41:2025-02-05_19-24-00.jpg)
    
    1. 생성 (new)
        - 코루틴 빌더를 통해 코루틴을 생성하면 기본적으로 생성 상태에 놓이고 자동으로 실행 중으로 이동
        - CoroutineStart.LAZY 설정 시 생성 상태를 유지함
    2. 실행 중 (active)
        - 코루틴이 실행 중 & 일시 중단된 상태
    3. 실행 완료 (Completed)
        - 코루틴의 모든 코드가 실행 완료된 상태
    4. 취소 중 (Cancelling)
        - 코루틴에 취소가 요청된 상태 (아직 취소 상태가 아니어서 코루틴은 계속해서 실행됨)
    5. 취소 완료 (Cancelled)
        - 코루틴의 취소 확인 시점에서 취소가 확인된 상태, 더 이상 코루틴이 실행되지 않음

- Job 객체 : 코루틴이 어떤 상태에 있는지 나타내는 상태 변수들을 외부로 공개
    1. isActive : 코루틴 활성화 여부 (실행 중)
    2. isCancelled : 코루틴 취소 요청 여부 (취소 중 & 취소 완료)
    3. isCompleted : 코루틴 실행 완료 여부 (실행 완료 & 취소 완료)
        
        ![2025-02-05_19-34-39.jpg](attachment:f23cd9f9-35a0-4d16-a3bf-c3091f16891b:2025-02-05_19-34-39.jpg)
