---
layout: post
title: "JPA Enum 매핑 전략: @Convert vs @Enumerated 실전 비교"
date: 2025-11-05
categories: [Backend, JPA]
tags: [JPA, Enum, AttributeConverter, Spring Boot, Database]
---

## 들어가며

이전 포스팅에서 JPA의 `AttributeConverter`를 활용하여 Enum을 DB의 숫자 코드로 매핑하는 방법을 다루었다. 가계부 프로젝트의 지출 타입(고정지출/변동지출)을 `ExpenseType` Enum으로 정의하고, Converter를 통해 0과 1로 저장했다.

그런데 새로운 요구사항으로 계좌(`Account`) 엔티티에 계좌 상태를 추가하게 되었다. 이번에는 Converter 대신 `@Enumerated` 어노테이션을 사용하기로 결정했다. 왜 같은 Enum 매핑인데 다른 방식을 선택했을까?

이번 포스팅에서는 두 방식을 실전 관점에서 비교하고, 어떤 상황에 어떤 방식을 선택해야 하는지 정리했다.

## 이전 접근: AttributeConverter를 사용한 ExpenseType

### 요구사항

프론트엔드와 API 스펙을 협의하며 지출 타입을 다음과 같이 정의했다.
```json
{
  "expenseType": 0,  // 고정지출
  "amount": 50000
}
```

숫자 코드로 통신하기로 약속했기 때문에, DB에도 0과 1로 저장해야 했다.

### 구현

**ExpenseType Enum**
```java
public enum ExpenseType {
    FIXED(0, "고정지출"),
    VARIABLE(1, "변동지출");

    private final int value;
    private final String description;

    ExpenseType(int value, String description) {
        this.value = value;
        this.description = description;
    }

    public int getValue() {
        return value;
    }

    public static ExpenseType valueOf(int value) {
        return Arrays.stream(values())
                .filter(type -> type.value == value)
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException(
                    "유효하지 않은 지출 타입: " + value));
    }
}
```

**ExpenseTypeConverter**
```java
@Converter
public class ExpenseTypeConverter 
    implements AttributeConverter<ExpenseType, Integer> {
    
    @Override
    public Integer convertToDatabaseColumn(ExpenseType attribute) {
        if (attribute == null) {
            return null;
        }
        return attribute.getValue();
    }

    @Override
    public ExpenseType convertToEntityAttribute(Integer dbData) {
        if (dbData == null) {
            return null;
        }
        return ExpenseType.valueOf(dbData);
    }
}
```

**Entity 적용**
```java
@Entity
public class Expense {
    @Convert(converter = ExpenseTypeConverter.class)
    @Column(nullable = false)
    private ExpenseType expenseType;
}
```

### 실행 결과

**DB 저장 값**
```
| id | expense_type | amount |
|----|--------------|--------|
| 1  | 0            | 50000  |
| 2  | 1            | 30000  |
```

정수 타입으로 저장되어 공간을 절약하고, API 스펙과도 일치했다.

## 새로운 접근: @Enumerated를 사용한 AccountStatus

### 요구사항

계좌 엔티티에 상태 관리 기능을 추가했다. 이번에는 외부 API와 연동할 필요가 없고, 내부적으로만 사용하는 상태값이었다.
```java
// 계좌 상태: 활성, 비활성, 정지
```

### 고민

ExpenseType처럼 Converter를 만들어야 할까? 상태값을 0, 1, 2로 관리해야 할까?

**고민 포인트:**
1. DB에서 직접 쿼리로 확인할 때 0, 1, 2보다는 'ACTIVE', 'INACTIVE'가 명확하다
2. 나중에 상태가 추가될 가능성이 있다
3. 외부 시스템과 숫자 코드로 통신할 필요가 없다
4. 개발자 간 커뮤니케이션에서 가독성이 중요하다

### 구현

**AccountStatus Enum**
```java
public enum AccountStatus {
    ACTIVE("활성"),
    INACTIVE("비활성"),
    SUSPENDED("정지");

    private final String description;

    AccountStatus(String description) {
        this.description = description;
    }

    public String getDescription() {
        return description;
    }
}
```

ExpenseType과 달리 `value` 필드가 없다. Enum 자체의 이름을 DB에 저장할 것이기 때문이다.

**Entity 적용**
```java
@Entity
public class Account {
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private AccountStatus status;
}
```

단 한 줄의 어노테이션으로 끝났다. 별도의 Converter 클래스가 필요 없었다.

### 실행 결과

**DB 저장 값**
```
| id | inst_cd | acct_no      | status   |
|----|---------|--------------|----------|
| 1  | 004     | 123456789    | ACTIVE   |
| 2  | 088     | 987654321    | INACTIVE |
```

Enum의 이름 그대로 VARCHAR 타입으로 저장되었다.

**SQL 쿼리 가독성**
```sql
-- 활성 계좌만 조회
SELECT * FROM account WHERE status = 'ACTIVE';

-- Converter 방식이었다면
SELECT * FROM account WHERE status = 0;  -- 0이 뭐지?
```

## 두 방식의 실전 비교

### 1. 코드 복잡도

**AttributeConverter**
```
Enum 정의 (value 필드 포함)
    ↓
valueOf 메서드 구현
    ↓
Converter 클래스 작성
    ↓
Entity에 @Convert 적용
```

총 3개의 컴포넌트가 필요하다.

**@Enumerated(STRING)**
```
Enum 정의
    ↓
Entity에 @Enumerated 적용
```

Converter 클래스가 필요 없어 코드가 간결하다.

### 2. DB 저장 공간

실제 테스트 결과를 비교했다.

**ExpenseType (INTEGER)**
```
컬럼 타입: INT (4 bytes)
100만 건 기준: 4MB
```

**AccountStatus (VARCHAR)**
```
컬럼 타입: VARCHAR(20)
평균 문자열 길이: 8 (ACTIVE, INACTIVE 평균)
100만 건 기준: 8MB (UTF-8)
```

약 2배의 차이가 있지만, 실무에서는 인덱싱을 고려하면 큰 차이가 아니었다.

### 3. 유지보수성

**시나리오: 새로운 상태 추가**

**Converter 방식**
```java
public enum ExpenseType {
    FIXED(0, "고정지출"),
    VARIABLE(1, "변동지출"),
    SUBSCRIPTION(2, "구독지출");  // 추가
}
```
- Enum만 수정하면 됨
- DB 데이터는 안전 (기존 0, 1과 충돌 없음)

**@Enumerated 방식**
```java
public enum AccountStatus {
    ACTIVE("활성"),
    INACTIVE("비활성"),
    SUSPENDED("정지"),
    DORMANT("휴면");  // 추가
}
```
- Enum만 수정하면 됨
- DB 데이터는 안전
- 마이그레이션 불필요

둘 다 안전하다. 하지만 `EnumType.ORDINAL`을 사용했다면 재앙이 발생한다.

### 4. EnumType.ORDINAL의 위험성

절대 사용하지 말아야 할 방식이다.
```java
// 초기 정의
public enum AccountStatus {
    ACTIVE,    // 0
    INACTIVE   // 1
}

// 나중에 상태 추가
public enum AccountStatus {
    PENDING,   // 0 ← 새로 추가
    ACTIVE,    // 1 ← 기존 0에서 변경됨!
    INACTIVE   // 2 ← 기존 1에서 변경됨!
}
```

**결과:**
```sql
-- 기존 DB 데이터
| id | status |  -- 원래 의미
|----|--------|
| 1  | 0      |  -- ACTIVE였음
| 2  | 1      |  -- INACTIVE였음

-- Enum 순서 변경 후
| id | status |  -- 변경된 의미
|----|--------|
| 1  | 0      |  -- PENDING으로 변함!
| 2  | 1      |  -- ACTIVE로 변함!
```

모든 기존 데이터가 잘못된 상태를 가리키게 된다. 이 버그는 발견하기도 어렵다.

### 5. 디버깅 편의성

**개발 중 로그**

**Converter 방식**
```
expense.expenseType = 0
```
→ 0이 고정지출인지 변동지출인지 매번 코드를 확인해야 한다.

**@Enumerated 방식**
```
account.status = ACTIVE
```
→ 로그만 봐도 의미가 명확하다.

**SQL 쿼리**
```sql
-- Converter: 숫자로 조건 작성
WHERE expense_type = 0

-- @Enumerated: 의미 있는 문자열
WHERE status = 'ACTIVE'
```

협업 시 코드 리뷰나 장애 대응에서 가독성은 중요한 요소였다.

### 6. 외부 시스템 연동

**ExpenseType의 경우**
```json
// 프론트엔드 API 요청
{
  "expenseType": 0,  // 숫자로 약속됨
  "amount": 50000
}
```

숫자 코드가 API 스펙이므로 Converter가 적합했다.

**AccountStatus의 경우**
```java
// 내부 서비스에서만 사용
accountService.activate(accountId);  // → ACTIVE
accountService.suspend(accountId);   // → SUSPENDED
```

외부 노출이 없으므로 가독성 우선이었다.

## 선택 기준 정리

실무 경험을 바탕으로 정리한 선택 기준이다.

### AttributeConverter를 선택하는 경우

1. **레거시 DB와 연동**
```java
   // 기존 DB: 지불 방법이 1, 2, 3으로 저장됨
   @Convert(converter = PaymentMethodConverter.class)
   private PaymentMethod paymentMethod;
```

2. **외부 API 스펙이 숫자 코드**
```java
   // 외부 결제 시스템: 0(카드), 1(계좌), 2(포인트)
   @Convert(converter = PaymentTypeConverter.class)
   private PaymentType paymentType;
```

3. **대용량 테이블의 극한 최적화**
```java
   // 거래 테이블: 수억 건, 매 쿼리마다 조회
   @Convert(converter = TransactionTypeConverter.class)
   private TransactionType type;
```

### @Enumerated(STRING)을 선택하는 경우

1. **내부 도메인 상태 관리**
```java
   @Enumerated(EnumType.STRING)
   private OrderStatus orderStatus;
```

2. **명확성과 가독성이 중요**
```java
   @Enumerated(EnumType.STRING)
   private UserRole role;  // ADMIN, USER, GUEST
```

3. **DB 직접 쿼리/디버깅 필요**
```sql
   -- 운영 중 긴급 상황에서 빠른 확인
   SELECT * FROM orders WHERE status = 'PENDING';
```

4. **상태 추가 가능성**
```java
   // 주문 상태: 계속 추가될 가능성
   public enum OrderStatus {
       PENDING, CONFIRMED, SHIPPED, 
       DELIVERED, CANCELLED  // 나중에 RETURNED 추가 예정
   }
```

## 실전 적용 사례

우리 프로젝트에서 실제로 적용한 기준이다.

**Converter 사용:**
- ExpenseType (지출 타입): 프론트엔드 API 스펙에 0, 1로 명시됨

**@Enumerated 사용:**
- AccountStatus (계좌 상태): 내부 상태 관리
- CategoryType (카테고리 타입): 내부 분류 체계
- AuditStatus (감사 상태): 로깅 및 추적용

## 성능 테스트 결과

실제로 성능 차이가 있는지 테스트했다.

**테스트 환경:**
- 데이터: 100만 건
- DB: MySQL 8.0
- 인덱스: 각 컬럼에 인덱스 생성

**조회 성능 (WHERE 조건)**
```sql
-- INTEGER (Converter)
SELECT * FROM expense WHERE expense_type = 0;
→ 평균 15ms

-- VARCHAR (Enumerated)
SELECT * FROM account WHERE status = 'ACTIVE';
→ 평균 18ms
```

약 3ms(20%) 차이가 있지만, 실무에서는 네트워크나 비즈니스 로직 처리 시간이 훨씬 크기 때문에 무시할 수 있는 수준이었다.

**인덱스 크기**
```
-- INTEGER 인덱스: 16MB
-- VARCHAR(20) 인덱스: 24MB
```

차이는 있지만, 가독성과 유지보수성 대비 감수할 만한 수준이었다.

## 마이그레이션 전략

만약 Converter 방식에서 @Enumerated로 전환해야 한다면?
```sql
-- 1. 새 컬럼 추가
ALTER TABLE expense 
ADD COLUMN expense_type_str VARCHAR(10);

-- 2. 데이터 마이그레이션
UPDATE expense 
SET expense_type_str = CASE expense_type
    WHEN 0 THEN 'FIXED'
    WHEN 1 THEN 'VARIABLE'
END;

-- 3. NOT NULL 제약조건 추가
ALTER TABLE expense 
MODIFY expense_type_str VARCHAR(10) NOT NULL;

-- 4. 기존 컬럼 삭제
ALTER TABLE expense DROP COLUMN expense_type;

-- 5. 컬럼명 변경
ALTER TABLE expense 
RENAME COLUMN expense_type_str TO expense_type;

-- 6. 인덱스 재생성
CREATE INDEX idx_expense_type ON expense(expense_type);
```

무중단 배포를 위해서는 더블 라이팅 전략이 필요하다.

## 결론

Enum 매핑 방식은 정답이 없다. 상황에 따라 적절한 방식을 선택해야 한다.

**기본 원칙:**
- 새 프로젝트, 내부 도메인 → `@Enumerated(EnumType.STRING)`
- 외부 연동, 레거시 → `AttributeConverter`
- 절대 → `EnumType.ORDINAL` 사용 금지

**선택 기준:**
1. 외부 시스템과의 약속된 코드 값이 있는가?
2. 레거시 DB와 호환해야 하는가?
3. 극한의 성능 최적화가 필요한가?

하나라도 '예'라면 Converter를 사용하고, 그렇지 않다면 @Enumerated(STRING)을 사용하는 것이 일반적으로 더 나은 선택이었다.

가독성과 유지보수성은 장기적으로 팀 생산성에 큰 영향을 미친다. 성능 최적화는 필요할 때 하되, 기본은 명확성이 우선이다.

---

## 요약

- JPA에서 Enum을 매핑하는 두 가지 주요 방식은 `AttributeConverter`와 `@Enumerated`이다
- `AttributeConverter`는 숫자나 커스텀 값으로 저장할 수 있어 외부 시스템 연동이나 레거시 DB와 호환성이 필요할 때 유용하다
- `@Enumerated(EnumType.STRING)`은 Enum 이름을 그대로 저장하여 가독성과 안전성이 뛰어나며 대부분의 경우 권장된다
- `EnumType.ORDINAL`은 Enum 순서 변경 시 데이터가 깨지므로 절대 사용하지 말아야 한다
- 실무에서는 API 스펙이 숫자로 정해진 경우 Converter를, 내부 도메인 상태 관리는 @Enumerated를 사용했다
- 성능 차이는 크지 않지만, 가독성과 유지보수성은 장기적으로 더 큰 가치를 제공한다