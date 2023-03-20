# Chapter 11. 확장과 튜닝

## 배치 처리 프로파일링하기
### VisualVM 알아보기
- JVM이 어떤 상태인지 파악할 수 있는 도구
  - overview, monitor, threads, sampler 탭
  - 기본 제공 기능을 확장할 수 있는 여러 플러그인들도 제공
    - Thread Inspector, Visual GC, MBean 등

### 스프링 배치 애플리케이션 프로파일링하기
- 어떤 부분에서 얼마나 많은 CPU를 사용하는가?
  - CPU가 어떤 작업을 하는가와 연관
  - (ex. 잡이 어려운 계산을 수행하는가? CPU가 비즈니스 로직이 아닌 다른 곳에 노력을 들이고 있는가?)
- 무엇 때문에 얼마나 많은 메모리가 사용되는가?
  - (ex. 가용 메모리를 거의 다 소모했는가? 무엇이 메모리를 가득 차지하고 있는가?)
#### CPU 프로파일링
- Monitor 탭에서 CPU usage를 확인, Sampler 탭에서 어느 메서드가 핫스팟에 해당하는지 확인
  - Sampler 탭에서 패키지 기반으로 필터를 거는 것도 가능
  - Sampler 탭에서 핫스팟 메서드와 CPU 점유 시간 확인
#### 메모리 프로파일링
- CPU 프로파일링과 동일하게 Monitor 탭에서 메모리 누수가 발생하는 것을 확인한 후 Sampler 탭을 통해 어떤 객체가 메모리를 차지하는지 확인
- Snapshot 기능을 통해 어느 부분에서 문제가 발생하는지 이전 스냅샷과 비교헤 추적이 가능

## 잡 확장하기
### 다중 스레드 스텝
- 각 청크가 자체적인 스레드에서 실행되게 하는 방식
  - 오류 발생 시 종료하거나 롤백
  - 스텝이 TaskExecutor를 참조하도록 구성
- 단점
  - 원래의 ItemReader는 상태를 유지하여 오류 발생 시 처리가 중단된 위치를 추적해 재시작 가능
  - 다중 스레드 환경에서는 상태 저장이 어려움. 여러 스레드가 접근 가능한 객체의 경우 기존 정보가 덮어씌워지는 등의 문제 발생
  - 입력 메커니즘이 이미 네트워크, 디스크 버스 등과 같은 자원을 소모하고 있는 경우 성능이 크게 나아지지 않을 수 있음

### 병렬 스텝
- 파일에서 데이터를 읽어오는 작업 진행 시 한 파일을 읽는 일이 끝나지 않았다고 해서 다른 파일을 읽는 일을 미룰 필요까진 없을 때 병렬스텝 구현

### 병렬 스텝 구성하기
- FlowBuilder의 split 메서드 사용
  - TaskExecutor를 통해 스텝의 플로우를 각각의 스레드에서 병렬로 실행
  - 병렬로 수행되도록 구성된 여러 플로우가 모두 완료된 이후에 step 실행

### AsyncItemProcessor와 AsyncItemWriter
- 병목 구간이 ItemProcessor에 존재하는 경우, 새 스레드에서 스텝의 ItemProcessor 부분만 실행하게 하는 방식
- AsyncItemProcessor와 AsyncItemWriter 사용
  - AsyncItemProcessor : ItemProcessor 래핑, AsyncItemProcessor가 새 스레드에서 위임자인 ItemProcessor 실행시킴
    - Future 반환
  - AsyncItemWriter : ItemWriter 래핑, Futre를 처리해 위임 ItemWriter에게 결과 전달
  - 반드시 함께 사용 필요
- 스텝 구성 시 AsyncItemProcessor와 AsyncItemWriter를 사용하도록 구성
  - dhunk 설정 시 <domain 객체, Future<domain객체>> chunk(chunk 사이즈) 로 설정해야 함

### 파티셔닝
- 마스터 스텝이 처리할 일을 여러 워커 스텝으로 넘기는 개념
  - 각 워커는 자체적으로 읽기(ItemReader), 처리(ItemProcessor), 쓰기(ItemWriter) 등을 담당하여 병렬로 처리
- 두 가지 주요 추상화를 이해해야 제대로 사용 가능
  1. Partitioner 인터페이스
     1. 파티셔닝할 데이터를 여러 파티션으로 나누는 역할을 담당
     2. 스프링 배치에서는 기본적으로 단 하나의 Partitioner 구현체인 MultiResourcePartitioner   
     (여러 리소스 배열을 확인하여 리소스당 파티션을 만듬)
     3. partition(int gridSize) 단일 메서드로 구성됨
        - gridSize : 가장 효율적으로 데이터를 분할할 수 있는 워커 개수 지정(자동으로 지정해주지 않음)
        - Map<String, ExecutionContext> 반환
          1. 키는 파티션의 이름으로 고유해야 함
          2. ExecutionContext는 처리할 대상을 식별하는 파티션 메타데이터
  2. PartitionHandler 인터페이스
     1. 워커와 의사소통(작업대상을 어떻게 알려줄지 / 작업이 완료된 시점을 어떻게 식별할지) 하는 데 사용되는 인터페이스
     2. 대부분의 경우 Partitioner 구현체는 구현하더라도 PartitionHandler는 직접 작성하지 않음
     3. 스프링 포트폴리오는 세 가지 PartitionerHandler 구현체를 제공
        1. TaskExecutorPartitionHandler
           - 스프링 배치가 제공
           - 단일 JVM 내에서 파티셔닝 개념 사용, 동일 JVM 내의 여러 스레드에서 워커 실행
        2. MessageChannelPartitionHandler
           - 스프링 배치가 제공
           - 원격에서 처리 가능하도록 원격 JVM에 메타데이터 전송(spring integration 사용)
        3. DeployerPartitionHandler
           - 스프링 클라우드 태스크 프로젝트가 제공
           - 스프링 클라우드 디플로이어 구현체에 위임하여 워커 실행
           - 워커가 실시간으로 동적 확장되어 시작, 파티션 실행, 종료 수행
- 주의할 점
  - 마스터와 모든 워커 스텝이 모두 동일한 JobRepository 데이터베이스와 통신하도록 구성되어야 함
  - MessageChannelPartitionHandler를 사용할 시, 원격 JVM과 통신이 가능해야 함

#### TaskExecutorPartitionHandler
- 단일 JVM 내에서 여러 스레드를 사용해 워커를 실행할 수 있게 해주는 컴포넌트
- 잡 파라미터 대신 스텝의 ExecutionContext에서 파일 위치 혹은 실행에 필요한 데이터를 얻도록 구성
- 잡 구성 시에는 Partitioner 구현체와 PartitionHandler 구현체를 사용하여 파티션 스텝을 정의하고 해당 스텝으로 잡을 구성
- TaskExecutorPartitionHandler는 실행하려는 스텝, TaskExecutor 설정 필요
  - TaskExecutor 설정하지 않을 시 기본적으로 SyncTaskExecutor 사용됨
  - 기본값이 선택될 경우 비동기 처리도 불가하고 스레드 생성 개수를 제한하지도 않아 위험. 따라서 반드시 다중 스레드를 사용할 수 있도록 TaskExecutor 설정 필요
- BATCH_STEP_EXECUTION 테이블에 파티션 스텝과 관련된 레코드 + 각 파티션마다의 레코드가 하나씩 추가되어 있음
  - 예시 : step1 레코드 + step1:partition1 & step1:partition2 & step1:partition0 레코드

#### MessageChannelPartitionHandler
- 외부 JVM과 통신하는 데 스프링 인티그레이션의 MessageChannel 추상화를 사용
- 마스터 스텝은 메시지로 워커와 통신, 각 워커는 큐에서 메시지를 청취하는 리스너를 갖고 있으며 마스터로부터 요청이 들어왔을 때 스텝 실행 후 결과 반환
- 주의할 점
  - 각 JVM이 동일한 JobRepository를 바라보도록 구성 필요(그렇지 않을 경우 잡 재시작 불가)
  - Step이 Job 컨텍스트 외부에서 실행되므로 JobExecution 또는 JobExecutionContext의 정보를 워커 스텝으로 전달 불가
- MasterConfiguration과 WorkerConfiguration 별도 구성 필요
  - MasterConfiguration : Job 구성 / WorkerConfiguration : 스텝이 사용하는 리더, 라이터 구성
- MasterConfiguration
  - RemotePartitioningMasterStepBuilderFactory로 원격 마스터 파티션 스텝을 만듬
  - outbound flow 구성 : 요청 채널에 메시지가 들어올 시 핸들러는 AMQP 아웃바운드 어댑터로써, 메시지를 래빗 MQ 큐로 보냄
  - inbound flow 구성
    1. 각 워커가 회신한 메시지를 마스터가 수신하면 이를 집계하여 결과를 평가, 스텝 성공여부 결정
       - 래빗MQ의 응답큐에서 메시지를 수신하면 해당 메시지를 가져와 응답 채널에 넣음
    2. JobRepository를 폴링하여 StepExecution의 상태를 확인, 모두 완료된 것으로 저장되면 성공한 것으로 평가
- WorkerConfiguration
  - RemotePartitioningWorkerStepBuilderFactory로 원격 파티션 스텝의 워커 측에 필요한 컴포넌트 구성 가능
    - StepExecutionRequestHandler 구성 : 마스터 스텝으로부터 메시지를 수신해 원격 JVM에서 실행하는 역할
  - inboundChannel, outboundChannel, chunk size, reader, writer 으로 WorkerStep 구성
- 래빗 MQ 가 실행 중인지 확인한 후 세 개의 워커 JVM에서는 ``--spring.profiles.active=worker`` 명령어로 애플리케이션 실행
- 하나의 JVM에서는 ``--spring.profiles.active=master`` 명령어로 어플리케이션 실행, master가 실행되어야 worker도 동작하기 시작함

#### DeployerPartitionHandler
### 원격 청킹