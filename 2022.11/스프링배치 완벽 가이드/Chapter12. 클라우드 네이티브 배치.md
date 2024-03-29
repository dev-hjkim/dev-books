# Chapter 12. 클라우드 네이티브 배치

## 12요소 애플리케이션
- 클라우드에서 애플리케이션 실행에 관련된 추가적인 문제를 해결해 주는 방법론
- 목표 : 서비스 형태로 애플리케이션을 개발할 수 있는 패턴을 개발하는 것

### 코드베이스
- 버전 관리되는 하나의 코드베이스와 다양한 배포
- 하나의 애플리케이션이 하나의 배치 잡인 모델을 채택하도록 구성

### 의존성
- 명시적으로 선언되고 분리되는 의존성
- maven, gradle과 같은 빌드 시스템으로 필요한 의존성을 포함시켜 다운로드
- 모든 의존성은 애플리케이션 내에 캡슐화되어야 함. 스프링 부트가 이를 처리함

### 구성
- 환경에 구성 정보 저장
- 구성은 코드와 분리되어야 함. 환경에 독립적이어야 하기 때문

### 백엔드 서비스
- 백엔드 서비스를 연결된 리소스로 취급
- 애플리케이션이 네트워크를 통해 이용하는 모든 서비스(RDBMS, SMTP 서버, S3, 서드파티 API 등)
- 모든 서비스들에 직접적으로 의존성을 가지면 안됨
  - MySQL이든 Amazon RDS이든 코드가 변경되면 안됨. 구성 정보만 변경하면 실행 가능해야 함

### 빌드, 릴리스, 실행
- 철저하게 분리된 빌드와 실행 단계
- 빌드 : 애플리케이션 코드를 컴파일하고 테스트 하는 것
- 릴리스 : 아티팩트 생성과 해당 버전에 대한 고유 식별자를 제공하여 수정할 수 없는 장소에 저장하는 것
- 실행 : 릴리스 된 아티팩트를 가져와서 시스템 환경에서 실행하는 것

### 프로세스
- 애플리케이션을 하나 이상의 무상태 프로세스로 실행
- 애플리케이션을 실행할 때 어떠한 기존 데이터도 고려하지 않을 것이라는 의미
- 이에 반하는 대표적인 예는 sticky session

### 포트 바인딩
- 포트 바인딩을 사용해 서비스 공개
- 애플리케이션을 서비스로 노출하기 위해 서버를 런타임에 추가하는 방식이 아니라 런타임을 포함하고 있음. 필요한 경우에는 포트 자체에 서비스가 바인딩 됨
- 스프링 부트는 애플리케이션에 톰캣을 내장하고 있음
- 스프링 배치 잡이 다양한 이유로 포트를 오픈해 외부에 공개하는 시나리오 처리 기능을 제공

### 동시성
- 프로세스 모델을 사용한 확장
- 프로세스를 수평적으로 확장할 수 있어야 함
- 원격 청킹, 원격 파티셔닝

### 폐기 가능
- 빠른 시작과 정상적인 종료를 통한 안정성 극대화
- 프로세스는 가능한 한 빨리 시작하고 중지 요청을 받으면 정상적으로 종료되어야 함

### 개발/운영 환경 일치
- 개발, 스테이징, 운영 환경을 최대한 비슷하게 유지
- 클라우드의 목표는 비즈니스 민첩성을 제공, 지속적인 배포를 가능하게 하는 것
- 환경 간 차이로 인해 발생되는 문제를 최소화

### 로그
- 로그를 이벤트 스트림으로 취급
- 모든 로그 출력을 기록하여 개발자가 터미널 작업 시 사용하거나 스플렁크와 같은 로그 수집 시스템을 통해 로그를 사용할 수 있어야 함

### 관리자 프로세스
- 관리자 및 관리 작업을 일회성 프로세스로 실행
- 데이터 마이그레이션과 같은 관리 작업, 일회성 작업으로 실행되어야 함

## 간단한 배치 잡
- 스프링 배치 애플리케이션을 스프링의 기능을 사용하여 클라우드 네이티브 애플리케이션으로 발전시키는 예제
- JobExecutionListener 구현체 : beforeJob 메서드 구현을 통해 잡 시작 전 실행
  - S3 버킷에 있는 모든 리소스 목록을 가져와 다운로드하여 파일로 저장
- MultiResourceItemReader, EnrichmentItemProcessor, JdbcBatchItemWriter로 스텝 구성
  - MultiResourceItemReader : 디렉터리 내에 존재하는 모든 파일을 읽어들임
  - EnrichmentItemProcessor : REST API로 단순한 GET 요청 실행하여 아이템 객체에 정보 저장
  - JdbcBatchItemWriter : 아이템 객체 jdbc 정의해둔 DB에 저장
- ``spring.cloud.aws.region.auto`` 옵션이 true일 경우, 애플리케이션이 US 리전에서 실행되면 S3 리전도 동일하게 설정됨
  - 책의 예제에서는 애플리케이션을 AWS에서 실행하지 않으므로 S3만 US 리전에 띄우고자 false 값으로 설정

## 서킷 브레이커
- 발생한 예외 건수가 임곗값을 초과하면 서킷 브레이커가 해당 메서드에 대한 호출을 중지하고 대체 메서드로 트래픽을 라우팅
  - 예시) REST API 과부하 상황에서 REST API가 반환하는 값 대신 기본값을 반환, 추후 특정 알고리즘으로 다시 원래 메서드로 트래픽을 되돌림
- 넷플릭스 : 하이스트릭스 -> resilience4j
- 스프링 배치 : 스프링 리트라이라는 라이브러리에 의존
- 스프링 배치 내 스텝에 내결함성(fault tolerent) 기능 제공
- 내결함성 기능을 사용할 수 있음에도 서킷 브레이커를 사용하는 이유
  - 성능
    - 특정 처리 작업이 재시도되면 트랜잭션 롤백, 커밋 카운트 1 세팅 후 처리작업 재시도 하게 되는데 이 과정이 성능을 저하시킴
    - 오류 발생했음을 표시만 해두고 나주에 재실행하는 것이 훨씬 효율적 -> 서킷 브레이커
  - 사용 사례
    - 스프링 배치는 재시도까지는 가능하나, 문제가 되는 코드에 대한 부하를 줄일 수는 없음
- 스프링 리트라이가 사용하는 서킷 브레이커의 핵심 구성 요소
  - @CircuitBreaker
    - 서킷 브레이커에서 무언가를 래핑해야 함을 나타냄
    - 예외 발생이 몇 초 이내에 이루어졌을 때 circuitBreaker의 기본값 메서드를 실행시킬지, 몇 초 이내에 다시 시도할지 애너테이션의 파라미터로 지정 가능
  - @Recover
    - 재시도 가능한 메서드가 실패했을 때 호출되는 메서드임을 나타냄
    - @CircuitBreaker 애너테이션이 적용된 메서드와 동일한 메서드 시그니처를 가져야 함
- 애플리케이션의 복원력 추가에 도움이 되는 서킷 브레이커

## 구성 외부화
- application.properties 또는 application.yml을 사용할 경우 jar 파일의 애플리케이션과 함께 번들로 제공되므로 외부 환경이 변화할 때 쉽게 변화가 불가
- 스프링 클라우드 컨피그 서버를 사용하거나 서비스 바인딩으로 구성을 적용하는 방법을 통해 구성을 외부화할 필요가 있음

### 스프링 클라우드 컨피그
- 깃 저장소 또는 데이터베이스 백엔드에 저장된 구성을 제공하기 위한 구성 서버
- 로컬에서 적용할 시 ``spring.application.name``과 ``spring.cloud.config.failFast`` 설정 필요
  - spring.application.name
    - 클라이언트가 서버에 올바른 구성을 요청하기 위해 사용
  - spring.cloud.config.failFast
    - 컨피그 서버에서 구성을 검색할 수 없는 경우 클라이언트에게 예외를 발생시켜 애플리케이션이 시작되지 않게 함
    - 기본적으로는 구성을 검색할 수 없는 경우 로컬 환경의 구성을 사용
- 컨피그 서버를 실행하는 가장 쉬운 방법은 스프링 클라우드 CLI를 사용하는 것
  - 클라우드 서버 구성 요소를 시작하는 기능과 값 암호화와 같은 유틸리티 제공
- 배치 잡 실행 -> spring cloud configserver 명령어로 컨피그 서버 시작 -> 클라이언트가 컨피그 서버에서 구성을 읽어감 -> 배치 잡 시작
- 잡 구성을 가져오는 메커니즘에 차이는 생겼으나 동작은 동일

### 유레카를 사용한 서비스 바인딩
- 서비스 검색 도구인 유레카의 구현체를 포함한 스프링 클라우드 넷플릭스를 사용
- 서비스를 검색 가능하게 등록하고 해당 서비스와 어떤 통신을 해야 하는지, 서비스 검색을 위해 유레카가 어디 있는지만 제공하면 구성 완료
- bootstrap.yml
  - application.yml은 ApplicationContext가 로드될 때 읽어들여짐
  - 더 이른 시점에 설정을 읽어들여야 할 경우 bootstrap.yml을 통해 ApplicationContext의 상위 컨텍스트 역할을 하는 부트스트랩 ApplicationContext를 생성해 동작시킴
- @EnableDiscoveryClient의 autoRegister 옵션은 유레카 서버에 서비스로 등록할지 말지를 결정하는 옵션
- @LoadBalanced 애너테이션은 RestTemplate을 자동 구성하고 클라이언트 측의 부하 분산 등 유레카를 통해 제공되는 구성을 사용할 수 있게 해줌
- 호스트와 포트를 직접 지정해줄 필요 없이 서비스 이름만 지정하여 통신 가능, 통신 관련한 부분들은 스프링 클라우드가 알아서 처리
- spring cloud eureka를 통해 유레카 실행
- localhost:8761(default) 에서 웹 대시보드 제공
- 새로운 동적 환경에 탄력적으로 대응 가능

## 배치 처리 오케스트레이션
- 스프링 클라우드 데이터 플로우 : 데이터 처리 애플리케이션을 오케스트레이션하는 도구

### 유레카를 사용한 서비스 바인딩
- 스프링 클라우드 데이터 플로우 : 클라우드 파운드리, 쿠버네티스 및 로컬 환경과 같은 사용 중인 플랫폼에서 배치 잡을 시작하는 서버 어플리케이션
  - 적절한 플랫폼에서 배치 잡을 배포하고 시작할 수 있음
  - 대화형 셸 또는 웹 기반 UI를 통해 서버와 연동, REST API를 통해 서버와 통신
  - wget으로 다운로드
- 유용한 모니터링 정보를 얻기 위해서는 배치 잡의 JobRepository와 스프링 클라우드 컨피그 서버의 JobRepository 구성이 동일해야 함
- 애플리케이션을 오케스트레이션하기 위해 데이터 플로우에 실행 파일의 이름과 좌표를 제공해 애플리케이션 등록
  - jar 파일 또는 도커 이미지일 수 있음
  - app register 명령어 사용
- 애플리케이션 등록 후에는 태스트 정의(task definition)를 만들어야 함
  - 태스크 정의 : 태스크를 시작하기 위한 템플릿, 태스크 이름과 태스크를 실행하기 위해 설정해야 하는 프로퍼티의 조합
- 마지막으로 잡 시작 방법 결정
  - 셸, GUI, REST API 등을 통해 요청 시 태스크 시작 가능
  - 태스크를 지원하는 플랫폼에서 스프링 클라우드 데이터 플로우를 실행할 경우, 태스크 스케줄링도 가능
  - 명령행 아규먼트가 있는 경우에는 --arguments 파라미터와 -properties 파라미터를 통해 추가 가능
- 대시보드를 통해 잡 모니터링 가능
  - localhost:9393/dashboard
    - Apps 탭 : 시스템에 등록된 모든 애플리케이션 목록 확인, 등록
    - Runtime 탭 : 클라우드 데이터 플로우를 통해 배포된 모든 실행 중인 애플리케이션의 상태 확인 가능
    - Streams 탭 : 스프링 클라우드 스트림을 기반으로 메시지 기반 마이크로 서비스를 스트림으로 정의하고 실행 가능
    - Tasks 탭 : 태스크 정의, 실행 + task repository 확인 가능
    - Jobs 탭 : 스프링 배치의 jobRepository를 검색할 수 있는 Task 탭의 확장
    - Analytics 탭 : 데이터 플로우에 내장된 분석 기능을 사용해 기본적 시각화 수행
    - Audit Record 탭 : 보안 및 규정 준수 사용 사례에 제공하는 감사 흐름 확인 가능