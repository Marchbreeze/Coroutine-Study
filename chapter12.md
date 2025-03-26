## 1. 단위 테스트 기초

> 코틀린이 아닌, 코루틴 테스트에 필요한 최소한만을 다룸
> 
> 
> 단위 테스트 시 일반적으로 사용되는 Mokito나 MockK와 같은 라이브러리들도 생략하고, 테스트를 진행하는 데 필수적인 JUnit5만을 다룸
> 

### (1) 단위 테스트

- 단위 (Unit)
    - 명확히 정의된 역할의 범위를 갖는 코드의 집합 (소프트웨어의 기능을 담는 코드 블록)
    - ex. 정의된 동작을 실행하는 개별 함수, 클래스, 모듈

- 단위 테스트 (Unit Test)
    - 단위에 대한 자동화된 테스트를 작성하고 실행하는 프로세스
    - 소프트웨어의 특정 기능이 제대로 동작하는지 확인
    - OOP에서, 테스트 대상이 되는 단위는 주로 객체
        
        ![2025-03-26_05-02-31.jpg](attachment:f25e4661-39c4-400d-9b7b-66520e57e49f:2025-03-26_05-02-31.jpg)
        

### (2) 테스트 환경 설정

- build.gradle.kts 파일에 테스트 라이브러리 의존성 추가
    
    ```kotlin
    dependencies {
        // 코루틴 라이브러리
        implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.2")
        // JUnit5 테스트 프레임워크
        testImplementation("org.junit.jupiter:junit-jupiter-api:5.10.0")
        testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine:5.10.0")
    }
    
    // JUnit5을 사용하기 위한 옵션 추가
    tasks.test {
        useJUnitPlatform()
    }
    ```
    
    - testImplementation : 테스트 코드를 컴파일하고 실행하는 데 사용하는 의존성
    - testRuntimeOnly : 컴파일 시에는 필요없지만, 테스트가 실행될 때 필요한 의존성
        
        → src/test 경로에서만 사용할 수 있음
        
        → 앱의 프로덕션 빌드를 만들 때는 포함되지 않음
        

### (3) 간단한 테스트 만들고 실행하기

- 예시 UseCase 설정
    
    ```kotlin
    class AddUseCase {
      fun add(vararg args: Int): Int {
        return args.sum()
      }
    }
    ```
    

- 외부에 add 함수를 노출하고 있기 때문에, add 함수에 대한 테스트가 필요함
- src/test 경로에 테스트 생성
    
    ```kotlin
    class AddUseCaseTest {
      @Test
      fun `1 더하기 2는 3이다`() {
        val addUseCase: AddUseCase = AddUseCase()
        val result = addUseCase.add(1, 2)
        assertEquals(3, result)
      }
    }
    ```
    
    - 개별 테스트는 `@Test` 어노테이션이 붙은 함수로 작성됨
    - 클래스 내부에는 ````로 테스트가 작성됨
    - `assertEquals`를 사용해 결과값이 3인지 단언
        
        → 함수 왼쪽에 생성되는 재생 버튼을 클릭하여 테스트 실행
        
    
- 단언(Assert)
    - 테스트를 검증하는데 사용하는 개념
    - 특정한 조건이 참임을 검증함으로써 코드가 올바르게 동작하는 지 확인
    - assertEquals 말고도 assertTrue 등 여러 단언 함수가 JUnit5에서 지원됨

### (4) @BeforeEach 어노테이션을 사용한 테스트 환경 설정

- 테스트를 여러개 포함하는 경우
    
    ```kotlin
    class AddUseCaseTestMultipleCase {
      @Test
      fun `1 더하기 2는 3이다`() {
        val addUseCase: AddUseCase = AddUseCase()
        val result = addUseCase.add(1, 2)
        assertEquals(3, result)
      }
    
      @Test
      fun `-1 더하기 2는 1이다`() {
        val addUseCase: AddUseCase = AddUseCase()
        val result = addUseCase.add(-1, 2)
        assertEquals(1, result)
      }
    }
    ```
    
    - AddUseCase 클래스를 인스턴스화하는 코드가 똑같이 반복됨
        
        → @BeforeEach 어노테이션 사용
        
        → 모든 테스트 실행 전에 공통으로 실행 : AddUseCase 클래스 인스턴스화를 재사용할 수 있음
        

- @BeforeEach 적용 :
    
    ```kotlin
    class AddUseCaseTestBeforeEach {
      lateinit var addUseCase: AddUseCase
    
      @BeforeEach
      fun setUp() {
        addUseCase = AddUseCase()
      }
    
      @Test
      fun `1 더하기 2는 3이다`() {
        val result = addUseCase.add(1, 2)
        println(result)
        assertEquals(3, result)
      }
    
      @Test
      fun `-1 더하기 2는 1이다`() {
        val result = addUseCase.add(-1, 2)
        println(result)
        assertEquals(1, result)
      }
    }
    ```
    

### (5) 테스트 더블을 사용해 의존성 있는 객체 테스트

- 다른 객체와 의존성이 있는 객체의 예시
    
    ```kotlin
    class UserProfileFetcher(
        private val userNameRepository: UserNameRepository,
        private val userPhoneNumberRepository: UserPhoneNumberRepository
    ) {
        fun getUserProfileById(id: String): UserProfile {
            // 유저의 이름을 UserNameRepository로부터 가져옴
            val userName = userNameRepository.getNameByUserId(id)
            // 유저의 전화번호를 UserPhoneNumberRepository로부터 가져옴
            val userPhoneNumber = userPhoneNumberRepository.getPhoneNumberByUserId(id)
            return UserProfile(
                id = id,
                name = userName,
                phoneNumber = userPhoneNumber
            )
        }
    }
    ```
    
    - UserNameRepository에서 유저 이름을, UserPhoneNumberRepository 유저 번호를 가져와 합쳐 UserProfile 타입의 객체를 반환하는 함수

- 파라미터로 받아야할 Repository 인터페이스에 대한 구현체가 없어 사용이 어려움
- 실제 구현체를 사용하게 되면, 테스트가 해당 구현체에 영향을 받기 때문에 정확한 결과를 파악하기 어려움

### (5-1) 테스트 더블을 통한 객체 모방

- 테스트 더블(Test Double)
    - 객체에 대한 대체물 → 객체의 행동을 모방하는 객체를 만드는데 사용
    - 다른 객체와의 의존성을 가진 객체를 테스트하기 위해서 필요
    - 종류 : Stub, Fake, Mock, Dummy, Spy, …
        
        ![2025-03-26_18-29-25.jpg](attachment:7d684caa-fd84-4b7c-ab29-cf1011a2fa93:2025-03-26_18-29-25.jpg)
        

1. 스텁(Stub)
    - 스텁 객체 : 미리 정의된 데이터를 반환하는 모방 객체
    - 반환값이 없는 동작을 구현하지 않고, 반환값이 있는 동작만 미리 정의된 데이터를 반환하도록 구현
        
        ```kotlin
        class StubUserNameRepository(
            private val userNameMap: Map<String, String> // 데이터 주입
        ) : UserNameRepository {
            override fun saveUserName(id: String, name: String) {
                // 구현하지 않는다.
            }
        
            override fun getNameByUserId(id: String): String {
                return userNameMap[id] ?: ""
            }
        }
        ```
        
2. 페이크(Fake)
    - 페이크 객체 : 실제 객체와 비슷하게 동작하도록 구현된 모방 객체
        
        ```kotlin
        class FakeUserPhoneNumberRepository : UserPhoneNumberRepository {
          private val userPhoneNumberMap = mutableMapOf<String, String>()
        
          override fun saveUserPhoneNumber(id: String, phoneNumber: String) {
            userPhoneNumberMap[id] = phoneNumber
          }
        
          override fun getPhoneNumberByUserId(id: String): String {
            return userPhoneNumberMap[id] ?: ""
          }
        }
        ```
        
        - 유저의 전화번호를 로컬 DB 대신 인메모리에 저장해 실제 객체처럼 동작할 수 있도록 설정

### (5-2) 테스트 더블을 사용한 테스트

- Given-When-Then
    - 테스트 코드 작성법 중 하나로, 테스트의 시나리오를 설명함으로써 가독성 향상
    1. Given : 테스트 환경 설정 작업
    2. When : 동작이나 이벤트를 발생시키고 결과 획득
    3. Then : 테스트 결과 검증

1. 스텁 테스트 더블 테스트
    
    ```kotlin
    @Test
    fun `UserNameRepository가 반환하는 이름이 홍길동이면 UserProfileFetcher에서 UserProfile를 가져왔을 때 이름이 홍길동이어야 한다`() {
      // Given
      val userProfileFetcher = UserProfileFetcher(
        userNameRepository = StubUserNameRepository(
          userNameMap = mapOf<String, String>(
            "0x1111" to "홍길동",
            "0x2222" to "조세영"
          )
        ),
        userPhoneNumberRepository = FakeUserPhoneNumberRepository()
      )
    
      // When
      val userProfile = userProfileFetcher.getUserProfileById("0x1111")
    
      // Then
      assertEquals("홍길동", userProfile.name)
    }
    ```
    

1. 페이크 테스트 더블 테스트
    
    ```kotlin
    @Test
    fun `UserPhoneNumberRepository에 휴대폰 번호가 저장되어 있으면, UserProfile를 가져왔을 때 해당 휴대폰 번호가 반환되어야 한다`() {
      // Given
      val userProfileFetcher = UserProfileFetcher(
        userNameRepository = StubUserNameRepository(
          userNameMap = mapOf<String, String>(
            "0x1111" to "홍길동",
            "0x2222" to "조세영"
          )
        ),
        userPhoneNumberRepository = FakeUserPhoneNumberRepository().apply {
          this.saveUserPhoneNumber("0x1111", "010-xxxx-xxxx")
        }
      )
    
      // When
      val userProfile = userProfileFetcher.getUserProfileById("0x1111")
    
      // Then
      assertEquals("010-xxxx-xxxx", userProfile.phoneNumber)
    }
    ```
    

- 테스트를 위해 매번 인터페이스를 구현해 테스트 더블을 만드는 것은 매우 비효율적
    
    → Mokito나 MockK와 같은 라이브러리를 활용해야 함
    

## 2. 코루틴 단위 테스트 시작하기

### (1) 첫 코루틴 테스트 작성하기

- 테스트 대상 객체 설정
    
    ```kotlin
    class RepeatAddUseCase {
      suspend fun add(repeatTime: Int): Int = 
    	  withContext(Dispatchers.Default) {
    	    var result = 0
    	    repeat(repeatTime) { repeat += 1 }
    	    return@withContext result
    	  }
    }
    ```
    
- 일시 중단 함수에 대한 테스트 진행
    - 일시 중단 함수는 코루틴 내부에서 실행되어야 하므로, 테스트 함수를 코루틴 내부에서 실행될 수 있도록 처리해줘야 함
    
    ```kotlin
    class RepeatAddUseCaseTest {
      @Test
      fun `100번 더하면 100이 반환된다`() = runBlocking {
        // Given
        val repeatAddUseCase = RepeatAddUseCase()
    
        // When
        val result = repeatAddUseCase.add(100)
    
        // Then
        assertEquals(100, result)
      }
    }
    ```
    

### (2) runBlocking을 활용한 테스트의 한계

- runBlocking 함수를 사용한 테스트에서 실행에 오래 걸리는 일시 중단 함수를 실행하는 경우 문제 발생
    
    ```kotlin
    class RepeatAddWithDelayUseCase {
        suspend fun add(repeatTime: Int): Int {
            var result = 0
            repeat(repeatTime) {
                delay(100L)
                result += 1
            }
            return result
        }
    }
    ```
    
    → add 함수는 더하기에 사용하는 코루틴이 하나이므로 모든 더하기 연산이 순차적으로 실행됨
    
    → 테스트 소요 시간이 길어짐
    

## 3. 코루틴 테스트 라이브러리

- 테스트 작성 원칙 중 하나 : 테스트에 걸리는 시간을 짧게 해서 부담없이 실행하도록 하는 것
    
    → 코루틴 테스트 라이브러리 : 가상 시간에서 테스트를 진행할 수 있도록 코루틴 스케줄러 제공
    

### (1) 코루틴 테스트 라이브러리 의존성 설정하기

```kotlin
dependencies {
    // 코루틴 테스트 라이브러리
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.2")
}

```

### (2) TestCoroutineScheduler 사용해 가상 시간에서 테스트 진행하기

- 오랜 시간이 걸리는 작업이 포함돼 있는 경우
- 가상 시간을 사용해 코루틴의 동작이 자신이 원하는 시간까지 단번에 진행될 수 있도록 만들어서 해결

1. advanceTimeBy
    - 함수의 인자로 입력된 값만큼 가상 시간이 밀리초 단위로 흐르게 됨
    - currentTime 프로퍼티로 흐른 가상 시간을 밀리초 단위로 확인할 수 있음
        
        ```kotlin
        @Test
        fun `가상 시간 조절 테스트`() {
          // 테스트 환경 설정
          val testCoroutineScheduler = TestCoroutineScheduler()
        
          testCoroutineScheduler.advanceTimeBy(5000L) // 가상 시간에서 5초를 흐르게 만듦 : 현재 시간 5초
          assertEquals(5000L, testCoroutineScheduler.currentTime) // 현재 시간이 5초임을 단언
          testCoroutineScheduler.advanceTimeBy(6000L) // 가상 시간에서 6초를 흐르게 만듦 : 현재 시간 11초
          assertEquals(11000L, testCoroutineScheduler.currentTime) // 현재 시간이 11초임을 단언
          testCoroutineScheduler.advanceTimeBy(10000L) // 가상 시간에서 10초를 흐르게 만듦 : 현재 시간 21초
          assertEquals(21000L, testCoroutineScheduler.currentTime) // 현재 시간이 21초임을 단언
        }
        ```
        
        ![2025-03-27_01-04-02.jpg](attachment:216a7eb6-ac17-4aec-8a06-71fca4198f92:2025-03-27_01-04-02.jpg)
        
2. StandardTestDispatcher
    - 테스트용 CoroutineDispatcher 객체인 TestDispatcher를 생성
    - 이후 생성된 TestDispatcher를 사용하는 CoroutineScope 객체를 생성해 활용
    - TestCoroutineScheduler 객체와 TestDispatcher 객체를 함께 사용해서 테스트 진행
        
        ```kotlin
        @Test
        fun `가상 시간 위에서 테스트 진행`() {
          // 테스트 환경 설정
          val testCoroutineScheduler: TestCoroutineScheduler = TestCoroutineScheduler()
          val testDispatcher: TestDispatcher = StandardTestDispatcher(scheduler = testCoroutineScheduler)
          val testCoroutineScope = CoroutineScope(context = testDispatcher)
        
          // Given
          var result = 0
        
          // When
          testCoroutineScope.launch {
            delay(10000L) // 10초간 대기
            result = 1
            delay(10000L) // 10초간 대기
            result = 2
            println(Thread.currentThread().name)
          }
        
          // Then
          assertEquals(0, result)
          testCoroutineScheduler.advanceTimeBy(5000L) // 가상 시간에서 5초를 흐르게 만듦 : 현재 시간 5초
          assertEquals(0, result)
          testCoroutineScheduler.advanceTimeBy(6000L) // 가상 시간에서 6초를 흐르게 만듦 : 현재 시간 11초
          assertEquals(1, result)
          testCoroutineScheduler.advanceTimeBy(10000L) // 가상 시간에서 10초를 흐르게 만듦 : 현재 시간 21초
          assertEquals(2, result)
        }
        ```
        
        ![2025-03-27_01-52-34.jpg](attachment:5faa8e7d-e3e2-4c6a-bfa7-f6a664bb30cb:2025-03-27_01-52-34.jpg)
        
        - 실제로 테스트 시 직접 시간을 컨트롤하는 경우는 적음

1. advanceUntilIdle
    - 객체를 사용하는 모든 디스페처와 연결된 작업이 모두 완료될 때까지 가상 시간을 흐르게 만듬
        
        ```kotlin
        @Test
        fun `advanceUntilIdle의 동작 살펴보기`() {
          // 테스트 환경 설정
          val testCoroutineScheduler: TestCoroutineScheduler = TestCoroutineScheduler()
          val testDispatcher: TestDispatcher = StandardTestDispatcher(scheduler = testCoroutineScheduler)
          val testCoroutineScope = CoroutineScope(context = testDispatcher)
        
          // Given
          var result = 0
        
          // When
          testCoroutineScope.launch {
            delay(10_000L) // 10초간 대기
            result = 1
            delay(10_000L) // 10초간 대기
            result = 2
          }
          testCoroutineScheduler.advanceUntilIdle() // testCoroutineScope 내부의 코루틴이 모두 실행되게 만듦
        
          // Then
          assertEquals(2, result)
        }
        ```
        
        - launch 코루틴이 모두 실행될 때까지 가상 시간을 흐르게 하고 단언 진행
    

### (3) TestCoroutineScheduler를 포함하는 StandardTestDispatcher

- StandardTestDispatcher 함수에는 기본적으로 TestCoroutineScheduler 객체를 생성하는 부분 포함 → 직접 생성해서 인자로 넘겨줄 필요 X
    
    ```kotlin
    public fun StandardTestDispatcher(
    	scheduler: TestCoroutineScheduler? = null,
    	name: String? = null,
    ): TestDispatcher = StandardTestDispatcherImpl(
    	scheduler ?: TestMainDispatcher.currentTestScheduler :?
    TestCoroutineScheduler(), name)
    ```
    

- StandardTestDispatcher를 사용해서 testDispatcher를 바로 생성할 수 있음
    
    ```kotlin
    @Test
    fun `StandardTestDispatcher 사용하기`() {
      // 테스트 환경 설정
      val testDispatcher: TestDispatcher = StandardTestDispatcher()
      val testCoroutineScope = CoroutineScope(context = testDispatcher)
    
      //...
    }
    ```
    

### (4) TestScope 사용해 가상 시간에서 테스트 진행하기

- TestScope 함수 호출 시 내부에 TestDispatcher 객체를 가진 TestScope 객체가 반환됨
    
    ```kotlin
    @Test
    fun `TestScope 사용하기`() {
      // 테스트 환경 설정
      val testCoroutineScope: TestScope = TestScope()
      
      //...
    }
    ```
    
    - TestScope > StandardTestDispatcher > TestCoroutineScheduler

### (5) runTest 사용해 테스트 만들기

1. runTest 사용해 TestScope 대체하기
    - runTest { }
        
        TestScope 객체를 사용해 코루틴을 실행시키고, 내부의 일시 중단 함수가 실행되더라도 작업이 곧바로 실행 완료될 수 있도록 가상 시간을 흐르게 만드는 기능을 가진 코루틴 빌더
        
    - runTest > TestScope > StandardTestDispatcher > TestCoroutineScheduler
        
        ```kotlin
        @Test
        fun `runTest 사용하기`() {
          // Given
          var result = 0
        
          // When
          runTest { // this: TestScope
            delay(10000L) // 10초간 대기
            result = 1
            delay(10000L) // 10초간 대기
            result = 2
          }
        
          // Then
          assertEquals(2, result)
        }
        
        ```
        

1. runTest로 테스트 전체 감싸기
    - 일반적으로는 runTest의 람다식이 테스트 전체를 감싸는 방식으로 활용됨
        
        ```kotlin
        @Test
        fun `runTest로 테스트 감싸기`() = runTest {  // this: TestScope
          // Given
          var result = 0
        
          // When
          delay(10000L) // 10초간 대기
          result = 1
          delay(10000L) // 10초간 대기
          result = 2
        
          // Then
          assertEquals(2, result)
        }
        ```
        
        - 테스트 환경 설정, 결과 비교 부분에서도 일시 중단 함수가 호출될 수 있기 때문

1. runTest 함수의 람다식에서 TestScope 사용하기
    - this를 통해 TestScope 객체에 접근할 수 있음 → 확장 프로퍼티 모두 사용 가능
        
        ```kotlin
        @Test
        fun `runTest에서 가상 시간 확인`() = runTest {  // this: TestScope
          delay(10000L) // 10초간 대기
          println("가상 시간: ${this.currentTime}ms") // 가상 시간: 10000ms
          delay(10000L) // 10초간 대기
          println("가상 시간: ${this.currentTime}ms") // 가상 시간: 20000ms
        }
        /*
        // 결과:
        가상 시간: 10000ms
        가상 시간: 20000ms
        */
        ```
        

## 4. 코루틴 단위 테스트 만들어보기

### (1) 코드 준비하기

- SNS에서 자신의 팔로워를 검색하는 클래스에 대한 테스트 작성해보기
    
    ```kotlin
    // 모델 클래스
    sealed class Follower(
      open val id: String,
      open val name: String
    ) {
      data class OfficialAccount(
        override val id: String,
        override val name: String
      ) : Follower(id, name)
    
      data class PersonAccount(
        override val id: String,
        override val name: String
      ) : Follower(id, name)
    }
    ```
    
    ```kotlin
    // 각 유저의 팔로워 데이터를 가져와 합쳐서 반환
    class FollowerSearcher(
      private val officialAccountRepository: OfficialAccountRepository,
      private val personAccountRepository: PersonAccountRepository
    ) {
      suspend fun searchByName(name: String): List<Follower> = coroutineScope {
        val officialAccountsDeferred = async {
          officialAccountRepository.searchByName(name)
        }
        val personAccountsDeferred = async {
          personAccountRepository.searchByName(name)
        }
    
        return@coroutineScope listOf(
          *officialAccountsDeferred.await(),
          *personAccountsDeferred.await()
        )
      }
    }
    ```
    
    ```kotlin
    // 각 저장소
    interface OfficialAccountRepository {
      suspend fun searchByName(name: String): Array<Follower.OfficialAccount>
    }
    
    interface PersonAccountRepository {
      suspend fun searchByName(name: String): Array<Follower.PersonAccount>
    }
    ```
    

- 테스트 더블 작성
    
    ```kotlin
    class StubOfficialAccountRepository(
      private val users: List<Follower.OfficialAccount>
    ) : OfficialAccountRepository {
      override suspend fun searchByName(name: String): Array<Follower.OfficialAccount> {
        delay(1000L)
        return users.filter { user ->
          user.name.contains(name)
        }.toTypedArray()
      }
    }
    ```
    
    ```kotlin
    class StubPersonAccountRepository(
      private val users: List<Follower.PersonAccount>
    ) : PersonAccountRepository {
      override suspend fun searchByName(name: String): Array<Follower.PersonAccount> {
        delay(1000L)
        return users.filter { user ->
          user.name.contains(name)
        }.toTypedArray()
      }
    }
    ```
    

### (2) 테스트 작성하기

1. @BeforeEach로 테스트 실행 환경 설정
    
    ```kotlin
    class FollowerSearcherTest {
    
      private lateinit var followerSearcher: FollowerSearcher
    
      @BeforeEach
      fun setUp() {
        followerSearcher = FollowerSearcher(
          officialAccountRepository = stubOfficialAccountRepository,
          personAccountRepository = stubPersonAccountRepository
        )
      }
    
    	//...
    	
    	companion object {
        private val companyA = Follower.OfficialAccount(id = "0x0000", name = "CompanyA")
        private val companyB = Follower.OfficialAccount(id = "0x0001", name = "CompanyB")
        private val companyC = Follower.OfficialAccount(id = "0x0002", name = "CompanyC")
    
        private val stubOfficialAccountRepository = StubOfficialAccountRepository(
          users = listOf(companyA, companyB, companyC)
        )
    
        private val personA = Follower.PersonAccount(id = "0x1000", name = "PersonA")
        private val personB = Follower.PersonAccount(id = "0x1001", name = "PersonB")
        private val personC = Follower.PersonAccount(id = "0x1002", name = "PersonC")
    
        private val stubPersonAccountRepository = StubPersonAccountRepository(
          users = listOf(personA, personB, personC)
        )
      }
    }
    ```
    
    - stub 테스트 더블의 인스턴스를 생성하고, @BeforeEach 블록에서 객체가 초기화될 때 주입

1. 테스트 작성
    
    ```kotlin
    @Test
    fun `공식 계정과 개인 계정이 합쳐져 반환되는지 테스트`() = runTest {
      // Given
      val searchName = "A"
      val expectedResults = listOf(companyA, personA)
    
      // When
      val results = followerSearcher.searchByName(searchName)
    
      // Then
      Assertions.assertEquals(
        expectedResults,
        results
      )
    }
    ```
    
    ![2025-03-27_02-20-39.jpg](attachment:2b958d54-343e-49bb-8431-cbe411cc5181:2025-03-27_02-20-39.jpg)
    

## 5. 코루틴 테스트 심화

### (1) 함수 내부에서 새로운 코루틴을 실행하는 객체에 대한 테스트

- 일시 중단 함수가 아닌 함수 내부에서 새로운 코루틴을 실행하는 경우
- ex. 문자열의 상태를 저장하는 객체
    
    ```kotlin
    class StringStateHolder(
      private val dispatcher: CoroutineDispatcher = Dispatchers.IO
    ) {
      private val coroutineScope = CoroutineScope(dispatcher)
    
      var stringState = ""
        private set
    
      fun updateStringWithDelay(string: String) {
        coroutineScope.launch {
          delay(1000L)
          stringState = string
        }
      }
    }
    ```
    
    → CoroutineDispatcher 객체가 TestCoroutineScheduler 객체를 사용할 수 있도록 주입 받음
    
- 테스트
    
    ```kotlin
    class StringStateHolderTestSuccess {
      @Test
      fun `updateStringWithDelay(ABC)가 호출되면 문자열이 ABC로 변경된다`() {
        // Given
        val testDispatcher = StandardTestDispatcher()
        val stringStateHolder = StringStateHolder(
          dispatcher = testDispatcher
        )
    
        // When
        stringStateHolder.updateStringWithDelay("ABC")
    
        // Then
        testDispatcher.scheduler.advanceUntilIdle()
        Assertions.assertEquals("ABC", stringStateHolder.stringState)
      }
    }
    ```
    
    - 테스트에서 일시 중단 함수를 호출하는 부분이 없기 때문에 runTest {}로 감싸지 않음
    - 대신 StandardTestDispatcher로 CoroutineDispatcher 객체를 만들어서, StringStateHolder를 초기화할 때 주입
    - StringStateHolder 객체 내부에서 만들어진 코루틴이 모두 실행 완료됨의 보장을 위해 Then에서 advanceUntilIdle()를 호출

### (2) backgroundScope를 사용해 테스트 만들기

- runTest 함수를 사용해 테스트 진행 시 메인 스레드를 사용 → 내부의 모든 코루틴 실행 전까지 종료 X
    
    ```kotlin
    @Test
    fun `메인 스레드만 사용하는 runTest`() = runTest {
      println(Thread.currentThread())
    }
    /*
    // 결과:
    Thread[Test worker @kotlinx.coroutines.test runner#5,5,main]
    */
    ```
    
- 무한히 실행되는 작업이 실행된다면 테스트가 계속해서 실행됨
    
    ```kotlin
    @Test
    fun `끝나지 않아 실패하는 테스트`() = runTest {
      var result = 0
    
      launch {
        while (true) {
          delay(1000L)
          result += 1
        }
      }
    
      advanceTimeBy(1500L)
      Assertions.assertEquals(1, result)
      advanceTimeBy(1000L)
      Assertions.assertEquals(2, result)
    }
    /*
    // 결과:
    After waiting for 10s, the test coroutine is not completing, there were active child jobs: ["coroutine#3":StandaloneCoroutine{Active}@381f03c1]
    kotlinx.coroutines.test.UncompletedCoroutinesError: After waiting for 10s, the test coroutine is not completing, there were active child jobs: ["coroutine#3":StandaloneCoroutine{Active}@381f03c1]
    at app//kotlinx.coroutines.test.TestBuildersKt__TestBuildersKt$runTest$2$1$2$1.invoke(TestBuilders.kt:349)
    at app//kotlinx.coroutines.test.TestBuildersKt__TestBuildersKt$runTest$2$1$2$1.invoke(TestBuilders.kt:333)
    ...
    */
    ```
    
    - launch 코루틴이 계속해서 실행돼 테스트가 종료되지 않음
    - runTest 코루틴은 테스트가 무한히 실행되는 것을 방지하고자 일정 시간 (10초) 뒤에서 종료되지 않으면 예외를 던짐

- backgroundScope
    - 무한히 실행되는 작업을 테스트하기 위해 사용
    - runTest 코루틴의 모든 코드가 실행되면 자동으로 취소됨
        
        ```kotlin
        @Test
        fun `backgroundScope를 사용하는 테스트`() = runTest {
          var result = 0
        
          backgroundScope.launch {
            while (true) {
              delay(1000L)
              result += 1
            }
          }
        
          advanceTimeBy(1500L)
          Assertions.assertEquals(1, result)
          advanceTimeBy(1000L)
          Assertions.assertEquals(2, result)
        }
        ```
        
        → runTest 코루틴의 마지막 코드가 실행되면 backgroundScope가 취소됨
