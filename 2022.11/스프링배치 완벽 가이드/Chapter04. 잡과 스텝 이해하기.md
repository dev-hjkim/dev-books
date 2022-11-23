# Chapter 04. 잡과 스텝 이해하기

Chapter 02의 예제보다 더 깊게 잡과 스텝에 대해 이해하기

## 잡 소개하기
잡 : 처음부터 끝까지 독립적으로 실행할 수 있는 고유하며 순서가 지정된 여러 스텝의 목록
- 유일하다 : 잡은 스프링의 빈, 유일함. 매번 생성할 필요 없음
- 순서를 가진 여러 스텝의 목록이다 : 장바구니가 비어 있다면 결제 처리 불가, 거래내용 반영 전까지 잔액 계산 불가. 순서가 중요!
- 처음부터 끝까지 실행 가능하다 : 외부 의존성 없이 처음 실행하면 끝까지 일련의 스텝들이 수행됨
- 독립적이다 : 자신이 처리하기로 정의된 모든 요소를 제어, 외부 의존성이 있다면 의존성을 관리(ex. 파일 읽는 잡에서 파일이 없으면 오류 처리)

### 잡의 생명주기 따라가보기

스프링 배치에서 제공하는 두 가지 잡 러너
- CommandLineJobRunner : 스프링을 부트스트랩하고 전달받은 파라미터로 잡 실행
- JobRegistryBackgroundJobRunner : 쿼츠나 JMX 후크와 같은 스케줄러를 사용할 경우  
스프링이 부트스트랩 될 때 실행 가능한 잡을 가지고 있는 JobRegistry를 생성, 이때 사용되는 JobRunner
- JobLauncherCommandLineRunner : 별도의 구성이 없을 시 기본적으로 Application Context 내의 모든 Job을  
기동 시에 실행

배치 실행 시 잡 러너가 사용되긴 하지만 실제 진입점은 JobLauncher의 인터페이스 구현체임에 유의  
CommandLineJobRunner, JobLauncherCommandLineRunner -> JobLauncher 인터페이스 구현체 사용  

JobLauncher의 도움으로 Job 실행 -> JobInstance 생성(BATCH_JOB_INSTANCE, BATCH_JOB_EXECUTION_PARAMS 메타 테이블로 식별)  
-> JobExecution, Job 실행 시마다 생성됨(BATCH_JOB_EXECUTION, 상태정보는 BATCH_JOB_EXECUTION_CONTEXT)  

## 잡 구성하기
잡을 구성하는 다양한 방법 소개

### 잡의 기본 구성
(Chapter 02와 겹치는 내용)

### 잡 파라미터
- 동일한 파라미터로 잡을 두 번 실행하면 JobInstanceAlreadyCompleteException 발생
- 잡 러너가 JobParameters 객체를 생성해 JobInstance에 전달
- JobParameters는 Map<String, JobParameter> 객체의 wrapper, 스프링 배치는 파라미터의 타입에 따라 JobParameter의 접근자 제공
- 타입 지정 방법
  - ```java -jar demo.jar executionDate(date)=2022/11/22```
  - 괄호로 명시, 타입 명은 반드시 소문자
  - string, long, double, date 타입 기본적으로 지원
- 잡에 전달한 파라미터 확인은 JobRepository의 BATCH_JOB_EXECUTION_PARAMS를 통해 가능
- 특정 잡 파라미터가 식별에 사용되지 않게 하려면 접두사 "-"를 사용한다.
  - ```java -jar demo.jar executionDate(date)=2022/11/22 -name=hjkim```
  - name을 바꿔도 기존의 JobInstance로 새로운 JobExecution을 생성

#### 잡 파라미터에 접근하기
- ChunkContext
  - 실행 시점의 잡 상태, 처리중인 chunk 관련 정보 존재
  - JobParameters를 포함하고 있는 StepContext의 참조 존재
- ChunkContext를 통해 잡 파라미터에 접근
  - ```chunkContext.getStepContext().getJobParameters()```
  - getJobParameters()는 Map<String, Object>를 리턴하므로 Object의 타입캐스팅이 필요
- 늦은 바인딩으로 잡 파라미터 얻기
  - ```public Tasklet helloWorldTasklet(@Value("#{jobParameters['name']}") String name)```
  - @JobScope나 @StepScope 어노테이션을 통해 잡이나 스텝의 실행 범위에 들어갈 때까지 빈 생성을 지연시킴

#### 잡 파라미터 유효성 검증하기
- JobParametersValidator 인터페이스의 validate 메서드 구현을 통해 유효성 검증
- DefaultJobParametersValidator는 필수 파라미터가 누락없이 전달되었는지 확인하는 기능 제공
- 위의 두 개의 유효성 검증기를 모두 사용하려면 CompositeJobParametersValidator 사용해야 함

#### 잡 파라미터 증가시키기
- 동일한 파라미터로 잡을 두 번 수행하면 에러 발생 -> 피하기 위한 간단한 방법, JobParameterIncrementer
  - 기본적으로 run.id라는 이름의 long 타입 파라미터의 값을 증가시킴
- 타임스탬프를 파라미터로 사용하려 할 때 -> JobParametersIncrementer 인터페이스의 getNext 메서드로 구현

### 잡 리스너 적용하기
- 잡 실행과 관련있는 실행주기에서 JobExecutionListener 인터페이스의 beforeJob & afterJob 사용하여 로직 추가 가능  
- 알림, 초기화, 정리(clean up) 등에 사용하기 좋음  
- afterJob은 Job 실행의 성공여부와 관계없이 항상 실행됨에 유의
- JobExecutionListener의 구현 없이 @BeforeJob, @AfterJob 어노테이션 제공됨, 이때 래핑 필요

### ExecutionContext

### ExecutionContext 조작하기
#### ExecutionContext 저장하기

## 스텝 알아보기

### 테스크릿 처리와 청크 처리 비교
### 스텝 구성
#### 테스크릿 스텝

### 그 밖의 여러 다른 유형의 태스크릿 이해하기
#### CallableTaskletAdapter
#### MethodInvokingTaskletAdapter
#### SystemCommandTasklet
#### 청크 기반 스텝
#### 청크 크기 구성하기
#### 스텝 리스너

### 스텝 플로우
#### 조건 로직
#### 잡 종료하기
#### 플로우 외부화하기
