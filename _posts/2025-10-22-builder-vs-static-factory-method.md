---
title: "Builder 패턴과 정적 팩토리 메서드 비교"
date: 2025-10-22 23:30:00 +0900
categories: [Backend, DDD]
tags: [ddd, domain-driven-design, factory-method, builder-pattern]
---

## 객체 생성 방식 선택

카테고리 엔티티 설계 중 객체 생성 방식을 선택해야 했다. Lombok의 `@Builder`와 정적 팩토리 메서드 중 어느 것이 더 적합한지 비교해보기로 했다.

## Builder 패턴으로 시작

초기 설계는 Builder 패턴을 사용했다.
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;           // 필수
    private CategoryType categoryType;  // 필수
    private Category parent;       // 선택
    private Integer displayOrder;  // 선택
}
```

사용 예시:
```java
Category category = Category.builder()
    .name("급여")
    .categoryType(incomeType)
    .displayOrder(1)
    .build();
```

## 필수 필드 누락 문제 발견

Builder 패턴으로 필수 필드를 빠뜨리고 객체를 생성해봤다.
```java
@Test
void builder_필수필드_누락() {
    CategoryType incomeType = CategoryType.of("수입");
    
    // categoryType을 의도적으로 누락
    Category category = Category.builder()
        .name("급여")
        .build();
    
    categoryRepository.save(category);
}
```

결과:
```
jakarta.validation.ConstraintViolationException: 
Validation failed for object='category'. Error count: 1
```

컴파일은 성공했지만 런타임에 에러가 발생했다. IDE도 필수 필드 누락을 경고하지 않았다.

## 정적 팩토리 메서드로 변경

다른 방식을 시도해봤다.
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private CategoryType categoryType;
    private Category parent;
    private Integer displayOrder;
    
    // 필수 파라미터를 메서드 시그니처로 강제
    public static Category createTopLevel(String name, CategoryType categoryType) {
        validateName(name);
        validateCategoryType(categoryType);
        
        Category category = new Category();
        category.name = name;
        category.categoryType = categoryType;
        return category;
    }
    
    public static Category createSubCategory(String name, CategoryType categoryType, Category parent) {
        validateName(name);
        validateCategoryType(categoryType);
        validateParent(parent);
        
        Category category = new Category();
        category.name = name;
        category.categoryType = categoryType;
        category.parent = parent;
        return category;
    }
    
    // 선택적 파라미터는 메서드 체이닝
    public Category withDisplayOrder(Integer displayOrder) {
        this.displayOrder = displayOrder;
        return this;
    }
}
```

필수 파라미터를 빠뜨리면 어떻게 되는지 확인했다:
```java
// 파라미터를 빠뜨리면
Category category = Category.createTopLevel("급여");  // 컴파일 에러
// Required type: CategoryType
// Provided: <no argument>
```

IDE에서 즉시 에러를 표시했다. 컴파일 단계에서 필수 파라미터 누락을 감지할 수 있다.

## 비즈니스 규칙 검증

하위 카테고리는 부모와 같은 CategoryType을 가져야 한다는 요구사항이 있었다.

Builder 패턴에서는 이런 검증이 어렵다:
```java
// 빌더로는 파라미터 간의 관계를 검증하기 어려움
Category category = Category.builder()
    .name("외식")
    .categoryType(expenseType)  // 지출
    .parent(incomeParent)       // 부모는 수입 (논리 오류)
    .build();
```

정적 팩토리 메서드에서는 생성 시점에 검증할 수 있다:
```java
public static Category createSubCategory(String name, CategoryType categoryType, Category parent) {
    validateName(name);
    validateCategoryType(categoryType);
    validateParent(parent);
    
    // 비즈니스 규칙 검증
    if (!parent.categoryType.equals(categoryType)) {
        throw new IllegalArgumentException(
            "하위 카테고리는 부모와 같은 타입이어야 합니다"
        );
    }
    
    Category category = new Category();
    category.name = name;
    category.categoryType = categoryType;
    category.parent = parent;
    return category;
}
```

테스트로 검증:
```java
@Test
void 하위카테고리_타입불일치_검증() {
    CategoryType incomeType = CategoryType.of("수입");
    CategoryType expenseType = CategoryType.of("지출");
    
    Category incomeParent = Category.createTopLevel("근로소득", incomeType);
    
    assertThatThrownBy(() -> 
        Category.createSubCategory("식비", expenseType, incomeParent))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("하위 카테고리는 부모와 같은 타입이어야 합니다");
}
```

## 도메인 언어 표현

Builder 패턴:
```java
Category category = Category.builder()
    .name("급여")
    .categoryType(incomeType)
    .build();
```

기술적 관점: "빌더로 필드를 설정하고 빌드한다"

정적 팩토리 메서드:
```java
Category topLevel = Category.createTopLevel("급여", incomeType);
Category subCategory = Category.createSubCategory("외식", expenseType, parent);
```

도메인 관점: "최상위 카테고리를 생성한다", "하위 카테고리를 생성한다"

메서드명이 도메인 용어를 직접 표현한다.

## 다양한 생성 전략

여러 생성 컨텍스트가 있을 때 정적 팩토리 메서드가 유리하다.
```java
public class Order {
    // 온라인 주문
    public static Order createOnlineOrder(Customer customer, List<OrderItem> items) {
        validateCustomer(customer);
        validateItems(items);
        
        Order order = new Order();
        order.orderType = OrderType.ONLINE;
        order.customer = customer;
        order.items = items;
        order.deliveryRequired = true;
        order.createdAt = LocalDateTime.now();
        return order;
    }
    
    // 매장 주문
    public static Order createStoreOrder(List<OrderItem> items, Store store) {
        validateItems(items);
        validateStore(store);
        
        Order order = new Order();
        order.orderType = OrderType.STORE;
        order.items = items;
        order.store = store;
        order.deliveryRequired = false;
        order.createdAt = LocalDateTime.now();
        return order;
    }
}
```

각 생성 컨텍스트마다 다른 기본값과 검증 로직을 적용할 수 있다.

## Step Builder 패턴 검토

완벽한 컴파일 타임 안전성을 위해 Step Builder 패턴도 검토했다.
```java
public interface NameStep {
    CategoryTypeStep name(String name);
}

public interface CategoryTypeStep {
    BuildStep categoryType(CategoryType categoryType);
}

public interface BuildStep {
    BuildStep parent(Category parent);
    BuildStep displayOrder(Integer displayOrder);
    Category build();
}
```

Step Builder는 완벽한 컴파일 타임 안전성을 제공하지만, 구현 복잡도가 매우 높다. 필드가 추가되면 인터페이스를 수정해야 하고 유지보수 비용이 크다. 프로젝트 규모를 고려했을 때 과도한 엔지니어링으로 판단했다.

## 비교 결과

| 기준 | Builder | 정적 팩토리 | Step Builder |
|------|---------|------------|--------------|
| 컴파일 타임 안전성 | 없음 | 있음 | 있음 |
| 도메인 언어 표현 | 제한적 | 우수 | 제한적 |
| 비즈니스 규칙 강제 | 어려움 | 쉬움 | 쉬움 |
| 구현 복잡도 | 낮음 | 보통 | 매우 높음 |
| 유지보수성 | 좋음 | 좋음 | 나쁨 |

## 선택 기준

Builder 패턴이 적합한 경우:
- DTO, VO 등 단순 데이터 전달 객체
- 필수 필드가 없거나 매우 적음
- 복잡한 비즈니스 규칙이 없음

정적 팩토리 메서드가 적합한 경우:
- 도메인 엔티티
- 필수 필드가 있음
- 생성 시점의 비즈니스 규칙 검증 필요
- 여러 생성 전략 존재
- 도메인 언어 표현 중요

## 최종 선택

본 프로젝트의 카테고리 엔티티는 정적 팩토리 메서드를 선택했다. 이유는 다음과 같다:
- 필수 필드 존재 (name, categoryType)
- 부모-자식 타입 일치 검증 필요
- 최상위/하위 카테고리 생성 컨텍스트 구분
- 컴파일 타임 안전성 확보

이를 통해 컴파일 타임 안전성과 도메인 무결성을 모두 확보할 수 있었다.