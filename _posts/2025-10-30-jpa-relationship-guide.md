---
layout: post
title: "JPA 연관관계 완벽 가이드 - @ManyToOne, @OneToOne, @OneToMany 깊이 이해하기"
date: 2025-10-29 22:00:00 +0900
categories: [Spring, JPA]
tags: [jpa, relationship, many-to-one, one-to-many, one-to-one, foreign-key]
---

## 들어가며

JPA의 연관관계 매핑을 깊이 있게 이해할 필요가 있었다. 예를 들어:

- 하나의 지출은 하나의 카테고리에 속한다 (Many-to-One)
- 하나의 카테고리는 여러 지출을 가질 수 있다 (One-to-Many)
- 카테고리는 부모-자식 관계를 가진다 (Self-referencing)

이 글에서는 실제 도메인 예시를 통해 `@ManyToOne`, `@OneToOne`, `@OneToMany`의 차이와 사용법을 정리했다.

---

## 1. 연관관계의 기본 개념

### 데이터베이스 관점

데이터베이스에서는 **외래 키(Foreign Key)**로 관계를 표현한다.
```sql
-- 카테고리 테이블
CREATE TABLE category (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50)
);

-- 지출 테이블
CREATE TABLE expense (
    id BIGINT PRIMARY KEY,
    amount DECIMAL(15, 2),
    category_id BIGINT,  -- 외래 키
    FOREIGN KEY (category_id) REFERENCES category(id)
);
```

### JPA 관점

JPA에서는 **객체 참조**로 관계를 표현한다.
```java
public class Expense {
    private Long id;
    private BigDecimal amount;
    private Category category;  // 객체 참조
}
```

JPA는 이 객체 참조를 데이터베이스의 외래 키와 매핑한다.

---

## 2. @ManyToOne: 다대일 관계

### 개념

"여러 개의 A가 하나의 B에 속한다"

**예시:**
- 여러 지출(Expense)이 하나의 카테고리(Category)에 속한다
- 여러 주문(Order)이 하나의 고객(Customer)에 속한다
- 여러 댓글(Comment)이 하나의 게시글(Post)에 속한다

### 데이터베이스 구조
```
Category (1)
    ↓
Expense (N)

category
  id  | name
  ----+------
  1   | 식비
  2   | 교통비

expense
  id  | amount  | category_id
  ----+---------+-------------
  1   | 5000    | 1
  2   | 3000    | 1
  3   | 2000    | 2
```

### 기본 매핑
```java
@Entity
public class Expense extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private BigDecimal amount;
    
    // Many Expenses : One Category
    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;
}
```

**핵심 포인트:**
- `@ManyToOne`은 **외래 키를 가진 쪽**에 선언
- `@JoinColumn(name = "category_id")`로 외래 키 컬럼명 지정
- 생략하면 `필드명_참조테이블PK`가 기본값 (category_id)

### fetch 속성
```java
@ManyToOne(fetch = FetchType.LAZY)  // 권장
@JoinColumn(name = "category_id")
private Category category;
```

| FetchType | 동작 | 사용 시점 |
|-----------|------|-----------|
| `LAZY` | 실제 사용 시점에 조회 | 대부분의 경우 (권장) |
| `EAGER` | 엔티티 조회 시 즉시 조회 | N+1 문제 주의 |

**LAZY 예시:**
```java
Expense expense = expenseRepository.findById(1L).get();
// SQL: SELECT * FROM expense WHERE id = 1

System.out.println(expense.getAmount());  // OK, 이미 조회됨

System.out.println(expense.getCategory().getName());  // 이 시점에 조회
// SQL: SELECT * FROM category WHERE id = ?
```

### optional 속성
```java
@ManyToOne(optional = false)  // NOT NULL
@JoinColumn(name = "category_id", nullable = false)
private Category category;
```

- `optional = true` (기본값): 외래 키가 null 허용
- `optional = false`: 외래 키가 NOT NULL (필수 관계)

---

## 3. @OneToMany: 일대다 관계

### 개념

"하나의 A가 여러 개의 B를 가진다"

**예시:**
- 하나의 카테고리(Category)가 여러 지출(Expense)을 가진다
- 하나의 게시글(Post)이 여러 댓글(Comment)을 가진다

### 양방향 연관관계
```java
@Entity
public class Category extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 50)
    private String name;
    
    // One Category : Many Expenses
    @OneToMany(mappedBy = "category")
    private List<Expense> expenses = new ArrayList<>();
}
```

**핵심 포인트:**
- `mappedBy`: 연관관계의 **주인을 지정**
- `"category"`는 Expense의 필드명
- 주인이 아닌 쪽은 읽기 전용

### 연관관계의 주인

JPA에서는 **외래 키를 관리하는 쪽이 연관관계의 주인**이다.
```java
// Expense (연관관계의 주인)
@ManyToOne
@JoinColumn(name = "category_id")  // 외래 키 관리
private Category category;

// Category (주인이 아님)
@OneToMany(mappedBy = "category")  // 읽기 전용
private List<Expense> expenses;
```

**규칙:**
- 주인만 외래 키를 등록/수정/삭제 가능
- 주인이 아닌 쪽은 조회만 가능
- **Many 쪽이 주인이 되는 것이 일반적** (외래 키가 있는 테이블)

### cascade 속성
```java
@OneToMany(mappedBy = "category", cascade = CascadeType.ALL)
private List<Expense> expenses = new ArrayList<>();
```

| CascadeType | 설명 | 예시 |
|-------------|------|------|
| `PERSIST` | 저장 시 연관된 엔티티도 저장 | 부모 저장 → 자식들도 저장 |
| `REMOVE` | 삭제 시 연관된 엔티티도 삭제 | 부모 삭제 → 자식들도 삭제 |
| `ALL` | 모든 연산 전파 | PERSIST + REMOVE + ... |

**예시:**
```java
Category category = new Category("식비");
category.getExpenses().add(new Expense(5000, category));
category.getExpenses().add(new Expense(3000, category));

categoryRepository.save(category);
// cascade = PERSIST이면 expenses도 자동 저장
```

### orphanRemoval 속성
```java
@OneToMany(mappedBy = "category", orphanRemoval = true)
private List<Expense> expenses = new ArrayList<>();
```

**동작:**
```java
Category category = categoryRepository.findById(1L).get();
category.getExpenses().remove(0);  // 컬렉션에서 제거

categoryRepository.save(category);
// orphanRemoval = true이면 해당 Expense 삭제
```

---

## 4. @OneToOne: 일대일 관계

### 개념

"하나의 A가 하나의 B에 대응된다"

**예시:**
- 하나의 사용자(User)가 하나의 프로필(Profile)을 가진다
- 하나의 주문(Order)이 하나의 배송정보(Delivery)를 가진다

### 주 테이블에 외래 키
```java
@Entity
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    
    @OneToOne
    @JoinColumn(name = "profile_id")  // User 테이블에 외래 키
    private Profile profile;
}

@Entity
public class Profile {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String bio;
    
    @OneToOne(mappedBy = "profile")
    private User user;
}
```

**데이터베이스:**
```sql
user
  id  | username | profile_id
  ----+----------+-----------
  1   | john     | 1

profile
  id  | bio
  ----+-----
  1   | Hello
```

### 대상 테이블에 외래 키
```java
@Entity
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    
    @OneToOne(mappedBy = "user")
    private Profile profile;
}

@Entity
public class Profile {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String bio;
    
    @OneToOne
    @JoinColumn(name = "user_id")  // Profile 테이블에 외래 키
    private User user;
}
```

**데이터베이스:**
```sql
user
  id  | username
  ----+----------
  1   | john

profile
  id  | bio   | user_id
  ----+-------+---------
  1   | Hello | 1
```

### 선택 기준

| 방식 | 장점 | 단점 |
|------|------|------|
| 주 테이블에 외래 키 | 객체지향적, 조회 성능 좋음 | 외래 키 null 허용 시 무결성 약화 |
| 대상 테이블에 외래 키 | 일대다 관계로 확장 용이 | 프록시 사용 제약, 지연 로딩 어려움 |

---

## 5. 실전 예시: 계층형 카테고리

### Self-referencing 관계
```java
@Entity
public class Category extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 50)
    private String name;
    
    // 부모 카테고리 (Many-to-One)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;
    
    // 자식 카테고리들 (One-to-Many)
    @OneToMany(mappedBy = "parent")
    private List<Category> children = new ArrayList<>();
    
    // 비즈니스 로직
    public boolean isTopLevel() {
        return parent == null;
    }
    
    public void addChild(Category child) {
        children.add(child);
        child.parent = this;
    }
}
```

**데이터베이스:**
```sql
category
  id  | name   | parent_id
  ----+--------+-----------
  1   | 식비   | NULL       (최상위)
  2   | 외식   | 1          (식비의 하위)
  3   | 장보기 | 1          (식비의 하위)
  4   | 교통비 | NULL       (최상위)
```

**사용 예시:**
```java
Category food = new Category("식비");
Category dining = new Category("외식");
Category grocery = new Category("장보기");

food.addChild(dining);
food.addChild(grocery);

categoryRepository.save(food);
// cascade 설정 시 dining, grocery도 함께 저장
```

---

## 6. 양방향 연관관계의 주의사항

### 연관관계 편의 메서드
```java
@Entity
public class Expense {
    
    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;
    
    // 연관관계 편의 메서드
    public void setCategory(Category category) {
        // 기존 관계 제거
        if (this.category != null) {
            this.category.getExpenses().remove(this);
        }
        
        // 새 관계 설정
        this.category = category;
        if (category != null) {
            category.getExpenses().add(this);
        }
    }
}
```

**사용:**
```java
Category category = new Category("식비");
Expense expense = new Expense(5000);

expense.setCategory(category);
// 양쪽 모두 설정됨
// expense.category = category
// category.expenses.contains(expense) = true
```

### 무한 루프 방지
```java
@Entity
public class Category {
    
    @OneToMany(mappedBy = "category")
    @JsonIgnore  // JSON 직렬화 시 무한 루프 방지
    private List<Expense> expenses = new ArrayList<>();
    
    @Override
    public String toString() {
        return "Category{" +
                "id=" + id +
                ", name='" + name + '\'' +
                // expenses는 제외 (무한 루프 방지)
                '}';
    }
}
```

---

## 7. N+1 문제와 해결

### 문제 상황
```java
List<Expense> expenses = expenseRepository.findAll();

for (Expense expense : expenses) {
    System.out.println(expense.getCategory().getName());
}

// SQL 실행 횟수: 1 + N
// SELECT * FROM expense;           (1번)
// SELECT * FROM category WHERE id = 1;  (N번)
// SELECT * FROM category WHERE id = 2;
// ...
```

### 해결 방법 1: Fetch Join
```java
@Query("SELECT e FROM Expense e JOIN FETCH e.category")
List<Expense> findAllWithCategory();

// SQL: SELECT e.*, c.* FROM expense e 
//      INNER JOIN category c ON e.category_id = c.id
// (1번의 쿼리로 모두 조회)
```

### 해결 방법 2: @EntityGraph
```java
@EntityGraph(attributePaths = {"category"})
List<Expense> findAll();
```

### 해결 방법 3: Batch Size
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "category_id")
@BatchSize(size = 100)  // 100개씩 IN 절로 조회
private Category category;
```
```sql
-- N+1이 아닌 1+1로 개선
SELECT * FROM expense;
SELECT * FROM category WHERE id IN (1, 2, 3, ..., 100);
```

---

## 8. 실무 권장사항

### 기본 전략

1. **지연 로딩 기본 사용**
```java
   @ManyToOne(fetch = FetchType.LAZY)
   @OneToOne(fetch = FetchType.LAZY)
```

2. **연관관계 주인은 Many 쪽**
```java
   // Expense가 주인 (외래 키 보유)
   @ManyToOne
   private Category category;
```

3. **양방향은 필요한 경우만**
   - 조회 성능이 중요한 경우
   - 비즈니스 로직상 필요한 경우

4. **Cascade는 신중히**
   - `REMOVE`는 특히 주의 (의도치 않은 삭제)
   - 부모-자식 관계에서만 사용

### 테스트 코드
```java
@Test
void 연관관계_테스트() {
    // given
    Category category = new Category("식비");
    categoryRepository.save(category);
    
    Expense expense = new Expense(5000);
    expense.setCategory(category);
    expenseRepository.save(expense);
    
    // when
    Expense found = expenseRepository.findById(expense.getId()).get();
    
    // then
    assertThat(found.getCategory().getName()).isEqualTo("식비");
}

@Test
void N플러스1_문제_해결_테스트() {
    // given
    Category category = new Category("식비");
    categoryRepository.save(category);
    
    for (int i = 0; i < 10; i++) {
        Expense expense = new Expense(1000 * i);
        expense.setCategory(category);
        expenseRepository.save(expense);
    }
    
    entityManager.clear();
    
    // when
    List<Expense> expenses = expenseRepository.findAllWithCategory();
    
    // then
    assertThat(expenses).hasSize(10);
    // 쿼리 1번만 실행됨을 확인 (로그 확인)
}
```

---

## 정리

- **@ManyToOne**: 외래 키를 가진 쪽에 선언, 다대일 관계 매핑
- **@OneToMany**: 컬렉션으로 일대다 관계 표현, mappedBy로 주인 지정
- **@OneToOne**: 일대일 관계, 주 테이블 또는 대상 테이블에 외래 키
- **연관관계의 주인**: 외래 키를 관리하는 쪽, Many 쪽이 주인
- **FetchType.LAZY**: 기본 전략으로 사용, 성능 최적화
- **Cascade**: 부모-자식 생명주기 관리, REMOVE는 신중히
- **N+1 문제**: Fetch Join, @EntityGraph, Batch Size로 해결
- 양방향 연관관계는 필요한 경우만 사용하고 편의 메서드로 일관성 유지