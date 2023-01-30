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
#### 구분자로 구분된 파일
#### 파일 관리 옵션

### StaxEventItemWriter

## 데이터베이스 기반 ItemWriter
### JdbcBatchItemWrite
### HibernateItemWriter
### JpaItemWriter

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