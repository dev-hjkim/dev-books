# Chapter 06. 잡 실행하기

as-is : 서블릿 컨테이너나 애플리케이션 서버에서 배치를 실행 시 jar 파일의 classpath를 지정하거나 maven의 shade 플러그인 사용  
to-be : 스프링 부트가 생성하는 실행 가능한 jar 파일로 기동하여 실행  

어떻게 잡을 실행하는지, 스케쥴러를 사용하는 방법, 실행 중인 잡을 중지하고 재시작하는 내용 등에 대해 6장에서 다룰 예정

## 스프링 부트로 배치 잡 시작시키기
- CommandLineRunner와 ApplicationRunner : ApplicationContext가 리프레시 되고 코드를 실행할 준비가 되면 호출하는 메서드 가짐
- JobLauncherCommandLineRunner(CommandLineRunner의 일종)
  - JobLauncher를 사용해 잡을 실행
  - ApplicationContext 내에서 찾아낸 모든 잡을 실행시킴
- 스프링 부트로 배치 잡 시작시키기 위한 두 가지 구상 방법
  - REST 호출이나 특정 이벤트 등으로 잡을 실행할 계획이라면 spring.batch.job.enabled 프로퍼티를 false로 수정할 것
  - 여러 잡 중에서 특정 잡만 실행시키고 싶을 경우 spring.batch.job.names 프로퍼티를 수정할 것(쉼표 가능, 순서대로 실행)

## REST 방식으로 잡 실행하기
- JobLauncher 인터페이스 사용, 잡과 잡 파라미터 전달
  - 잡 파라미터는 JobLauncher의 구현체인 SimpleJobLauncher는 잡 파라미터의 조작을 지원하지 않으므로 전달 전에 전처리 필요
  - @EnableBatchProcessing 어노테이션을 사용할 시 자동으로 사용되는 JobLauncher가 SimpleJobLauncher
  - 적절한 TaskExecutor 구현체를 통해 동기/비동기 구동방식 선택
    - 대부분의 배치 잡은 처리량이 많으므로 비동기 방식으로 실행하는 것이 더 적합할 때가 있음
    - SimpleJobLauncher는 동기식으로 실행, ExitStatus 반환
    - 비동기식으로 실행할 경우 JobExecution의 ID 값만 만환
- JobLauncher는 JobParametersIncrementer도 관리
  - JobParametersBuilder의 getNextJobParameters 메서드를 통해 JobParametersIncrementer를 증가시킴
  - getNextJobParameters 메서드에 Job을 파라미터로 넘겨주는데, 이는 해당 잡이 Incrementer를 가지고 있는지 판별하기 위함
  - Incrementer가 있다면 JobParameters에 적용
- JobLauncher는 잡이 재시작된 것인지 판별하여 JobParameters를 적절히 처리하는 것도 담당

### 쿼츠를 사용해 스케줄링하기
- 엔터프라이즈 환경에서 사용할 수 있는 스케줄러, 쿼츠(Quartz)
  - 오래 전부터 스프링과 연동을 지원하고 있음
- 스케줄러, 잡, 트리거라는 세 가지 주요 컴포넌트로 구성
  - 스케줄러 : SchedulerFactory를 통해 가져오며 JobDetails 및 트리거의 저장소 기능, 트리거가 작동할 때 잡을 실행하는 역할
  - 잡 : 실행할 작업의 단위
  - 트리거 : 작업 실행 시점 정의
  - 트리거 작동 -> 쿼츠에게 잡 실행하라고 지시 -> 잡 개별 실행 정의하는 JobDetails 객체 생성
- 스프링 배치와 쿼츠 함께 동작하는 방식
  1. 스프링 배치 잡 생성
  2. QuartzJobBean으로 스프링 배치 잡을 기동하는 쿼츠 잡 작성
  3. 쿼츠 JobDetail을 생성하도록 스프링이 제공하는 JobDetailBean 구성
  4. 트리거 구성

## 잡 중지하기
### 자연스러운 완료
- 모든 Step이 COMPLETED 상태를 반환하고 잡 자신도 COMPLETED 종료 코드를 반환했을 경우
- 정상적으로, 자연스럽게 완료된 경우 동일한 Jobparameter로 새로운 JobInstance 생성 불가
- 쿼츠 스케줄러를 사용했던 예제와 같이 매일 실행하는 잡이 있다면 타임스탬프를 증분기로 두는 것이 바람직

### 프로그래밍적으로 중지하기
#### 중지 트랜지션 사용하기
- 주입받은 ItemReader에 read 작업 위임
- process 메서드에서 읽어들인 데이터들의 줄 수 count하며 데이터를 model에 매핑
- 스텝이 완료되면 afterStep 메서드 호출, 읽어들인 레코드 수와 기대되는 레코드 수를 비교하여 ExitStatus 반환
  - ExitStatus.STOPPED : 중지
- 스프링 배치는 ItemReader, ItemProcessor, ItemWriter가 ItemStream의 구현체인지 자동으로 확인/등록
  - custom ItemReader의 경우 스프링에 명시적으로 등록되어 있지 않아 ItemStream을 구현했는지 프레임워크가 확인하지 않음
    - 해결방법 1 : 잡에서 ItemReader 명시적으로 등록
    - 해결방법 2 : custom ItemReader에서 ItemStream을 구현하고 적절한 라이프 사이클에 따라 메서드를 호출하도록 하는 방법
- @StepScope를 사용하면 Step마다 ItemReader, ItemProcessor, ItemWriter 인스턴스를 새로 생성하므로 재사용이 가능
- 아래와 같이 트랜지션 API로 잡 플로우 구성하여 중지/재시작 처리
  - step1 STOPPED 반환할 경우 해당 스텝부터 재시작, STOPPED 아닐 경우 step2 -> step3 -> 종료
```
@Bean
public Job transactionJob() {
    return this.jobBuilderFactory.get("transactionJob")
            .start(step1())
            .on("STOPPED").stopAndRestart(step1())
            .from(step1()).on("*").to(step2())
            .from(step2()).next(step3())
            .end()
            .build();
}
```
#### StepExecution을 사용해 중지하기
- @BeforeStep에서 StepExecution에 접근할 수 있는 특징을 이용하여 중지시키는 방법
  - 코드와 구성이 좀더 깔끔해질 수 있다는 장점 존재
- 잡이 STOPPED 상태를 반환했던 중지 트랜지션을 사용한 방법과 달리 JobInterruptionException을 반환
- ItemReader, ItemProcessor, ItemWriter에 대해 자세히 알기 전에 해당 컴포넌트들을 사용하는 예제들이 등장하여 제대로 이해 안됨
  - 뒷장을 읽고 다시 돌아와 읽기!!!

### 오류 처리
- 잘못된 데이터의 수신, NullPointerException 등등의 오류 처리 수행 옵션과 구현 방법 소개
#### 잡 실패
- 스프링 배치 기본 동작이 가장 안전(잡이 중지되면 롤백하므로)
  - 스프링은 예외가 발생하면 스텝과 잡이 실패한 것으로 간주
- 잡 실패는 잡의 중지와는 다름
  - 잡 실패 : ExitStatus.FAILED
  - 잡 중지 : ExitStatus.STOPPED
- 잡 실패 역시 잡을 재시작하면 중단됐던 부분부터 시작

## 재시작 제어하기
### 잡의 재시작 방지하기
- 첫 시도에 실패한 잡을 다시 실행시키지 않게 함
  - JobBuilder의 preventRestart() 호출
  - 잡 인스턴스가 이미 존재하며 재시작 불가하다는 메시지를 출력하며 재시작 불가능

### 재시작 횟수를 제한하도록 구성하기
- 무한정 재시도하는 것이 아니라 정해진 횟수 만큼만 재시도하도록 제한하고 싶을 경우
  - 스텝 수준에서 startLimit 호출
  - limit 이상 재시도할 경우 StartLimitExceedException 발생
  - 재시작 할 때 사용할 구성은 allowStartIfComplete() 메서드로 지정

### 완료된 스텝 재실행하기
- 스프링 배치 잡의 경우 동일 파라미터로 한 번만 실행 가능하지만 스텝의 경우 두 번 이상 실행 가능
  - allowStartIfComplete()이 이를 가능케 함
  - 단, 잡의 ExitStatus가 COMPLETE일 경우에는 allowStartIfComplete(true) 이더라도 다시 실행 불가  
  (잡 자체를 다시 실행할 수 없음)
