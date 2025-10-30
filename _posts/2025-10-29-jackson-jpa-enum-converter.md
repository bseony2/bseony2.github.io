---
layout: post
title: "Jackson과 JPA의 Enum 변환 전략 - @JsonValue, @JsonCreator, @Converter 완벽 가이드"
date: 2025-10-29 21:00:00 +0900
categories: [Spring, JPA, Jackson]
tags: [enum, json, converter, jackson, jpa, attribute-converter]
---

## 들어가며

프로젝트에서 Enum 타입을 사용할 때, 클라이언트와 서버 간 데이터 교환(JSON), 그리고 데이터베이스 저장 시 각각 다른 형태로 변환해야 하는 경우가 있었다. 예를 들어, 고정지출과 변동지출을 구분하는 `ExpenseType` Enum을 다음과 같이 관리해야 했다:

- **API 통신**: 정수 코드 (0, 1)
- **Java 코드**: Enum 타입 (FIXED, VARIABLE)
- **데이터베이스**: 정수 컬럼 (0, 1)

이 과정에서 Jackson의 `@JsonValue`, `@JsonCreator`와 JPA의 `@Converter`, `AttributeConverter`를 활용했고, 그 과정을 정리했다.

---

## 전체 데이터 흐름도
```
[Frontend] ─── JSON (0, 1) ─── [Spring Controller] 
                                       ↓
                                 @JsonCreator
                                       ↓
                              [Service Layer]
                           (Enum: FIXED, VARIABLE)
                                       ↓
                               @Converter
                                       ↓
                          [Database] (INTEGER: 0, 1)

────────────────────────────────────────────────────────

[Database] (INTEGER: 0, 1)
              ↓
      AttributeConverter
              ↓
    [Service Layer] (Enum)
              ↓
         @JsonValue
              ↓
    [Frontend] (JSON: 0, 1)
```

---

## 1. Jackson의 Enum 변환: @JsonValue와 @JsonCreator

### 문제 상황

기본적으로 Jackson은 Enum을 문자열로 직렬화한다.
```java
public enum ExpenseType {
    FIXED,
    VARIABLE
}

// 기본 직렬화 결과
{"expenseType": "FIXED"}  // 문자열
```

하지만 우리는 정수 코드값으로 통신하고 싶었다.
```json
{"expenseType": 0}
```

### @JsonValue: Enum → JSON 변환

`@JsonValue`는 Enum을 JSON으로 직렬화할 때 사용할 값을 지정한다.
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

    @JsonValue  // 이 메서드의 반환값으로 직렬화
    public int getValue() {
        return value;
    }
}
```

**동작 방식:**
```java
ExpenseType type = ExpenseType.FIXED;
// Jackson 직렬화 → {"expenseType": 0}
```

### @JsonCreator: JSON → Enum 변환

`@JsonCreator`는 JSON 값을 받아 Enum으로 역직렬화하는 정적 메서드를 지정한다.
```java
@JsonCreator
public static ExpenseType fromJson(int value) {
    for (ExpenseType type : values()) {
        if (type.value == value) {
            return type;
        }
    }
    throw new IllegalArgumentException("유효하지 않은 지출 구분 코드: " + value);
}
```

**동작 방식:**
```java
// JSON: {"expenseType": 0}
// Jackson 역직렬화 → ExpenseType.FIXED
```

### 전체 코드
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

    @JsonValue
    public int getValue() {
        return value;
    }

    @JsonCreator
    public static ExpenseType fromJson(int value) {
        return Arrays.stream(values())
                .filter(type -> type.value == value)
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException(
                    "유효하지 않은 지출 구분 코드: " + value));
    }

    public String getDescription() {
        return description;
    }
}
```

---

## 2. JPA의 Enum 변환: @Converter와 AttributeConverter

### 문제 상황

JPA는 기본적으로 Enum을 두 가지 방식으로 저장한다:
```java
@Enumerated(EnumType.STRING)   // "FIXED", "VARIABLE" (문자열)
@Enumerated(EnumType.ORDINAL)  // 0, 1 (순서값)
```

하지만 `EnumType.ORDINAL`은 위험하다. Enum의 순서가 바뀌면 데이터가 깨진다.
```java
// 기존
public enum ExpenseType {
    FIXED,      // 0
    VARIABLE    // 1
}

// 새로운 값 추가 시
public enum ExpenseType {
    SAVINGS,    // 0 ← 기존 FIXED 데이터와 충돌!
    FIXED,      // 1
    VARIABLE    // 2
}
```

따라서 **코드값을 명시적으로 관리**하고, 커스텀 Converter를 사용해야 했다.

### @Converter 어노테이션
```java
@Converter(autoApply = true)
public class ExpenseTypeConverter 
        implements AttributeConverter<ExpenseType, Integer> {
    // ...
}
```

**@Converter 속성:**

| 속성 | 설명 | 기본값 |
|------|------|--------|
| `autoApply` | 해당 타입을 가진 모든 필드에 자동 적용 여부 | `false` |

- `autoApply = true`: 전역 적용 (권장)
- `autoApply = false`: `@Convert` 어노테이션으로 명시적 지정 필요
```java
// autoApply = false인 경우
@Convert(converter = ExpenseTypeConverter.class)
private ExpenseType expenseType;
```

### AttributeConverter 인터페이스
```java
public interface AttributeConverter<X, Y> {
    // Entity → Database
    Y convertToDatabaseColumn(X attribute);
    
    // Database → Entity
    X convertToEntityAttribute(Y dbData);
}
```

- `X`: Java 엔티티 속성 타입 (ExpenseType)
- `Y`: 데이터베이스 컬럼 타입 (Integer)

### 구현 예시
```java
@Converter(autoApply = true)
public class ExpenseTypeConverter 
        implements AttributeConverter<ExpenseType, Integer> {

    /**
     * Entity의 Enum → Database의 Integer
     */
    @Override
    public Integer convertToDatabaseColumn(ExpenseType attribute) {
        if (attribute == null) {
            return null;
        }
        return attribute.getValue();
    }

    /**
     * Database의 Integer → Entity의 Enum
     */
    @Override
    public ExpenseType convertToEntityAttribute(Integer dbData) {
        if (dbData == null) {
            return null;
        }
        return ExpenseType.valueOf(dbData);
    }
}
```

### Entity에서 사용
```java
@Entity
public class Expense {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    // autoApply = true이므로 @Convert 불필요
    @Column(name = "expense_type", nullable = false)
    private ExpenseType expenseType;
    
    private BigDecimal amount;
}
```

**데이터베이스 스키마:**
```sql
CREATE TABLE expense (
    id BIGINT PRIMARY KEY,
    expense_type INT NOT NULL,  -- 0 또는 1 저장
    amount DECIMAL(15, 2)
);
```

---

## 3. 전체 흐름 예시

### 요청 흐름 (Frontend → Database)
```java
// 1. Frontend에서 요청
POST /api/expenses
{
  "expenseType": 0,
  "amount": 50000
}

// 2. Controller에서 수신
@PostMapping("/api/expenses")
public ResponseEntity<ExpenseResponse> createExpense(
        @RequestBody ExpenseRequest request) {
    // request.getExpenseType() → ExpenseType.FIXED
    // @JsonCreator가 0을 ExpenseType.FIXED로 변환
}

// 3. Service에서 처리
@Service
public class ExpenseService {
    public ExpenseResponse createExpense(ExpenseRequest request) {
        Expense expense = Expense.of(
            request.getExpenseType(),  // ExpenseType.FIXED
            request.getAmount()
        );
        // ...
    }
}

// 4. Repository에 저장
expenseRepository.save(expense);
// AttributeConverter가 ExpenseType.FIXED → 0으로 변환
// Database에는 0이 저장됨
```

### 응답 흐름 (Database → Frontend)
```java
// 1. Database에서 조회
SELECT * FROM expense WHERE id = 1;
-- expense_type = 0

// 2. Entity로 변환
// AttributeConverter가 0 → ExpenseType.FIXED로 변환

// 3. Service에서 DTO 변환
ExpenseResponse response = ExpenseResponse.from(expense);
// response.getExpenseType() → ExpenseType.FIXED

// 4. Controller에서 응답
return ResponseEntity.ok(response);
// @JsonValue가 ExpenseType.FIXED → 0으로 변환

// 5. Frontend에서 수신
{
  "id": 1,
  "expenseType": 0,
  "amount": 50000
}
```

---

## 4. 실무 고려사항

### Null 처리
```java
@Override
public Integer convertToDatabaseColumn(ExpenseType attribute) {
    if (attribute == null) {
        return null;  // 또는 기본값 설정
    }
    return attribute.getValue();
}
```

### 예외 처리
```java
@JsonCreator
public static ExpenseType fromJson(int value) {
    return Arrays.stream(values())
            .filter(type -> type.value == value)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                String.format("유효하지 않은 지출 구분 코드: %d. " +
                    "허용 범위: %s", value, 
                    Arrays.toString(Arrays.stream(values())
                        .mapToInt(ExpenseType::getValue)
                        .toArray()))
            ));
}
```

### 테스트 코드
```java
@Test
void json_직렬화_테스트() throws Exception {
    ExpenseType type = ExpenseType.FIXED;
    String json = objectMapper.writeValueAsString(type);
    
    assertThat(json).isEqualTo("0");
}

@Test
void json_역직렬화_테스트() throws Exception {
    String json = "0";
    ExpenseType type = objectMapper.readValue(json, ExpenseType.class);
    
    assertThat(type).isEqualTo(ExpenseType.FIXED);
}

@Test
void db_변환_테스트() {
    ExpenseTypeConverter converter = new ExpenseTypeConverter();
    
    // Enum → Integer
    assertThat(converter.convertToDatabaseColumn(ExpenseType.FIXED))
        .isEqualTo(0);
    
    // Integer → Enum
    assertThat(converter.convertToEntityAttribute(0))
        .isEqualTo(ExpenseType.FIXED);
}
```

---

## 5. 장단점 분석

### @JsonValue / @JsonCreator 방식

**장점:**
- 클라이언트와의 통신을 정수 코드로 간결하게 유지
- 네트워크 페이로드 감소
- 프론트엔드에서 숫자로 간단하게 처리 가능

**단점:**
- API 문서에 코드값 의미 명시 필요
- 디버깅 시 코드값만 보면 의미 파악이 어려움

### AttributeConverter 방식

**장점:**
- 코드값을 명시적으로 관리하여 Enum 순서 변경에 안전
- null 처리 및 예외 처리 커스터마이징 가능
- 데이터베이스 공간 절약 (INT vs VARCHAR)

**단점:**
- 추가 코드 작성 필요
- 디버깅 시 변환 로직 추적 필요

---

## 6. 대안: @Enumerated(EnumType.STRING)

간단한 경우 문자열 저장도 고려할 수 있다.
```java
@Enumerated(EnumType.STRING)
@Column(name = "expense_type", length = 20)
private ExpenseType expenseType;
```

**장점:**
- 별도 Converter 불필요
- 데이터베이스에서 가독성 좋음

**단점:**
- 스토리지 낭비 (INT: 4byte vs VARCHAR: 가변)
- Enum 이름 변경 시 마이그레이션 필요

---

## 정리

- **@JsonValue**: Enum을 JSON으로 직렬화할 때 사용할 값을 지정
- **@JsonCreator**: JSON 값을 Enum으로 역직렬화하는 로직 정의
- **@Converter**: JPA에서 Enum과 데이터베이스 타입 간 변환 담당
- **autoApply = true**: 해당 타입의 모든 필드에 자동 적용
- **AttributeConverter**: 양방향 변환 로직을 구현하는 인터페이스
- 코드값을 명시적으로 관리하여 유지보수성과 안정성 확보
- Frontend(JSON) ↔ Backend(Enum) ↔ Database(Integer) 간 일관된 데이터 흐름 구축