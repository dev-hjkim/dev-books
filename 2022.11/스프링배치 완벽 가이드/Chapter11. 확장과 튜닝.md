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
#### TaskExecutorPartitionHandler
#### MessageChannelPartitionHandler
#### DeployerPartitionHandler
### 원격 청킹