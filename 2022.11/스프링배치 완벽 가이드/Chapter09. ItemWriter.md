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
### 네오4j
### 피보탈 젬파이어와 아파치 지오드
### 리포지터리

## 그밖의 출력 방식을 위한 ItemWriter
### ItemWriterAdapter
### PropertyExtractingDelegatingItemWriter
### JmsItemWriter
### SimpleMailMessageItemWriter

## 여러 자원을 사용하는 ItemWriter
### MultiResourceItemWriter
#### 헤더와 푸터 XML 프래그먼트
#### 플랫 파일 내의 헤더와 푸터
### CompositeItemWriter
### ClassifierCompositeItemWriter
#### ItemStream 인터페이스