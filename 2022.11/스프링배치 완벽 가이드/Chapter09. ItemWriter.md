# Chapter 09. ItemWriter

## ItemWriter 소개
- 아이템을 청크 단위로 write
  - 단 하나의 아이템을 반환했던 ItemReader와 다르게 아이템 목록(List<T>)을 반환
  - item read->process->read->process->... 반복하다가 하나의 청크가 완성되면 그제서야 write 진행
- 트랜잭션이 적용되지 않은 리소스 처리 시, 롤백이 불가하므로 추가적인 보호조치 필요

## 파일 기반 ItemWriter
### FlatFileItemWriter
- 출력할 리소스와 LineAggregator 구현체로 구성
  - LineAggregator : 객체를 기반으로 출력 문자열 생성
  - FieldExtractor : 제공되는 아이템의 필드에 접근할 수 있게 해줌, 대부분의 LineAggregator 구현체에서 사용
    - BeanWrapperFieldExtractor : getter를 사용해 빈 프로퍼티에 접근
    - PassThroughFieldExtractor : 아이템 바로 반환
- 트랜잭션 주기 내에서 실제 쓰기 작업을 가능한 늦게 수행하도록 설계됨
  - TransactionSynchronizationAdapter의 beforeCommit 메서드를 사용해 메커니즘 구현
  - 디스크로 flush 되면 롤백이 불가하기 때문

#### 형식화된 텍스트 파일
- 출력 파일은 고정 너비이거나 훨씬 복잡한 형식일 수 있음
- FormatterLineAggregator 사용
  - FlatFileItemWriterBuilder 내의 FormattedBuilder로 출력 형식과 추출대상 필드를 순서대로 구성

#### 구분자로 구분된 파일
- DelimitedLineAggregator 사용
  - BeanWrapperFieldExtractor 사용하여 Object 추출, 엘리먼트 사이에 구분자를 넣어 문자열 연결
  - FlatFileItemWriterBuilder가 구분자로 구분된 파일을 생성하는 데 필요한 LineAggregator 리소스를 구성하는 별도 빌더 제공

#### 파일 관리 옵션
- shouldDeleteIfEmpty
  - true : 아무 아이템도 쓰여지지 않았을 경우 스텝이 완료되는 시점에 해당 파일이 삭제됨(헤더나 푸터만 존재하는 경우 역시 item 개수는 0이므로 삭제됨)
  - false : 파일이 생성되고 비어있는 채로 남아 있음(default)
- shouldDeleteIfExists
  - true : 쓰기 작업 대상 출력파일과 이름이 같은 파일이 존재하면 해당 파일을 삭제(default)
  - false : 이미 존재하는 파일이 있으므로 ItemStreamException 발생
- appendAllowed
  - true : shouldDeleteIfExists 플래그 자동으로 false로 설정, 이미 파일이 존재하면 기존 파일에 데이터 추가 / 없으면 새로 생성
    - 여러 스텝이 하나의 출력 파일에 쓰기 작업을 할 경우 유용
  - false : default

### StaxEventItemWriter
- FlatFileItemWriter와 동일하게 chunk 단위로 XML 생성
- 리소스, 루트 엘리먼트 이름, 아이템을 XML 프래그먼트로 변환하는 마샬러로 구성
  - StaxEventItemReader는 리소스, 루트 앨리먼트 이름, XML 입력을 객체로 변환하는 언마샬러로 구성
- XStreamMarshaller 클래스를 사용할 경우 어트리뷰트의 이름을 패키지명을 포함한 클래스의 전체 이름을 사용하므로  
  별칭을 지정한 Map을 setAliases를 통해 제공해주어야 함
  - org.springframework.spring-oxm 과 com.thoughtworks.xstream:xstream 의존성 추가도 필요

## 데이터베이스 기반 ItemWriter
쓰기 작업을 트랜잭션과 분리하는 파일 기반 처리와 달리 물리적 쓰기를 트랜잭션의 일부분으로 포함 가능

### JdbcBatchItemWrite
- JdbcTemplate 사용
  - 한 건마다 SQL을 한 번씩 호출하는 것이 아니라 하나의 청크에 대한 모든 SQL문을 한 번에 실행
- 물음표를 값의 placeholder로 사용하는 방법
  - ```insert into customer(name, age) values (?, ?)```
  - ItemPreparedStatementSetter 인터페이스 구현체 구현하여 JdbcBatchItemWriter 구성
- 네임드 파라미터(:name)를 값의 placeholder로 사용하는 방법
  - ```insert into customer(name, age) values (:name, :age)```
  - PreparedStatement 표기법보다 안전
  - ItemParameterSourceProvider 인터페이스 구현체 구현하여 JdbcBatchItemWriter 구성
    - 실행할 statement의 파라미터를 설정하지 않음. 값을 추출하여 PreparedStatement 객체로 반환하는 역할
    - 스프링에서 구현체를 제공하므로 값을 추출하기 위해 직접 코드를 작성할 필요 없음

### HibernateItemWriter
- 청크가 완료되면 아이템이 HibernateItemWriter로 전달되고 Session.saveOrUpdate 메서드 호출, 모든 아이템이 저장/수정되면 Session flush
- org.springframework.boot:spring-boot-starter-data-jpa 의존성 추가 필요
  - 데이터베이스에 매핑시킬 객체의 애너테이션은 JPA 지원 애너테이션이므로 하이버네이트에서 다른 프레임워크로 변경되어도 코드 변경이 필요없음
- DataSourceTransactionManager 대신 HibernateTransactionManager 사용
- 필수 의존성 : SessionFactory / 필수는 아닌 default가 true인 의존성 : flush 후 Session clear 여부
- 프레임워크가 많은 일을 수행해주어 스프링 배치의 구성은 매우 간단해짐

### JpaItemWriter
- HibernateItemWriter와 거의 동일한 구성
- 데이터베이스 쓰기 작업에 관련하여 많은 일을 수행해주므로 스프링 배치가 할 일이 적음
- 청크가 완료되면 아이템이 JpaItemWriter로 전달되고 저장한 뒤 flush 호출하기 전 아이템마다 EntityManager의 merge 메서드 호출
  - Hibernate에서 HibernateTransactionManager를 생성했으나 Jpa에서는 JpaTransactionManager 생성
- Hibernate에서는 SessionFactory / Jpa에서는 EntityManagerFactory 의존성

## 스프링 데이터의 ItemWriter
### 몽고DB
- MongoItemWriter를 통해 몽고DB 컬렉션에 객체를 문서로 저장할 수 있도록 지원
  - 문자열 타입의 id(관계형 DB에서는 long이었음)
- spring-boot-starter-data-mongodb 의존성 추가, 데이터베이스 관련 설정 application.yml 추가 필요
- 몽고 DB가 ACID 트랜잭션을 지원하지 않아 다른 데이터 저장소처럼 커밋이 발생하기 직전까지 쓰기를 버퍼링하고 가장 마지막 순간에 쓰기 작업을 수행

### 네오4j
- Neo4jItemWriter를 통해 데이터베이스에 레코드 저장
  - UUID 형식의 id
- @NodeEntity 애너테이션으로 해당 클래스가 그래프의 노드임을 ItemWriter에게 알려줌
- @Relationship 애너테이션으로 노드 간의 관계 매핑
- spring-boot-starter-data-neo4j 의존성 추가, 데이터베이스 관련 설정 application.yml 추가 필요

### 피보탈 젬파이어와 아파치 지오드
- 밀리초 단위를 다루는 금융권에서 부정행위를 감지하려면 빠르게 처리가 이루어져야 함. 메모리에 데이터를 캐시 -> 피보탈 젬파이어의 탄생
- 피보탈 젬파이어
  - 오픈소스 버전은 아파치 지오드
  - 인메모리 데이터 그리드, 고성능 분산 HashMap, 모든 데이터를 메모리에 보관
- 스프링 배치와의 연관성 : 어플리케이션 시작 시 비어있는 캐시를 준비 상태로 만듬
- GemfireItemWriter, GemfireTemplate과 Converter로 피보탈 젬파이어에 아이템 저장
- spring-data-gemfire, spring-shell 의존성 추가
  - 스프링 부트가 기본적으로 사용하는 로깅과 젬파이어가 기본적으로 사용하는 로깅 충돌 -> spring-boot-starter-batch에서 로깅 의존성 제외할 것
- @PeerCacheApplication 애너테이션으로 애플리케이션 내에서 피보탈 젬파이어 서비스를 부트스트랩\
  - 외부 메커니즘 사용하는 대신 스프링으로 직접 구성 가능

### 리포지터리
- 스프링 데이터의 Repository 추상화로 스프링 데이터가 지원하는 모든 데이터 저장소에 레코드 저장 가능
- CrudRepository 상속하여 구현
- RepositoryItemWriter를 통해 데이터 저장
- @EnableJpaRepositories를 적용하여 리포지터리가 존재하는 패키지 내의 클래스 지정

## 그밖의 출력 방식을 위한 ItemWriter
### ItemWriterAdapter
- 각각의 아이템 처리 시에 기존 서비스 메서드를 호출하는 작업을 반복
  - ItemWriterAdapter가 호출하는 메서드는 현재 스텝에서 처리 중인 타입의 아규먼트 단 하나만 받아들여야 함
- 두 가지 의존성 필요
  - targetObject : 호출할 메서드를 가지고 있는 스프링 빈
  - targetMethod : 아이템 처리 시 호출할 메서드

### PropertyExtractingDelegatingItemWriter
- ItemWriterAdapter와 동일하게 스프링 서비스의 지정된 메서드 호출
  - ItemWriterAdapter는 처리중인 아이템을 그대로 메서드의 아규먼트로 넘김
  - PropertyExtractingDelegatingItemWriter는 요청한 아이템 중 특정 속성만 뽑아서 아규먼트로 넘기는 것이 가능

### JmsItemWriter
- 둘 이상의 엔드포인트 간 통신하는 메시지 지향적 방식 JMS(Java Messaging Service)
- JmsItemWriter를 통해 JMS 큐에 메시지를 넣음
  - JMS 브로커로는 인메모리 방식으로 간단히 사용할 수 있는 아파치 액티브 MQ 사용 가능
  - MessageConverter :  메시지를 JSON으로 변환
  - JmsTemplate : 스프링 부트가 제공하는 ConnectionFactory는 JmsTemplate과 잘 동작하지 않으므로 CachingConnectionFactory를 활용해 구성해야 함

### SimpleMailMessageItemWriter
- 신규 고객 각각에게 환영 이메일을 보내거나 스팸 메일을 보내고 싶을 때 사용 가능
- SimpleMailMessageItemWriter
  - ItemWriter의 단일 write 메서드 구현
  - 단, 이메일을 보내기 위해 필요한 정보인 제목, 받는 사람 이메일 주소, 보낸 사람 이메일 주소를 포함한 SimpleMailMessage를 확장한 객체의 목록을 아이템으로 취함
  - ItemReader에서부터 SimpleMailMessage를 상속한 아이템을 다루어야 한다는 것을 의미하는 것은 아님. ItemProcessor에서 SimpleMailMessage를 상속한 객체를 반환케 하면 됨
  - MailSender라는 한 가지 의존성만 필요

## 여러 자원을 사용하는 ItemWriter
### MultiResourceItemWriter
- 처리한 레코드 수에 따라 출력 리소스를 동적으로 만듬
- write 메서드 호출 -> 리소스가 생성된 채 열려있는지 확인 -> 아이템을 위임 ItemWriter에 전달 -> 파일에 기록한 아이템 수가 임곗값에 도달했는지 확인 -> 도달했을 경우 파일 닫기
  - 청크 중간에 새 리소스를 생성하지 않음에 유의
- 리소스, delegateItemWriter, 쓰기 작업을 수행할 아이템 수 3가지의 의존성 필요
  - ResourceSuffixCreator를 통해 접미사 지정 가능
- delegateItemWriter는 출력 파일에 대한 직접적인 참조 없이 필요할 경우 multiResourceItemWriter로부터 제공받음

#### 헤더와 푸터 XML 프래그먼트
- XML 파일
  - 파일의 맨 위 또는 맨 아래에 XML 세그먼트 추가
  - StaxWriterCallback 인터페이스로 구현

#### 플랫 파일 내의 헤더와 푸터
- 플랫 파일
  - 파일의 상단 또는 하단에 하나 이상의 레코드 추가
  - 헤더 : FlatFileHeaderCallback 인터페이스로 구현
  - 푸터 : FlatFileFooterCallback 인터페이스로 구현
    - 파일에 쓰기 작업을 수행한 아이템 수를 계산하기 위해 Aspect의 pointcut 확인
      ItemWriterListener.beforeWrite -> Aspect 구현 -> FlatFileItemWriter.open -> Aspect 구현 -> FlatFileItemWriter.write

### CompositeItemWriter
- CompositeItemWriter를 사용하여 여러 장소에 쓰기 작업 가능
  - (이전까지의 예제는 item 처리 시 하나의 장소에 쓰기 작업을 하는 예제)
- 읽기 작업과 처리 작업은 하나의 아이템 당 한 번씩 발생, 쓰기 작업은 chunk 단위로 ItemWriter 구성 순서대로 발생
  - XMLWriter, JDBCWriter 순으로 구성하였다면 읽기 -> 처리 -> 읽기 -> 처리 -> ... -> 청크 사이즈 도달 -> XML 쓰기 -> JDBC 쓰기의 과정을 청크마다 반복
  - XMLWriter는 제대로 쓰기 성공, JDBCWriter 쓰기 실패 -> XMLWriter, JDBCWriter 둘 다 해당 chunk 전체 rollback
  - 10개의 아이템을 XML, JDBC에 각각 썼을 때 JobRepository에 남는 쓰기작업 처리 수는 20이 아니라 10, 아이템 수를 기준으로 기록됨

### ClassifierCompositeItemWriter
- 서로 다른 유형의 아이템을 확인하고 어떤 ItemWriter를 사용해 쓰기 작업을 수행할지 판별 후 적절한 writer에 아이템 전달
- 모든 ItemWriter로 모든 아이템을 쓰는 CompositeItemWriter와 차이 존재
  - A~M으로 시작하는 주에 거주하는 고객은 플랫 파일에 기록, N~Z로 시작하는 주에 거주하는 고객은 데이터베이스에 저장하도록 지정하는 것이 가능해짐
- ClassifierCompositeItemWriter
  - Classifier 구현체 필요
    - classify 단일 메서드로 구성됨 -> 해당 메서드 구현을 통해 어느 곳에 쓰기 작업을 진행할지 판별

#### ItemStream 인터페이스
- open, update, close 세 가지 메서드로 구성
- CompositeItemWriter는 ItemStream 인터페이스를 구현하여 위임 ItemWriter들의 open, update, close 메서드 반복적 호출 가능
- ClassifierCompositeItemWriter는 ItemStream 인터페이스 구현하지 않음
  - open 메서드가 호출되지 않은 상태에서 XMLEventFactory가 생성되거나 XML 쓰기가 시도되어 Exception 발생
  - 스텝 내에서 수동으로 ItemStream을 등록해야 함