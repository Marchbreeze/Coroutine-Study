## 1. CoroutineContext의 구성 요소

- 코루틴 빌더 함수의 선언에서 사용되는 CoroutineContext
    
    ```kotlin
    public fun CoroutineScope.launch(
    	context: CoroutineContext = EmptyCoroutineContext,
    	start: CoroutineStart = CoroutineStart.DEFAULT, 
    	block: suspend CoroutineScope.() -› Unit
    ): Job
    ```
    
    - context 자리에는 CoroutineName 객체, CoroutineDispather 객체가 사용될 수도 있음
        
        → 이들이 CoroutineContext 객체의 구성요소이기 때문
        

- CoroutineContext
    - 코루틴을 실행하는 실행 환경을 설정하고 관리하는 인터페이스
    - CoroutineDispatcher, CoroutineName, Job 등의 객체를 조합해 실행 환경 설정

- 구성 요소
    1. CoroutineName : 코루틴 이름 설정
    2. CoroutineDispatcher : 코루틴을 스레드에 할당해 실행
    3. Job : 코루틴의 추상체로 코루틴을 조작하는데 사용
    4. CoroutineExceptionHandler : 코루틴에서 발생한 예외를 처리

## 2. CoroutineContext 구성

### (1) CoroutineContext 구성 요소 관리 방법

- 키-값 쌍으로 각 구성 요소를 관리함
    
    ![2025-02-19_19-03-41.jpg](attachment:e1447f7d-d83b-4127-96b6-3dc70b545893:2025-02-19_19-03-41.jpg)
    
    - 각 구성 요소는 고유한 키를 가지며, 중복 값을 허용되지 않음
    - CoroutineContext 객체는 각 구성 요소에 해당하는 객체를 한 개씩만 가질 수 있음

### (2) CoroutineContext 구성하기

- 키에 값을 직접 대입하는 방법을 사용하지 않음
    
    → 대신 CoroutineContext 객체 간에 더하기 연산자(+)를 사용해 객체 구성
    
    ```kotlin
    val coroutineContext: CoroutineContext = newSingleThreadContext("MyThread")
      + CoroutineName("MyCoroutine")
    ```
    
    - 설정되지 않은 구성 요소는 설정되지 않은 상태로 유지됨

- 만들어진 CoroutineContext 객체는 launch의 context 인자로 넘겨서 코루틴에 사용할 수 있음
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val coroutineContext: CoroutineContext = newSingleThreadContext("MyThread") + CoroutineName("MyCoroutine")
    
      launch(context = coroutineContext) {
        println("[${Thread.currentThread().name}] 실행")
      }
    }
    /*
    // 결과:
    [MyThread @MyCoroutine#2] 실행
    */
    ```
    

- 구성 요소가 없는 CoroutineContext는 EmptyCoroutineContext로 생성 가능
    
    ```kotlin
    val emptyCoroutineContext: CoroutineContext = EmptyCoroutineContext
    ```
    

### (3) CoroutineContext 구성 요소 덮어쓰기

- CoroutineContext 객체에 같은 구성 요소가 둘 이상 더해지면, 나중에 추가된 값으로 덮어씀
    
    ```kotlin
    fun main() = runBlocking<Unit> {
      val coroutineContext: CoroutineContext = newSingleThreadContext("MyThread") + CoroutineName("MyCoroutine")
      val newCoroutineContext: CoroutineContext = coroutineContext + CoroutineName("NewCoroutine")
    
      launch(context = newCoroutineContext) {
        println("[${Thread.currentThread().name}] 실행")
      }
    }
    /*
    // 결과:
    [MyThread @NewCoroutine#2] 실행
    */
    ```
    
    ![2025-02-19_19-32-24.jpg](attachment:880a567e-2189-4d29-bb7d-3950a0fb29f4:2025-02-19_19-32-24.jpg)
    

### (4) 여러 구성 요소로 이루어진 CoroutineContext 합치기

- 두 CoroutineContext를 합치면 중복된 구성 요소는 나중에 추가된 값으로 사용
    
    ```kotlin
    val coroutineContext1 = CoroutineName("MyCoroutine1") + newSingleThreadContext("MyThread1")
    val coroutineContext2 = CoroutineName("MyCoroutine2") + newSingleThreadContext("MyThread2")
    val combinedCoroutineContext = coroutineContext1 + coroutineContext2
    ```
    
    ![2025-02-19_19-34-40.jpg](attachment:1fdb7c72-f3d0-4a9f-8c43-e1dc1e9142dc:2025-02-19_19-34-40.jpg)
    

### (5) CoroutineContext에 Job 생성해 추가하기

- Job() 호출을 통해 생성해서 추가할 수 있음
    
    ```kotlin
    import kotlinx.coroutines.*
    import kotlin.coroutines.CoroutineContext
    
    val myJob = Job()
    val coroutineContext: CoroutineContext = Dispatchers.IO + myJob
    ```
    

- Job 객체를 직접 생성해 추가하는 경우 : 코루틴의 구조화가 깨지기 때문에 주의 필요

## 3. CoroutineContext 구성 요소 접근

### (1) CoroutineContext 구성 요소의 키

- CoroutineContext.Key 인터페이스 활용 → CoroutineContext 구성 요소의 키 구현

- 일반적으로 CoroutineContext 자신 내부에 키를 싱글톤 객체로 구현
    - ex. CoroutineName
        
        ```kotlin
        public data class CoroutineName( 
        	val name: String
        ) : AbstractCoroutineContextElement(CoroutineName) {
        	public companion object Key : CoroutineContext.Key<CoroutineName>
        	••• 
        }
        ```
        
    - ex. Job
        
        ```kotlin
        public interface Job : CoroutineContext.Element {
        	public companion object Key : CoroutineContext.Key<Job>
        	••• 
        }
        ```
        

### (2) 키를 사용해 CoroutineContext 구성 요소 접근

1. 싱글톤 키를 사용
    - ex. CoroutineName.Key 활용
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val coroutineContext = CoroutineName("MyCoroutine") + Dispatchers.IO
          val nameFromContext = coroutineContext[CoroutineName.Key]
          println(nameFromContext)
        }
        /*
        // 결과:
        CoroutineName(MyCoroutine)
        */
        ```
        

1. 구성 요소 자체를 키로 사용
    - 구성 요소들은 동반 객체로 Key를 가지고 있기 때문에 명시적으로 사용하지 않아도 키로 사용 가능
    - ex. CoroutineName 활용
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val coroutineContext = CoroutineName("MyCoroutine") + Dispatchers.IO
          val nameFromContext = coroutineContext[CoroutineName] // '.Key'제거
          println(nameFromContext)
        }
        /*
        // 결과:
        CoroutineName(MyCoroutine)
        */
        ```
        

1. 구성 요소의 key 프로퍼티 활용
    - 구성 요소들은 모두 key 프로퍼티를 가지며, 구성 요소에 접근할 수 있음
    - ex. 각 구성 요소 인스턴스의 key 프로퍼티 활용
        
        ```kotlin
        fun main() = runBlocking<Unit> {
          val coroutineName : CoroutineName = CoroutineName("MyCoroutine")
          val dispatcher : CoroutineDispatcher = Dispatchers.IO
          val coroutineContext = coroutineName + dispatcher
        
          println(coroutineContext[coroutineName.key]) // CoroutineName("MyCoroutine")
          println(coroutineContext[dispatcher.key]) // Dispatchers.IO
        }
        /*
        // 결과:
        CoroutineName(MyCoroutine)
        Dispatchers.IO
        */
        ```
        

## 4. CoroutineeContext 구성 요소 제거

### (1) minusKey 사용해 구성 요소 제거

- coroutineContext에서 구성 요소를 제거하기 위해서는, minusKey 함수를 호출하고 구성 요소를 인자로
- ex.
    
    ```kotlin
    val deletedCoroutineContext = coroutineContext.minusKey(CoroutineName)
    ```
    

### (2) minusKey 사용 시 주의할 점

- minusKey를 호출한 CoroutineContext 객체는 그대로 유지되고, 구성요소가 제거될 새로운 객체가 반환됨
- ex.
    
    ```kotlin
    @OptIn(ExperimentalStdlibApi::class)
    fun main() = runBlocking<Unit> {
      val coroutineName = CoroutineName("MyCoroutine")
      val dispatcher = Dispatchers.IO
      val job = Job()
      val coroutineContext: CoroutineContext = coroutineName + dispatcher + job
    
      val deletedCoroutineContext = coroutineContext.minusKey(CoroutineName)
    
      println(coroutineContext[CoroutineName])
      println(coroutineContext[CoroutineDispatcher])
      println(coroutineContext[Job])
    }
    /*
    // 결과:
    CoroutineName(MyCoroutine)
    Dispatchers.IO
    JobImpl{Active}@65e2dbf3
    */
    ```
