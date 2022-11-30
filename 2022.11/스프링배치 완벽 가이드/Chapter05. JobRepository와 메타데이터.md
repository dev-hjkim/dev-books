# Chapter 05. JobRepository와 메타데이터

스프링 배치는 잡 실행 중 오류 복구, 다시 잡을 재개하면 어떻게 되는지 등의 상태 관리 제공하는데  
이때 사용되는 것이 JobRepository  
상태관리 뿐만 아니라 모니터링 영역에도 유용함

## JobRepository란?
- JobRepository 인터페이스
- JobRepository 인터페이스 구현체가 사용하는 데이터 저장소
  - 인메모리 저장소
  - 관계형 데이터베이스 저장소

### 관계형 데이터베이스 사용하기
스프링 배치에서 기본적으로 사용되는 JobRepository, 배치 메타데이터를 저장하는 6개의 테이블  
<img src="/book_image/jobReposiotory_schema.jpg">

- 시작점, BATCH_JOB_INSTANCE
  - 잡 파라미터로 잡 식별, 잡의 논리적 실행
- BATCH_JOB_EXECUTION
  - 잡의 실제 실행 기록
  - 실행 될 때마다 새로운 레코드에 기록되고 잡이 진행되는 동안 주기적으로 업데이트 됨
- BATCH_JOB_EXECUTION_CONTEXT
  - JobExecution의 ExecutionContext를 저장
- BATCH_JOB_EXECUTION_PARAMS
  - 잡이 매번 실행될 때마다 사용된 잡 파라미터 저장
- BATCH_STEP_EXECUTION
  - 스텝의 시작, 완료, 상태에 대한 메타데이터 저장
  - 읽기 횟수, 처리 횟수, 쓰기 횟수, 건너뛰기 횟수 데이터도 추가로 저장
- BATCH_STEP_EXECUTION_CONTEXT
  - StepExecution의 ExecutionContext 저장
  - 스텝 수준에서 ItemReader, ItemWriter와 같은 컴포넌트의 상태를 저장하는 데 사용

### 인메모리 JobRepository
- 배치 잡을 개발하거나 단위 테스트 수행할 때 사용
- Map 객체를 데이터 저장소로 사용하는 JobRepository 구현체 제공
  - 운영 시에는 H2 또는 HSQLDB와 같은 인메모리 db 사용할 것(멀티 스레딩 및 트랜잭션 기능 지원)



## 배치 인프라스트럭처 구성하기
- @EnableBatchProcessing 애너테이션 사용 시 추가 구성 없이 스프링 배치가 제공하는 JobRepository 사용
- 커스터 마이징이 필요할 때? -> BatchConfigurer 인터페이스 사용!

### BatchConfigurer 인터페이스
- 배치 인프라스트럭처 컴포넌트의 구성을 커스터마이징 하는 데 사용되는 전략 인터페이스
- 배치 관련 빈 등록되는 과정
  1. BatchConfigurer 구현체에서 빈 생성
  2. SimpleBatchConfiguration에서 스프링 ApplicationContext에 생성한 빈 등록
  - 보통 (i) 과정에서 커스터마이징 진행
- JobRepository, PlatformTransactionManager, JobLauncher, JobExplorer 커스터마이징 가능
- DefaultBatchCongigurer에는 기본 옵션이 제공되고 있으므로 이를 상속하여 필요한 메서드만 재정의 하는 방식이 좋음

### JobRepository 커스터마이징하기
### TransactionManager 커스터마이징하기
### JobExplorer 커스터마이징하기
### JobLauncher 커스터마이징하기
### 데이터베이스 구성하기

## 잡 메타데이터 사용하기
### JobExplorer