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

## 잡 중지하기
### 자연스러운 완료
### 프로그래밍적으로 중지하기
#### 중지 트랜지션 사용하기
#### StepExecution을 사용해 중지하기

### 오류 처리
#### 잡 실패

## 재시작 제어하기
### 잡의 재시작 방지하기
### 재시작 횟수를 제한하도록 구성하기
### 완료된 스텝 재실행하기

