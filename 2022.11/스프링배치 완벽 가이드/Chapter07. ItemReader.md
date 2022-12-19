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
#### 여러 개의 소스

### XML

### JSON

## 데이터베이스 입력

### JDBC
#### JDBC 커서 처리
#### JDBC 페이징 처리

### 하이버네이트
#### 하이버네이트로 커서 처리하기
#### 하이버네이트를 사용해 페이징 기법으로 데이터베이스에 접근하기

### JPA

### 저장 프로시저

### 스프링 데이터
#### 몽고 DB
#### 스프링 데이터 리포지터리

## 기존 서비스

## 커스텀 입력

## 에러 처리

### 레코드 건너뛰기
### 잘못된 레코드 로그 남기기
### 입력이 없을 때의 처리
