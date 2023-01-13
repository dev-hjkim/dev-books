# Chapter 07. ItemReader
그 어떤 프로그램이든 기반에는 데이터를 읽고, 처리하고, 기록하는 기능이 존재  
스프링 배치에서는 ItemReader, ItemProcessor, ItemWriter를 통해 이러한 특징이 극명히 드러나고 있음  
스프링 배치는 플랫 파일, XML, 다양한 데이터베이스, 커스텀한 Reader를 만드는 방법 등 모든 유형의 입력 데이를 처리할 수 있는 표준 방법 제공

## ItemReader 인터페이스
- 전략 인터페이스
- read 라는 단일 메서드 정의
  - 스텝 내에서 처리할 아이템 한 개 반환
- 스텝은 청크 내의 데이터가 몇 개나 처리되었는지 관리, ItemProcessor, ItemWriter로 넘겨짐

## 파일 입력
### 플랫 파일
- 파일 내의 데이터 포맷이나 의미를 정의하는 메타데이터 미존재
- 스프링 배치에서 파일을 읽을 때 사용하는 컴포넌트, FlatFileItemReader
  - 읽어들일 대상 파일인 Resource와 LineMapper 인터페이스 구현체로 구성
  - LineMapper의 구현체 중 하나인 DefaultLineMapper
    - LineTokenizer : 파일에서 가져온 데이터 한 줄을 파싱해 FieldSet으로 만듬
    - FieldSetMapper : FieldSet을 도메인 객체로 매핑

#### 고정 너비 파일
- 동일한 스텝 내에서 FlatFileItemReader를 사용하는 경우 각 리더의 상태를 저장하는 작업이 서로에게 영향을 주지 않도록 하는데 name 속성 필요
- FixedLengthTokenizer 구현체 사용, 컬럼 이름과 Range 객체의 배열을 지정
  - 그 외 FieldSetFactory : 기본적으로 DefaultFieldSetFactory, FieldSet 생성하는 데 사용
  - strict 플래그 : 기본적으로 true, 정의된 파싱 정보보다 더 많은 항목이 포함되어 있을 경우 예외 던지도록 하는 데 사용
- targetType 메서드 호출로 BeanWrapperFieldSetMapper 생성
  - LineTokenizer에 구성된 컬럼 이름들로 setter 메소드를 호출하는 방식
- FlatFileItemReader와 FixedLengthTokenizer를 사용하여 고정 너비 파일 처리

#### 필드가 구분자로 구분된 파일
- 메타데이터 파일 내 소량 제공
- DelimitedLineTokenizer 구현체 사용
  - 구분자 설정 가능, 기본값 : 쉼표(,)
  - 인용 문자로 사용할 값 설정 가능, (ex. #, " 등등)
- FieldSetMapper 인터페이스 구현을 통해 FieldSet의 각 필드를 도메인 객체에 커스텀하게 매핑 가능

#### 커스텀 레코드 파싱
- FieldSetMapper 외 LineTokenizer도 커스텀하게 구현 가능
- 용례
  - 특이한 파일 포맷 파싱
  - 엑셀 워크시트 같은 서드파티 파일 포맷 파싱
  - 특수한 타입 변환 요구 조건 처리

#### 여러 가지 레코드 포맷
- 이전 예제에서는 파일 내부 레코드가 모두 동일한 형식, DefaultLineMapper 사용
  - LineTokenizer 하나, FieldSetMapper 하나 사용
- 하나의 파일 내부에 여러 레코드 포맷이 존재하는 경우 PatternMatchingCompositeLineMapper 사용
  - LineTokenizer로 구성된 Map, FieldSetMapper로 구성된 Map
  - Map의 키 값은 레코드의 패턴
- Double, Date 타입의 경우 커스텀 FieldSetMapper 필요
  - BeanWrapperFieldSetMapper는 특수 타입의 필드를 변환할 수 없음(readDate, readDouble 등으로 구현)

#### 여러 줄에 걸친 레코드
- Customer 객체, Transaction 객체 사이에 종속 관계가 있는 경우
  - 즉, Customer 내부에 List\<Transaction>이 들어갈 수 있는 경우
- 레코드가 언제 끝나는지 알 수 없음. (footer가 있는 경우는 알기 쉬우나, 보통의 경우 footer가 주어지지 않음)
- 다음 Customer 객체가 나올 때까지 Transaction List에 데이터를 넣도록 CustomItemReader에 로직 추가
- ItemReader가 아닌 ItemStreamReader 구현
  - read 메서드 : Customer와 Transaction 레코드를 읽어들여 조합하는 역할
  - peek 메서드 : 레코드를 미리 읽어두는 역할, 읽어들였지만 아직 처리하지 않았을 경우 동일 레코드 다시 반환
- 커밋카운트는 Customer를 기준으로 카운트 됨

#### 여러 개의 소스
- 동일한 포맷으로 작성 된 복수개의 파일을 처리하는 경우
- 스프링에서 MultiResourceItemReader 제공, ItemReader를 매핑하지만, 리소스 정의를 자식 ItemReader에게 맡기지 않음
  - 리더의 이름을 전달받음
  - Resource 객체의 배열 전달받음(읽어들여야 할 파일 목록)
  - 실제 작업을 위임할 컴포넌트 전달받음
    - ItemStreamReader 인터페이스로 충분하지 않음. 하위 인터페이스인 ResourceAwareItemReaderItemStream 구현필요
    - setResource 메서드 추가 구현해야 함
- customerFile1.csv, customerFile2.csv, customerFile3.csv 로 배치 실행 중 customerFile2.csv에서 에러 발생, 이때 customerFile4.csv 추가
  - Expected : 재시작 했을 때 동일하게 customerFile1.csv, customerFile2.csv, customerFile3.csv 재실행
  - Actual : customerFile1.csv, customerFile2.csv, customerFile3.csv, customerFile4.csv 실행
  - 이러한 상황으로부터의 보호 방법 : 배치 실행 시 사용할 디렉터리를 별도로 생성하여 관리

### XML
- 파일 내 데이터를 설명할 수 있는 태그를 사용하여 파일에 포함된 데이터를 완벽히 설명
- DOM 파서 또는 SAX 파서를 사용
  - DOM 파서 : 전체 파일을 메모리에 트리 구조로 읽어들임
    - 성능상 큰 부하, 배치 처리에서 유용하지 않음
  - SAX 파서 : XML 문서 내 각 섹션을 독립적으로 파싱하는 기능 제공
    - 아이템 기반 읽기 작업과 직접적으로 연관, 한 번에 처리해야 할 아이템을 나타내는 파일 내 각 세션을 읽을 수 있음
- 스프링 배치는 파일 내 미리 지정한 XML 프래그먼트를 만날 때마다 단일 레코드로 간주, 처리 대상 아이템으로 변환
- StaxEventItemReader 사용
  - 각 XML 프래그먼트의 루트 엘리먼트 정의 필요
  - 리소스 전달 필요
  - Unmarshaller 구현체 전달 필요, XML을 도메인 객체로 변환하는 데 사용됨
    - Castor, JAXB, JiBX, XML, Beans, XStream 구현체 제공됨

### JSON
- JsonItemReader, 실제 파싱 작업은 JsonObjectReader 구현체에 위임됨(Jackson, Gson 2개의 파싱 엔진 사용)
  - 파싱에 사용할 JsonObjectReader 필요
  - 리소스 전달 필요
  - (Optional) 입력이 반드시 존재해야 하는지를 나타내는 플래그, 상태 저장 여부를 나타내는 플래그, 현재의 ItemCount 구성 가능
- ObjectMapper : Jackson이 JSON을 읽고 쓰는 데 사용하는 주요 클래스
  - 보통 ObjectMapper를 생성하는 코드를 작성할 필요 없으나, 날짜 형식 지정이 필요한 경우 생성하여 형식 지정 필요
- JacksonJsonObjectReader, JsonObjectReader의 구현체
  - 반환할 클래스 필요
  - ObjectMapper 필요(ex. 날짜 형식 커스터마이징 한 ObjectMapper)

## 데이터베이스 입력
### JDBC
- JdbcTemplate 사용 시 배치에서는 대용량의 전체 데이터를 한 번에 메모리에 적재하게 되는 문제 발생
  - 처리할 만큼의 레코드만 로딩하는 2가지 기법 제공
    - 커서, ResultSet으로 구현되며 ResultSet이 open되면 next() 메서드를 호출할 때마다 DB에서 레코드를 가져와 반환
    - 페이징, 청크 크기만큼의 레코드를 가져옴, 한개의 row씩 읽어들이는 커서와 달리 청크 크기 만큼(다수)의 레코드를 읽어들임

#### JDBC 커서 처리
- JdbcCursorItemReader 사용
  - ResultSet을 생성하며 커서를 열고 read 메서드를 호출할 때마다 도메인 객체로 매핑할 로우를 가져옴
  - 데이터 소스, 실행할 쿼리, 사용할 rowMapper 구현체와 같은 최소한 3가지의 의존성 필요
  - 실행할 쿼리에 파라미터를 설정하려면 PreparedStatementSetter 사용(파라미터를 SQL문에 매핑하는 역할 담당)
- 단점
  - 백만 단위의 레코드를 처리할 때 매번 요처을 할 때마다 네트워크 오버헤드가 추가됨
  - thread safe가 보장되지 않으므로 다중 스레드 환경에서 사용 불가

#### JDBC 페이징 처리
- 스프링 배치에서 페이지라 일컬어지는 청크로 결과 목록을 반환
- 레코드 처리 자체에는 커서 처리 방식과 다를 바 없음
  - 한 번에 SQL 쿼리 하나를 실행해 레코드 하나씩 가져오는(커서 방식) 대신 각 페이지마다 새로운 쿼리를 실행하여 결과를 한꺼번에 메모리에 적재함
- PagingQueryProvider 인터페이스의 구현체 필요
  - DB마다 다른 구현체를 제공함. 스프링 배치가 구현해 둔 구현체 사용하는 방법
  - SqlPagingQueryProviderFactoryBean을 사용하여 데이터베이스가 어떤 것인지 감지해 적절한 구현체 반환하면 그 구현체를 사용하는 방법
  - sortKey 중요!
    - 쿼리를 실행할 때마다 동일한 정렬 순서를 보장해야 함
    - 중복되지 않아야 함. 쿼리 생성 과정에서 정렬 키를 사용하기 때문
- JdbcPagingItemReader
  - 데이터 소스 필요
  - PagingQueryProvider 구현체 필요
  - RowMapper 구현체 필요
  - 페이지의 크기 필요

### 하이버네이트
- 자바 ORM 기술
- 커밋할 때 세션을 flush 하는 기능 제공(캐시에 계속 아이템을 쌓게 되면 OutOfMemoryException 발생)

#### 하이버네이트로 커서 처리하기
- spring-boot-starter-jpa 의존성 필요
- DataSource 커넥션과 하이버네이트 세션을 아우르는 TransactionManager 필요
  - DefaultBatchConfigurer.getTransactionManager() 오버라이드하여 BatchConfigurer의 커스텀 구현체 구현
- HibernateCursorItemReader를 사용하여 데이터 읽어들임
  - 이름, SessionFactory, 쿼리 문자열, 쿼리에 사용할 파라미터 필요

#### 하이버네이트를 사용해 페이징 기법으로 데이터베이스에 접근하기
- HibernatePagingItemReader
  - 사용할 페이지 크기 지정 필요

### JPA
- JPA는 아이템 조회 시 커서 기반 방법 미제공
- JpaPagingItemReader
  - 이름, entityManager, 실행할 쿼리, 파라미터 필요로 함
  - 실행할 쿼리를 Query API(JpaQueryProvider 인터페이스 구현)로 지정하는 방법도 존재

### 저장 프로시저
- StoredProcedureItemReader
  - JdbcCursorItemReader 바탕으로 설계됨
  - 이름, 데이터 소스, RowMapper, PreparedStatementSetter 구성 필요
  - SQL을 지정하는 대신 호출할 프로시저의 이름 지정

### 스프링 데이터
- 스프링 데이터의 목적 : 기본적인 데이터 저장소의 특징을 유지하면서도, 친숙하고 일관된 스프링 기반의 데이터 접근 프로그래밍 모델을 제공하는 것
- NoSQL, SQL 모두에 대해 고유 기능 접근 가능, 일관된 추상화의 집합 제공

#### 몽고 DB
- 높은 가용성과 확장성 : 기본적으로 replication 제공(가용성), 샤딩 제공(확장성)
- 지리공간정보 지원 : 특정 지점이 어떤 경계 내에 속하는지 결정하는 것 같은 질의 지원
  - (?????)
- MongoItemReader
  - 페이지 기반 ItemReader
  - MongoOperations 구현체, name, targetType, JSON 기반 쿼리 또는 Query 인스턴스 필요
  - 그 외 정렬, 힌트, 결과에 포함할 필드 목록, 질의할 몽고 DB 컬렉션 등 설정 가능
- org.springframework.boot:spring-boot-starter-data-mongodb 의존성 필요
- DB 설정, application.yml, spring.data.mongodb.databse : {db 이름} 프로퍼티 추가 필요

#### 스프링 데이터 리포지터리
- RepositoryItemReader
  - PagingAndSortingRepository를 통해 페이징 쿼리 실행, 따라서 이를 상속하는 리포지터리 필요함
  - 스프링 데이터가 리포지터리를 지원하는 어떤 데이터 저장소건 상관없이 질의 수행 가능
  - 페이징 메커니즘을 작동시키기 위해서는 Pageable 파라미터 추가 필요
    - 한 페이지만큼 요청하는 데 필요한 파라미터인 페이지 크기, 오프셋, 페이지 번호, 정렬 옵션 캡슐화

## 기존 서비스
- 배치 처리에서 기존 애플리케이션의 코드를 사용하는 방법
- ItemReaderAdapter
  - 호출 시마다 반환되는 객체는 ItemReader가 반환하는 객체로, 이 객체는 ItemProcessor -> ItemWriter로 전달됨
    - Collection을 반환하는 경우 단일 아이템으로 ItemProcessor, ItemWriter로 전달되어야 함
  - 입력 데이터를 모두 처리했을 때에는 반드시 null 반환해야 함

## 커스텀 입력
- 커스텀 ItemReader를 구현하였을 때 잡 실행 시마다 상태를 유지하는 법
  - JobRepository에 리더의 상태를 저장하여 이전에 종료된 지점부터 리더를 다시 시작하게 하려면 ItemStream 인터페이스 구현 필요
  - open, update, close 메서드로 구성
    - open : ItemReader에서 필요한 상태를 초기화하기 위해 호출(이전 상태 복원, 데이터베이스 연결 포함)
    - update : 잡의 상태 갱신
    - close : 리소스를 닫는 데 사용
  - ItemStreamSupport를 상속하면 컴포넌트의 이름으로 고유 키를 생성하는 getExecutionContextKey라는 유틸리티 메서드 제공
    - ItemStream 인터페스도 구현해줌

## 에러 처리

### 레코드 건너뛰기
### 잘못된 레코드 로그 남기기
### 입력이 없을 때의 처리
