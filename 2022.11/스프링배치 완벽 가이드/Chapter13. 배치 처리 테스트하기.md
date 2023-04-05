# Chapter 13. 배치 처리 테스트하기

## JUnit과 Mockito를 사용한 단위 테스트
- 테스트 주도 개발 : 소프트웨어의 품질 뿐 아니라 개별 개발자와 팀 전체의 생산성을 향상시키는 방법
- 단위 테스트
  - 단일 테스트 : 단 하나를 테스트함. 애플리케이션의 최소 컴포넌트로, 보통은 하나의 메서드를 테스트
  - 격리 : 개별 컴포넌트의 동작 방식을 테스트하는 것이 목적. 의존성 최소화
  - 반복 가능한 방식 : 동일한 시나리오를 반복해서 수행할 수 있어야 함. 이를 통해 시스템 변경 시 회귀 테스트에 사용 가능
- JUnit, Mockito, 스프링 프레임워크 단위 테스트에 사용 가능
  - JUnit, Mockito : 코드의 단위테스트를 만드는 데 유용
  - 스프링 프레임워크 : 서로 다른 레이어의 통합이나 애플리케이션의 시작부터 끝까지를 테스트 하는 등 광범위한 문제를 테스트하는 데 유용

### JUnit
- 표준화된 방식으로 자바 클래스의 단위 테스트를 할 수 있는 기능을 제공하는 간단한 프레임워크

#### JUnit 생명주기
- JUnit이 실행시킬 수 있도록 JUnit 애너테이션을 적용한 자바 클래스가 테스트 케이스
  - 테스트 케이스에는 테스트를 실행하는 메서드, 테스트 실행 전 전제 조건을 설정하거나 실행 이후 정리할 수 있는 메서드 제공됨
  - 테스트 케이스는 일반적인 자바 객체로, 생성자가 아규먼트를 가져서는 안됨
  - 각 테스트는 하나 이상의 테스트 메서드를 가짐
    - 테스트 메서드는 public이어야 하고 void 이며, 아규먼트를 갖지 않아야 함
    - @Test 애너테이션 적용하여 표시
- 필요한 조건을 설정 -> 테스트 실행 -> assert 메서드로 유효성 검증
- @BeforeEach -> @Test -> @AfterEach
  - @BeforeEach : 각 테스트 메서드 실행 이전 특정 메서드를 실행하고자 할 때 사용해야 할 애너테이션
  - @AfterEach : 각 테스트 메서드 실행 이후 해당 메서드를 실행하고자 할 때 사용해야 할 애너테이션
- @BeforeAll : 클래스 내의 모든 테스트 메서드들이 실행되기 전에 단 한번만 실행해야 할 경우에 사용
- @Ignore : 실행하지 않고 지나갈 테스트 메서드와 클래스를 나타낼 때 사용
- @RunWith : 테스트 케이스를 실행하는 클래스를 JUnit이 제공하는 클래스가 아닌 다른 클래스로 지정하고 싶을 경우 사용

### 목(Mock) 객체
- 애플리케이션 서버, 메시징 미들웨어, 데이터베이스 등과 같은 외부 시스템에 의존하는 경우가 많음 -> 단위 테스트의 범위를 넘어섬
- 비즈니스 로직을 테스트 할 목적이라면 그에 맞게 외부 의존성의 영향 없이 목 객체를 사용하여 테스트 필요
- mock 객체는 stub이 아니다!
  - 스텁은 애플리케이션의 다양한 부분을 대체하기 위해 작성하는 구현체로, 하드코딩 된 로직을 포함
- 목 객체 동작 방식
  1. 프록시 기반 방식
     - Mockito가 사용하는 방식
     - 목 프레임워크로 프록시 객체를 만들고 해당 프록시 객체를 필요로 하는 객체에게 setter 또는 constructor를 통해 세팅
     - 외부 수단을 통해 의존성을 설정할 수 있어야 함. 메서드 내에서 테스트 객체를 생성(new ~~())하면 안됨
     - 스프링 프레임워크와 같이 의존성 주입 프레임워크가 코드 수정 없이 프록시 객체를 주입해줌
  2. 클래스 재매핑 방식
     - JMockit가 채택한 방식
     - 클래스 로더에게 로딩되는 클래스 파일에 대한 참조를 재매핑하도록 지시하는 방식
     - new 연산자로 만든 객체도 매핑 가능
     - 모든 기능을 사용하기 위해 필요한 클래스 로더에 대해 이해하는 것이 어려울 수 있음

### Mockito
- 확인이 필요한 동작을 모킹해 중요한 동작만 검증할 수 있게 해줌
- JUnit과 Mockito는 spring-boot-starter-test 의존성에 포함되어 있음
- @Mock : 테스트가 실행되면 해당 객체의 프록시 객체(mock)를 생성하도록 지시
  - @BeforeEach 메서드에서 MockitoAnnotations.initMocks 메서드 사용 시 목 초기화
  - @BeforeEach 메서드에서 테스트 대상 클래스의 새 인스턴스를 생성하고 @Mock으로 만들어진 목 객체 주입
    - 각 테스트 메서드에서 대상 클래스를 생성하므로 항상 초기화된 상태로 사용할 수 있도록 보장, 다른 테스트 메서드에 영향 주지 않음
- ArgumentCaptor : 추후 분석에 사용할 수 있도록 데이터를 캡쳐 가능하게 해줌
  - 목에 전달되는 값이 예상하는 값과 동일한지 확인 가능
- assertThrows : 예외가 발생하지 않거나 잘못된 유형의 예외가 발생하면 해당 assertion은 실패

## 스프링 클래스를 사용해 통합 테스트하기
### 스프링을 사용해 통합 테스트하기
- 통합 테스트 : 서로 다른 여러 컴포넌트 간의 상호작용이 정상적으로 수행되는지 테스트
  - 데이터베이스와의 상호작용을 테스트
  - 스프링 빈과의 상호작용을 테스트
- HSQLDB : 100% 자바로 구현된 인메모리 데이터베이스로, 언제 어디서나 테스트 실행 가능

#### 테스팅 환경 구성하기
- 인메모리 HSQLDB 인스턴스를 생성하도록 의존성 추가
- @ExtendWith(SpringExtention.class) : 테스트 시에 스프링이 제공하는 모든 장점 이용 가능
  - @RunWith에 해당하는 JUnit5의 애너테이션
- @JdbcTest : 인메모리 데이터베이스를 생성하고 초기 데이터 적재

### 스프링 배치 테스트하기
스프링 배치 특화 컴포넌트(스텝, 잡에 의존적인 컴포넌트들 등등)의 테스트 방법에 대해 설명

#### 잡과 스텝 스코프 빈 테스트하기
- 잡과 스텝 스코프를 사용하는 컴포넌트의 테스트를 스텝 스코프 바깥에서 진행하려 할 때 의존성 문제 해결하기
  - TestExecutionListener 사용
    - 테스트 메서드 실행 전후에 수행되어야 하는 일을 정의하는 스프링 API
    - 스프링 배치에서는 TestExecutionListener 구현체인 JobScopeTestExecutionListener, StepScopeTestExecutionListener 제공
    - StepScopeTestExecutionListener
      - 테스트 케이스에서 팩토리 메서드를 사용하여 StepExecution을 가져오고 반환된 컨텍스트를 현재 메서드의 컨텍스트로 사용 가능
      - 각 테스트 메서드가 실행되는 동안 stepContext를 제공
- @SpringBatchTest
    - ApplicationContext에 자동으로 테스트할 수 있는 많은 유틸리티 제공
    - 대표적 4가지의 bean
      - 잡이나 스텝을 실행하는 JobLauncherTestUtils 인스턴스
      - JobRepository에서 JobExecutions를 생성하는 데 사용하는 JobRepositoryTestUtils
      - 스텝 스코프와 잡 스코프 빈을 테스트할 수 있는 StepScopeTestExecutionListener & JobScopeTestExecutionListener
- StepScopeTestExecutionListener를 사용해 스텝 스코프 의존성을 처리하는 방법
  1. getStepExecution 메서드 작성 필요
    - 책의 예제에서는 step scope로 JobParameter의 값을 주입받고 있는 중이라 메서드 내에서 우선 JobParameters 객체부터 생성
    - MetaDataInstanceFactory로 StepExecution을 생성, 생성자에 jobParameters 전달
      - MetaDataInstanceFactory : JobExecution, StepExecution 인스턴스를 생성하는 유틸리티 클래스
      - 생성한 Execution 인스턴스들이 JobRepository에 저장되지 않음(JobRepositoryTestUtils와 다른점)
  2. 테스트의 stepExecution을 생성하였으므로 ItemReader, ItemWriter 등을 주입받아 테스트 진행

#### 스텝 테스트하기
#### 잡 테스트하기
