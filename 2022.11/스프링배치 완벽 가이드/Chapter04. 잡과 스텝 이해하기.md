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
- 현재 어떤 스텝이 실행 중인지, 해당 스텝이 몇 개의 레코드까지 처리했는지, 현재 배치는 어떤 상태인지 등을 스프링 배치가 트래킹 해줌
- 상태를 저장해주는 곳 중 하나인 JobExecution, 잡의 상태를 JobExecution의 ExecutionContext에 저장
- 웹 애플리케이션에 HttpSession이 있는 것처럼 배치 잡의 세션 역할을 함
- JobExecution처럼 StepExecution도 존재, 적절한 ExecutionContext로 데이터 사용 범위 조절 가능
- ExecutionContext의 모든 내용이 JobRepository에 저장되므로 안전함

### ExecutionContext 조작하기
- ExecutionContext를 사용하려면 JobExecution 또는 StepExecution에서 가져와야 함
- 잡의 ExecutionContext 받아오기
  - ```chunkContext.getStepContext().getStepExecution().getJobExecution().getExecutionContext()```
  - stepContext()에서 바로 .getJobExecutionContext()로 받아올 수도 있음. -> Map<String, Object>로 리턴됨
  - 하지만 위에서 리턴된 Map을 수정해도 실제 ExecutonContext에 반영이 되질 않음! 주의!
- StepExecution의 ExecutionContext에 있는 키를 JobExecution의 ExecutionContext로 승격시키는 것 가능
  - ExecutionContextPromotionListener가 담당
  - 스텝이 성공적으로 완료되면 잡의 ExecutionContext에 키를 복사
- ItemStream 인터페이스를 통해서도 ExecutionContext 접근 가능, 이 내용은 뒷장에서 더 자세히 다룸

#### ExecutionContext 저장하기
- 잡이 처리되는 동안 스프링 배치는 각 청크를 커밋하며 잡이나 스텝의 현재 ExecutionContext를 데이터베이스에 저장
- BATCH_JOB_EXECUTION_CONTEXT 테이블의 SERIALIZED_CONTEXT : 잡이 실행 중이거나 실패한 경우에만 채워짐

## 스텝 알아보기
- 모든 단위 작업의 조각, 각각 독립적
- 자체적 입력, 자체적 처리기, 자체적 출력을 처리

### 테스크릿 처리와 청크 처리 비교
- Tasklet 모델
  - execute 메소드가 RepeatStatus.FINISHED를 반환할 때까지 반복적으로 실행
- Chunk 기반 처리 모델
  - ItemReader, (ItemProcessor), ItemWriter로 구성
  - 청크 또는 레코드 그룹 단위로 처리
  - <img src="/book_image/chunk_based_processing.jpg">

### 스텝 구성
#### 테스크릿 스텝
- 사용자가 작성한 일반 POJO를 스텝으로 활용하는 방법
  - MethodInvokingTaskletAdapter를 통해 태스크릿 스텝으로 정의
- Tasklet 인터페이스를 구현하는 방법
  - execute 메서드가 RepeatStatus.FINISHED 반환할 때까지 실행
  - RepeatStatus.CONTINUABLE : 해당 태스크릿을 다시 실행하라는 의미(조건 충족될 때까지 실행하게 할 때 사용)

### 그 밖의 여러 다른 유형의 태스크릿 이해하기
#### CallableTaskletAdapter
- Callable 인터페이스의 구현체를 구성할 수 있게 해주는 어댑터
  - Runnable 인터페이스와 비슷하나 값을 반환(RepeatStatus 반환)할 수 있고 체크 예외를 바깥으로 던질 수 있음
  - call 메서드 사용
- 해당 스텝이 실행되는 스레드가 아닌 다른 스레드에서 실행하고 싶을 때 사용, but! 스텝과 병렬 실행되진 않음

#### MethodInvokingTaskletAdapter
- 실행하고 싶은 로직을 어떤 서비스가 이미 갖고 있는 경우, 이를 잡 내의 태스크릿처럼 실행하기 위해 사용
  - 메서드 호출을 래핑할 뿐인 Tasklet 인터페이스 구현체 만드는 대신 MethodInvokingTaskletAdapter 사용
- 메서드가 ExistStatus 지정하지 않으면 ExitStatus.COMPLETED
- 파라미터를 넘기고 싶을 때에는 잡 파라미터 전달할 때와 같이 늦은 바인딩 방법을 사용

#### SystemCommandTasklet
- 시스템 명령 비동기로 실행 시 사용
- command : 실행할 시스템 명령(ex. "rm -rf /tmp.txt", "touch tmp.txt")
- interruptOnCancel : 잡이 비정상적으로 종료될 때 시스템 프로세스 관련 스레드를 강제로 종료할지 여부를 스프링 배치에 알려줌
- workingDirectory : 명령 실행할 디렉터리
- systemProcessExitCodeMapper : 시스템 반환 코드를 스프링 배치 상태 값으로 매핑해주는 구현체
  - ConfigurableSystemProcessExitCodeMapper
  - SimpleSystemProcessExitCodeMapper
- terminateCheckInterval : 비동기 방식으로 실행되므로 몇 밀리초 단위로 완료 여부 체크할 지 지정
- taskExecutor : 시스템 명령을 실행하는 자신만의 고유한 TaskExecutor 구성 가능(동기식으로 구성하지 않을 것, lock 걸림)
- environmentParams : 명령 실행 전 설정하는 환경 파라미터 목록

#### 청크 기반 스텝
- 커밋 간격에 의해 정의되는 chunk
  - 커밋 간격 50 지정 -> 50개 아이템을 읽고, 처리하고, 쓴 후 다음 chunk 읽고, 처리하고, 쓰고... 반복
  - 49번째 아이템 처리 실패 -> 쓰기 작업 없이 롤백

#### 청크 크기 구성하기
- 정적인 커밋 개수 설정
- CompletionPolicy 구현체 사용
  - SimpleCompletionPolicy : 아이템 개수가 미리 구성해둔 임곗값에 도달하면 청크 완료로 표시
  - TimeoutTerminationPolicy : 처리 시간이 해당 시간을 넘을 때 청크 완료로 표시
  - CompositeCompletionPolicy : 청크 완료 여부를 결정하는 정책이 여러 개일 때, 정책 중 하나라도 청크 완료로 판단되면 청크 완료로 표시
  - customCompletionPolicy : CompletionPolicy 인터페이스의 isComplete, start, update 메서드 오버라이드 하여 청크 완료 로직 구현

#### 스텝 리스너
- StepExecutionListener
  - beforeStep(void), afterStep(ExitStatus 리턴) 메서드
  - 데이터가 제대로 기록되었는지 확인하는 것처럼 무결성 검사 후 ExistStatus를 바꿀 수도 있음 -> 유용!
  - annotation @BeforeStep, @AfterStep
- ChunkListener
  - beforeChunk(void), afterChunk(void) 메서드
  - annotation @BeforeChunk, @AfterChunk

### 스텝 플로우
#### 조건 로직
- 빌더를 통해 잡의 진행방향 지정
  - on(조건).to(다음 진행할 스텝)
  - 조건에 들어가는 것은 ExitStatus, 와일드 카드
    - "*" : 0개 이상의 문자 일치 (ex. C\* : COMPLETE, CORRECT)
    - "?" : 1개의 문자 일치 (ex. ?AT : CAT, KAT)
- JobExecutionDecider : 현재 스텝에서 어떤 레코드 처리를 건너뛰었을 때 다른 스텝을 실행하지 않게 하는 로직 구현 가능
  - decide 메서드 구현
    - JobExecution, StepExecution 모두 이용 가능
    - FlowExecutionStatus(BatchStatus / ExitStatus 쌍 래핑한 래퍼 객체) 리턴

#### 잡 종료하기
- Completed : 처리가 성공적으로 종료되었음을 의미, 동일한 파라미터로 다시 실행 불가
  - Completed로 종료하는 법 : 빌더가 제공하는 end 메서드 사용
  - Step이 반환하는 ExitStatus 관계없이 end 메서드 사용되면 COMPLETED로 종료
- Failed : 성공적으로 완료되지 않음, 동일한 파라미터로 다시 실행 가능
  - Failed로 종료하는 법 : 빌더가 제공하는 fail 메서드 사용
- Stopped : 다시 시작 가능, 오류가 발생하지 않아도 중단된 위치에서 다시 시작 가능
  - 스텝 사이에 사람의 개입이 필요하거나 다른 검사/처리가 필요한 상황에 유용
  - Stopped로 종료하는 법 : 빌더가 제공하는 stopAndRestart 메서드 사용
  - 잡을 다시 시작할 때 처음부터 실행하는 것이 아니라 stopAndRestart 파라미터로 넘겨준 step부터 시작됨
- 스텝의 ExitStatus를 평가하여 BatchStatus를 판별 -> JobRepository에 저장

#### 플로우 외부화하기
- 스텝의 시퀀스를 독자적 플로우로 만드는 방법
  - 플로우 만드는 빌더로 플로우를 정의하고 잡 빌더에서 이를 참조하는 방식
  - JobRepository에 저장된 결과는 플로우를 사용하는 것이나 잡에서 스텝을 구성하는 것이나 차이가 없음
- 플로우 스텝을 사용하는 방법
  - 플로우를 잡 빌더가 아니라 스텝으로 래핑하고 이 래핑된 스텝을 잡 빌더로 전달하는 방식
  - 각각의 개별 스텝을 집계하지 않고도 플로우의 영향을 전체적으로 볼 수 있음
    - 모니터링, 리포팅에 유용
- 잡 내에서 다른 잡을 호출하는 방법(잡 스텝)
  - 스텝 빌더에 호출하려는 다른 잡을 지정, 앞에서 정의한 스텝을 다른 잡의 스텝으로 등록하는 방식
  - 서브 잡에 직접 파라미터 전달하지 않으므로 상위 잡의 JobParameters 또는 ExecutionContext에서 파라미터 추출해 전달
    - jobParametersExtractor bean
  - 하나의 마스터 잡이 연결된 여러 잡을 실행할 경우 실행 처리 제어에 제약이 있을 수 있음 -> 웬만하면 쓰지 말라!
    - 외부 요인에 의해(다른 부서에서 제시간에 파일을 주지 않는 경우 등) 잡 실행 건너뛰어야 할 때
    - 주기적으로 실행하던 배치 중지할 때