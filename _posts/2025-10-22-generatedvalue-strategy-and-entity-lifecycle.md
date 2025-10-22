---
title: "@GeneratedValue 전략과 Entity 생명주기"
date: 2025-10-22 20:30:00 +0900
categories: [Backend, JPA]
tags: [jpa, hibernate, entity-lifecycle, generationtype, generatedvalue]
---

## @GeneratedValue의 strategy 속성

JPA에서 엔티티의 ID를 자동 생성할 때 `@GeneratedValue` 어노테이션을 사용한다. 이 어노테이션의 `strategy` 속성에 따라 ID 생성 방식과 Entity의 생명주기가 달라진다는 걸 확인했다.
```java
@Entity
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.???)  // 이 선택이 중요
    private Long id;
}
```

## 4가지 전략 개요

### 1. IDENTITY
DB의 AUTO_INCREMENT 기능 사용
```java
@GeneratedValue(strategy = GenerationType.IDENTITY)
```

### 2. SEQUENCE
DB 시퀀스 객체 사용
```java
@GeneratedValue(strategy = GenerationType.SEQUENCE)
```

### 3. TABLE
별도 키 생성 테이블 사용
```java
@GeneratedValue(strategy = GenerationType.TABLE)
```

### 4. AUTO
JPA 구현체가 DB에 맞게 자동 선택

## 전략별 동작 방식 비교

카테고리 엔티티를 저장하는 코드로 각 전략을 테스트해봤다.
```java
Category category = Category.createTopLevel("급여", incomeType);
categoryRepository.save(category);  // 내부적으로 em.persist() 호출
```

### IDENTITY 전략
```java
@Entity
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

테스트 코드:
```java
@Test
void identityTest() {
    System.out.println("===== persist 호출 전 =====");
    
    Category category = Category.createTopLevel("급여", incomeType);
    categoryRepository.save(category);
    
    System.out.println("===== persist 호출 후 =====");
    System.out.println("ID: " + category.getId());
}
```

실행 결과:
```
===== persist 호출 전 =====
Hibernate: insert into category ...  // persist() 즉시 INSERT 실행
===== persist 호출 후 =====
ID: 1  // ID가 즉시 할당됨
```

**동작 원리:**
1. `em.persist()` 호출
2. 즉시 INSERT 실행
3. DB의 AUTO_INCREMENT로 생성된 ID를 받아옴
4. 엔티티의 ID 필드에 할당

**왜 즉시 INSERT가 실행될까?**

영속성 컨텍스트는 엔티티를 관리하기 위해 식별자(ID)가 반드시 필요하다. IDENTITY는 INSERT를 실행해야만 DB로부터 ID를 받을 수 있다.

### SEQUENCE 전략
```java
@Entity
@SequenceGenerator(
    name = "category_seq_generator",
    sequenceName = "category_seq",
    allocationSize = 1
)
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "category_seq_generator")
    private Long id;
}
```

실행 결과:
```
===== persist 호출 전 =====
Hibernate: select nextval('category_seq')  // 시퀀스에서 ID만 조회
===== persist 호출 후 =====
ID: 1  // ID는 할당됨
===== commit 시점 =====
Hibernate: insert into category ...  // INSERT는 나중에
```

**동작 원리:**
1. `em.persist()` 호출
2. DB 시퀀스에서 `SELECT nextval()` 실행
3. 받아온 ID를 엔티티에 할당
4. INSERT는 commit 시점까지 지연

### TABLE 전략
```java
@Entity
@TableGenerator(
    name = "category_table_generator",
    table = "id_generator",
    pkColumnValue = "category_id"
)
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "category_table_generator")
    private Long id;
}
```

별도 테이블에서 ID를 관리한다:
```sql
CREATE TABLE id_generator (
    gen_name VARCHAR(50) PRIMARY KEY,
    gen_value BIGINT
);
```

**동작 원리:**
1. `em.persist()` 호출
2. id_generator 테이블에서 다음 값 조회 및 업데이트
3. 받아온 ID를 엔티티에 할당
4. INSERT는 commit 시점까지 지연

## Entity 생명주기와의 관계

Entity는 다음 상태를 거친다:
```java
// 비영속 (Transient)
Category category = new Category();

// 영속 (Persistent) - persist() 호출 시점
em.persist(category);

// DB 동기화 - commit 또는 flush 시점
tx.commit();
```

각 전략에 따라 영속 상태 전환 시점이 다르다:

| 전략 | persist() 시점 | ID 할당 | INSERT 실행 | 영속 상태 |
|------|----------------|---------|-------------|-----------|
| IDENTITY | 즉시 INSERT | 즉시 | 즉시 | 가능 |
| SEQUENCE | 시퀀스 조회 | 즉시 | 지연(commit) | 가능 |
| TABLE | 테이블 조회 | 즉시 | 지연(commit) | 가능 |

## 영속성 컨텍스트의 1차 캐시

영속 상태의 엔티티는 1차 캐시의 이점을 누릴 수 있다:
```java
// SEQUENCE 전략
Category c1 = em.find(Category.class, 1L);  // DB 조회
Category c2 = em.find(Category.class, 1L);  // 캐시에서 가져옴 (쿼리 없음)
assertThat(c1 == c2).isTrue();  // 같은 인스턴스

// 변경 감지
c1.changeName("새이름");  // setter 호출만
tx.commit();  // 자동으로 UPDATE 쿼리
```

IDENTITY도 persist() 후에는 영속 상태가 되므로 동일한 이점을 누린다.

## 배치 INSERT와 성능 차이

### IDENTITY의 한계
```java
for (int i = 0; i < 100; i++) {
    Category category = Category.createTopLevel("카테고리" + i, incomeType);
    em.persist(category);  // 100번의 개별 INSERT
}
```

persist()마다 즉시 INSERT가 실행되므로 배치 최적화가 불가능하다.

### SEQUENCE의 장점
```java
for (int i = 0; i < 100; i++) {
    Category category = Category.createTopLevel("카테고리" + i, incomeType);
    em.persist(category);
}
// commit 시점에 배치 INSERT 가능 (설정에 따라)
```

`hibernate.jdbc.batch_size` 설정으로 배치 INSERT 최적화가 가능하다:
```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
```

## 성능 비교 테스트

100개 엔티티를 저장하는 시나리오로 각 전략을 측정했다:
```java
@Test
void performanceTest() {
    long start = System.currentTimeMillis();
    
    for (int i = 0; i < 100; i++) {
        Category category = Category.createTopLevel("카테고리" + i, incomeType);
        categoryRepository.save(category);
    }
    
    long elapsed = System.currentTimeMillis() - start;
    System.out.println("소요 시간: " + elapsed + "ms");
}
```

결과:
- IDENTITY: 250ms (100번의 개별 INSERT)
- SEQUENCE: 120ms (배치 INSERT 적용)
- TABLE: 350ms (테이블 락 경합)

## 상황별 전략 선택 가이드

### IDENTITY를 선택하는 경우

**장점:**
- 설정이 간단함
- 추가 DB 객체(시퀀스, 테이블) 불필요
- MySQL, PostgreSQL에서 자연스럽게 동작

**단점:**
- 쓰기 지연 불가능
- 배치 INSERT 최적화 불가능

**적합한 상황:**
```java
// 1. 소규모 데이터 (건당 저장)
@PostMapping("/category")
public void createCategory(@RequestBody CategoryRequest request) {
    Category category = Category.createTopLevel(request.getName(), categoryType);
    categoryRepository.save(category);  // 1건씩 저장
}

// 2. MySQL 사용
// MySQL은 AUTO_INCREMENT가 표준이므로 자연스러움
```

### SEQUENCE를 선택하는 경우

**장점:**
- 쓰기 지연 가능
- 배치 INSERT 최적화 가능
- 시퀀스 최적화 옵션 활용 가능

**단점:**
- 시퀀스 객체 필요 (MySQL 5.7 이하 미지원)
- 추가 SELECT 쿼리 발생

**적합한 상황:**
```java
// 1. 대량 데이터 처리
@Transactional
public void bulkInsert(List<CategoryRequest> requests) {
    for (CategoryRequest req : requests) {
        Category category = Category.createTopLevel(req.getName(), categoryType);
        categoryRepository.save(category);
    }
    // 배치 INSERT로 최적화됨
}

// 2. Oracle, PostgreSQL 사용
// 시퀀스를 기본 지원하는 DB
```

**시퀀스 최적화:**
```java
@SequenceGenerator(
    name = "category_seq_generator",
    sequenceName = "category_seq",
    allocationSize = 50  // 50개씩 미리 할당받아 메모리에서 관리
)
```

`allocationSize`를 설정하면 시퀀스 조회 횟수를 줄일 수 있다.

### TABLE을 선택하는 경우

**장점:**
- 모든 DB에서 동작
- 테이블만 있으면 됨

**단점:**
- 성능이 가장 낮음
- 테이블 락 경합 발생
- 별도 테이블 관리 필요

**적합한 상황:**
```java
// DB 독립적인 코드가 필요한 경우
// (실무에서는 거의 사용 안 함)
```

실무에서는 거의 사용하지 않는다. IDENTITY나 SEQUENCE를 권장한다.

### AUTO 전략
```java
@GeneratedValue  // strategy를 명시하지 않으면 AUTO
```

JPA 구현체(Hibernate)가 DB 방언(Dialect)에 따라 자동 선택:
- MySQL → IDENTITY
- Oracle → SEQUENCE
- H2 → SEQUENCE

**장점:**
- DB 변경 시 코드 수정 불필요

**단점:**
- 어떤 전략이 선택될지 명확하지 않음
- 성능 튜닝이 어려움

## 실전 권장 사항

### 1. MySQL 사용 시
```java
@Entity
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

- 대부분의 경우 IDENTITY로 충분
- 배치 INSERT가 빈번하다면 SEQUENCE 고려 (MySQL 8.0+)

### 2. PostgreSQL, Oracle 사용 시
```java
@Entity
@SequenceGenerator(
    name = "category_seq_generator",
    sequenceName = "category_seq",
    allocationSize = 50
)
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "category_seq_generator")
    private Long id;
}
```

- SEQUENCE 권장
- allocationSize로 최적화

### 3. 성능이 중요한 경우

배치 INSERT를 적극 활용:
```java
// application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

SEQUENCE 전략과 함께 사용하면 효과적이다.

## 정리

1. **IDENTITY**: 간단하지만 배치 최적화 불가, persist() 즉시 INSERT
2. **SEQUENCE**: 쓰기 지연 가능, 배치 최적화 가능, ID만 먼저 조회
3. **TABLE**: 범용적이지만 성능 낮음, 실무에서 거의 미사용
4. **전략 선택**: DB 종류와 데이터 처리 패턴에 따라 결정

@GeneratedValue의 전략 선택은 단순히 ID 생성 방식만 바꾸는 게 아니라, Entity의 영속화 시점과 성능에 직접적인 영향을 미친다는 걸 확인했다.