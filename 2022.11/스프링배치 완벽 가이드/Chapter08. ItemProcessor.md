# Chapter 08. ItemProcessor

스프링 배치 내에서 입력 데이터를 이용해 어떤 작업을 수행하는 컴포넌트

## ItemProcessor 소개
- ItemProcessor에서 진행할 수 있는 작업들
  - 입력의 유효성 검증 : ItemReader에서 ValidatingItemReader 클래스를 서브클래싱하여 수행 -> 스프링 배치에서 제공하는 ItemReader에서는 제공하지 않음 -> ItemProcessor에서 진행
  - 기존 서비스의 재사용 : ItemProcessorAdapter를 통해 제공
  - 스크립트 실행 : ScriptItemProcessor를 통해 제공
  - ItemProcessor의 체인 : 순서대로 실행 될 ItemProcessor 목록 만들기 가능
- ItemProcessor가 null을 반환할 경우, 이후 진행될 예정이었던 ItemProcessor, ItemWriter 모두 호출되지 않음
- ItemProcessor는 fault tolerant로 인해 두 번씩 호출될 수 있으므로 멱등해야 함
- process 라는 단일 메소드 지님
  - 입력 타입은 ItemReader의 반환 타입, 출력 타입은 ItemWriter의 입력 타입이어야 함

## 스프링 배치의 ItemProcessor 사용하기
ItemProcessor 부분은 비즈니스 요구 사항에 따라 서로 달라지는 부분  
직접 처리 로직을 개발하게 하거나 이미 존재하는 로직을 래핑하는 정도의 기능 제공

### ValidatingItemProcessor
#### 입력 데이터의 유효성 검증
- ItemProcessor의 구현체 ValidatingItemProcessor
  - 프로세스 처리 전 유효성 검증 수행
  - 오류 발생 시 ValidationException 발생
- spring-boot-starter-validation
  - JSR-303 유효성 검증 도구의 하이버네이트 구현체를 가져와 애너테이션으로 유효성 검증 규칙 적용
- BeanValidatingItemProcessor : 각 아이템을 검증하는 매커니즘 제공
  - JSR-303을 활용해 유효성 검증을 제공하는 ValidationItemProcessor를 상속한 ItemProcessor
  - Validator 구현체를 통해 유효성 검증
    - validate(T value)라는 단일 메서드 지님
  - JSR-303 사양에 따르는 Validator 객체를 생성한다는 점에서 특별, 스프링의 자체적인 Validator와 스프링 배치의 Validator는 다름에 유의
- 유효성 검증 기능을 직접 구현하는 경우 : ValidatingItemProcessor 적용
  - 직접 구현한 Validator 인터페이스 구현체 주입
  - 재시작 시 상태를 유지하려면 유효성 검증기는 ItemStreamSupport를 상속하여 ItemStream 인터페이스를 구현할 것
    - open 메서드 : 이전 Execution에 저장된 내용 확인 후 저장되어 있을 경우 해당 값으로 원복
    - update 메서드 : 오류 발생 시 현재 상태를 ExecutionContext에 저장

### ItemProcessorAdapter
- 이미 개발된 서비스가 ItemProcessor 역할을 하도록 함
  - TargetObject, TargetMethod 필수 설정 필요

### ScriptItemProcessor
- ItemProcessorAdapter의 기능과 동일하나 자바 서비스 로직을 수행하는 대신에 자바스크립트 로직을 수행

### CompositeItemProcessor
- ItemProcessor를 체인처럼 연결
  - 체인처럼 연결된 여러 ItemProcessor 중 하나의 ItemProcessor라도 null을 반환하면 해당 아이템은 더이상 처리되지 않음
  - ItemProcessor1, ItemProcessor2가 연결되어 있다고 할 때 ItemProcessor1의 반환값이 ItemProcessor2의 입력값이 되어야 함
- 여러 ItemProcessor의 래퍼 역할
- ClassifierCompositeItemProcessor
  - classify 메서드 : 해당 아이템을 처리할, 조건에 맞는 ItemProcessor 반환하여 위임

## ItemProcessor 직접 만들기
### 아이템 필터링하기
- 아이템 레코드가 ItemProcessor를 통과하며 null을 반환하기만 하면 필터링됨
  - 필터링 된 레코드의 수는 JobRepository에 저장됨
- 데이터가 기술적으로 유효한지 판단하여 예외를 사용한 아이템 건너뛰기와는 다른 점 존재
  - 비즈니스 규칙에 따른 필터링은 아이템 건너뛰기보다는 스텝 쪽에서 필터링으로 처리