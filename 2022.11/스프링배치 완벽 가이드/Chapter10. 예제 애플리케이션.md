# Chapter 10. 예제 애플리케이션

## 거래명세서 잡 검토하기
- 매월 말 고객은 거래명세서를 전달받음. 거래명세서는 하기의 내용을 포함
  - 자신의 모든 계좌
  - 지난 달에 발생한 모든 거래
  - 계좌에 입금된 총 금액
  - 계좌에서 차감된 총 금액
  - 현재 잔액

## 새 프로젝트 초기 구성하기
- Spring Initializer를 통해 새 프로젝트 생성
  - Batch, JDBC, HSQLDB 등의 의존성 추가
- main 메서드를 지닌 class에 @EnableBatchProcessing 애너테이션 추가

## 갱신할 고객 정보 가져오기
- 레코드 유형 1, 2, 3에 따라 데이터베이스에 저장된 고객정보를 업데이트
    - 각 유형에 따른 LineTokenizer 적용
    - PatternMatchingCompositeLineTokenizer

### 고겍 ID 유효성 검증하기
- Validator 구현체를 전달하여 ItemProcessor 구성

### 고객 정보 갱신
- 세 개의 서로 다른 ItemWriter 구현체에게 처리를 위임할 수 있게 해주는 ClassifierCompositeItemWriter 사용
  - 적절한 ItemWriter를 반환해주는 Classifier 구현 필요

## 거래 정보 가져오기
- xml 데이터 파일을 Transaction 도메인 객체에 매핑
  - @XmlRootElement
  - @XmlJavaTypeAdapter
    - 형변환을 위해 XmlAdapter를 상속받은 JaxbDateSerializer 구현 필요

### 거래 정보 읽어오기
- Xml 파일을 읽을 것이므로 StaxEventItemReader 사용

### 거래 정보 기록하기
- 읽어들인 아이템을 DB에 저장하는 역할을 수행하므로 JdbcBatchItemWriter 사용

## 잔액에 거래 내역 적용하기
### 거래 데이터 읽어오기
### 계좌 잔액 갱신하기
## 월별 거래명세서 생성하기
### 거래명세서 데이터 가져오기
### Statement 객체에 계좌 정보 추가하기
### 거래명세서 생성하기