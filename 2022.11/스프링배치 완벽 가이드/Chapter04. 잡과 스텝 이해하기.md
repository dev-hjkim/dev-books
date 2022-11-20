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

#### 잡 파라미터에 접근하기
#### 잡 파라미터 유효성 검증하기
#### 잡 파라미터 증가시키기

### 잡 리스너 적용하기

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
