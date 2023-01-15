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
### ItemProcessorAdapter
### ScriptItemProcessor
### CompositeItemProcessor

## ItemProcessor 직접 만들기
### 아이템 필터링하기