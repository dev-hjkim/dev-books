# Chapter 11. 확장과 튜닝

## 배치 처리 프로파일링하기
### VisualVM 알아보기
- JVM이 어떤 상태인지 파악할 수 있는 도구
  - overview, monitor, threads, sampler 탭
  - 기본 제공 기능을 확장할 수 있는 여러 플러그인들도 제공
    - Thread Inspector, Visual GC, MBean 등

### 스프링 배치 애플리케이션 프로파일링하기
#### CPU 프로파일링
#### 메모리 프로파일링
## 잡 확장하기
### 다중 스레드 스텝
### 병렬 스텝
### 병렬 스텝 구성하기
### AsyncItemProcessor와 AsyncItemWriter
### 파티셔닝
#### TaskExecutorPartitionHandler
#### MessageChannelPartitionHandler
#### DeployerPartitionHandler
### 원격 청킹