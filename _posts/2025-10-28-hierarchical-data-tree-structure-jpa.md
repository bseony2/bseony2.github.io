---
layout: post
title: "JPA로 계층형 데이터 조회하기 - Self Join과 Stream API를 활용한 트리 구조 변환"
date: 2025-10-28 18:00:00 +0900
categories: [JPA, Java]
tags: [jpa, self-join, hierarchical-data, stream-api, tree-structure]
---

## 들어가며

가계부 애플리케이션을 개발하면서 카테고리 데이터를 계층 구조로 설계했다. "수입" 아래에 "급여", "상여", "투자수익" 같은 하위 카테고리가 있고, "지출" 아래에는 "식비", "주거비", "교통비" 등이 있는 구조였다.

데이터베이스에는 평면(flat) 구조로 저장되어 있지만, 클라이언트에서는 트리(tree) 구조로 보여주어야 했다. 처음에는 재귀 쿼리나 복잡한 로직이 필요할 것 같았는데, JPA와 Java Stream API를 조합하니 생각보다 간단하게 해결할 수 있었다.

이번 글에서는 Self Join을 활용한 계층형 데이터 모델링부터, 평면 데이터를 트리 구조로 변환하는 방법까지 정리했다.

---

## 계층형 데이터 모델링

### Self Join 패턴

계층 구조를 표현하는 가장 일반적인 방법은 Self Join이다. 같은 테이블 내에서 부모-자식 관계를 표현했다.

```java
@Entity
public class Category {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;  // 부모 카테고리 (Self Join)
    
    @OneToMany(mappedBy = "parent")
    private List<Category> children = new ArrayList<>();  // 자식 카테고리들
    
    // 최상위 카테고리인지 확인
    public boolean isTopLevel() {
        return parent == null;
    }
}
```

#### 주요 포인트

**@ManyToOne으로 부모 참조**
- 각 카테고리는 하나의 부모를 가질 수 있다 (Many-to-One)
- `parent_id` 컬럼이 외래키로 생성된다
- `FetchType.LAZY`로 지연 로딩을 설정하여 성능을 최적화했다

**@OneToMany로 자식 참조**
- 양방향 관계를 설정하여 부모에서 자식 목록에 접근할 수 있다
- `mappedBy`로 외래키의 주인이 `parent` 필드임을 명시했다

**isTopLevel() 메서드**
- 최상위 카테고리(부모가 없는 카테고리)를 판별하는 헬퍼 메서드다
- 트리 구조를 만들 때 시작점을 찾는 데 사용했다

---

## 데이터베이스 구조

실제로 생성된 테이블은 다음과 같았다:

```sql
CREATE TABLE category (
    id BIGINT PRIMARY KEY,
    name VARCHAR(20) NOT NULL,
    parent_id BIGINT,  -- Self Join을 위한 외래키
    category_type_id BIGINT NOT NULL,
    display_order INT,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES category(id) ON DELETE CASCADE
);
```

### 샘플 데이터

```sql
-- 최상위 카테고리 (parent_id = NULL)
INSERT INTO category (id, name, parent_id, category_type_id, display_order) 
VALUES (1, '수입', NULL, 1, 1);

INSERT INTO category (id, name, parent_id, category_type_id, display_order) 
VALUES (3, '식비', NULL, 1, 3);

-- 하위 카테고리 (parent_id = 1, 수입의 자식)
INSERT INTO category (id, name, parent_id, category_type_id, display_order) 
VALUES (100, '급여', 1, 1, 1);

INSERT INTO category (id, name, parent_id, category_type_id, display_order) 
VALUES (101, '상여', 1, 1, 2);

-- 하위 카테고리 (parent_id = 3, 식비의 자식)
INSERT INTO category (id, name, parent_id, category_type_id, display_order) 
VALUES (300, '외식', 3, 1, 1);

INSERT INTO category (id, name, parent_id, category_type_id, display_order) 
VALUES (301, '장보기', 3, 1, 2);
```

이렇게 저장된 평면 데이터를 트리 구조로 변환해야 했다.

---

## Repository 계층

### 전체 데이터 조회

```java
public interface CategoryRepository extends JpaRepository<Category, Long> {
    
    // 특정 타입의 모든 카테고리 조회
    List<Category> findByCategoryTypeIn(List<CategoryType> types);
}
```

일단 모든 카테고리를 평면 리스트로 가져왔다. 여기서 중요한 점은 **한 번의 쿼리로 모든 데이터를 가져온다**는 것이다. N+1 문제를 방지하기 위해 재귀 쿼리를 사용하지 않았다.

---

## Service 계층 - 트리 구조 변환 로직

핵심은 평면 리스트를 트리 구조로 변환하는 것이었다. 이를 위해 Java Stream API를 활용했다.

### 전체 코드

```java
@Service
public class CategoryService {
    
    public List<CategoryResponse> getCategoryTree(List<String> typeNames) {
        // 1. 타입 이름으로 CategoryType 엔티티 조회
        List<CategoryType> types = typeNames.stream()
                .map(name -> categoryTypeRepository.findByName(name)
                        .orElseThrow(() -> new IllegalArgumentException(
                            "존재하지 않는 카테고리 타입: " + name)))
                .toList();
        
        // 2. 해당 타입들의 모든 카테고리를 한 번에 조회 (평면 리스트)
        List<Category> allCategories = categoryRepository.findByCategoryTypeIn(types);
        
        // 3. 타입별로 그룹핑
        Map<CategoryType, List<Category>> categoryByType = allCategories.stream()
                .collect(Collectors.groupingBy(Category::getCategoryType));
        
        // 4. 각 타입별로 트리 구조 생성
        return categoryByType.entrySet().stream()
                .flatMap(entry -> {
                    List<Category> categories = entry.getValue();
                    return categories.stream()
                            .filter(Category::isTopLevel)  // 최상위 카테고리만 필터링
                            .map(category -> CategoryResponse.fromWithChildren(
                                category, categories));  // 트리 구조로 변환
                })
                .toList();
    }
}
```

### 단계별 분석

#### 1단계: 타입 조회

```java
List<CategoryType> types = typeNames.stream()
        .map(name -> categoryTypeRepository.findByName(name)
                .orElseThrow(() -> new IllegalArgumentException(
                    "존재하지 않는 카테고리 타입: " + name)))
        .toList();
```

클라이언트가 요청한 타입 이름(예: "가계부")을 실제 엔티티로 변환했다. 존재하지 않는 타입이면 예외가 발생한다.

#### 2단계: 전체 카테고리 조회

```java
List<Category> allCategories = categoryRepository.findByCategoryTypeIn(types);
```

**한 번의 쿼리**로 필요한 모든 카테고리를 가져왔다. 이후 작업은 모두 메모리에서 처리되므로 추가 쿼리가 발생하지 않는다.

실행되는 SQL:
```sql
SELECT * FROM category 
WHERE category_type_id IN (1, 2, 3);
```

#### 3단계: 타입별 그룹핑

```java
Map<CategoryType, List<Category>> categoryByType = allCategories.stream()
        .collect(Collectors.groupingBy(Category::getCategoryType));
```

같은 타입의 카테고리들을 모았다. 예를 들어:
```
{
  CategoryType(id=1, name="가계부"): [
    Category(id=1, name="수입", parent=null),
    Category(id=100, name="급여", parent=Category(1)),
    Category(id=3, name="식비", parent=null),
    Category(id=300, name="외식", parent=Category(3)),
    ...
  ]
}
```

#### 4단계: 트리 구조 생성

```java
return categoryByType.entrySet().stream()
        .flatMap(entry -> {
            List<Category> categories = entry.getValue();
            return categories.stream()
                    .filter(Category::isTopLevel)  // ← 핵심: 최상위만 선택
                    .map(category -> CategoryResponse.fromWithChildren(
                        category, categories));
        })
        .toList();
```

**핵심 아이디어**: 최상위 카테고리(부모가 없는 카테고리)만 필터링한 후, 각각에 대해 자식들을 재귀적으로 찾아 붙인다.

- `filter(Category::isTopLevel)`: `parent == null`인 카테고리만 선택
- `CategoryResponse.fromWithChildren()`: 해당 카테고리와 전체 리스트를 받아서 트리 구조로 변환

---

## DTO 계층 - 트리 변환 로직의 핵심

### CategoryResponse 구조

```java
public record CategoryResponse(
        Long id,
        String name,
        Long parentId,
        Integer displayOrder,
        List<CategoryResponse> children  // 재귀적 구조
) {
    // 단일 카테고리 변환 (자식 없음)
    public static CategoryResponse from(Category category) {
        return new CategoryResponse(
                category.getId(),
                category.getName(),
                category.getParent() != null ? category.getParent().getId() : null,
                category.getDisplayOrder(),
                List.of()  // 빈 리스트
        );
    }
    
    // 트리 구조로 변환 (자식 포함)
    public static CategoryResponse fromWithChildren(
            Category category, 
            List<Category> allCategories) {
        
        // 현재 카테고리의 자식들을 찾아서 변환
        List<CategoryResponse> children = allCategories.stream()
                .filter(c -> c.getParent() != null && 
                           c.getParent().getId().equals(category.getId()))
                .map(CategoryResponse::from)  // 자식은 단순 변환
                .toList();
        
        return new CategoryResponse(
                category.getId(),
                category.getName(),
                category.getParent() != null ? category.getParent().getId() : null,
                category.displayOrder(),
                children  // 찾은 자식들 포함
        );
    }
}
```

### fromWithChildren() 메서드 상세 분석

```java
public static CategoryResponse fromWithChildren(
        Category category,      // 부모 카테고리
        List<Category> allCategories) {  // 전체 카테고리 리스트
    
    // 1. 전체 리스트에서 이 카테고리의 자식들만 필터링
    List<CategoryResponse> children = allCategories.stream()
            .filter(c -> 
                c.getParent() != null &&  // 부모가 있고
                c.getParent().getId().equals(category.getId())  // 부모가 현재 카테고리
            )
            .map(CategoryResponse::from)  // DTO로 변환
            .toList();
    
    // 2. 부모 카테고리 + 찾은 자식들로 응답 생성
    return new CategoryResponse(
            category.getId(),
            category.getName(),
            category.getParent() != null ? category.getParent().getId() : null,
            category.getDisplayOrder(),
            children
    );
}
```

#### 동작 과정 예시

전체 카테고리 리스트:
```
[
  Category(id=1, name="수입", parent=null),
  Category(id=100, name="급여", parent=Category(1)),
  Category(id=101, name="상여", parent=Category(1)),
  Category(id=3, name="식비", parent=null),
  Category(id=300, name="외식", parent=Category(3))
]
```

`fromWithChildren(Category(id=1, name="수입"), allCategories)` 호출 시:

1. **필터링**: `parent.id == 1`인 카테고리들 찾기
   ```
   Category(id=100, name="급여", parent=Category(1))
   Category(id=101, name="상여", parent=Category(1))
   ```

2. **변환**: 찾은 카테고리들을 `CategoryResponse`로 변환

3. **결과**:
   ```json
   {
     "id": 1,
     "name": "수입",
     "parentId": null,
     "displayOrder": 1,
     "children": [
       {
         "id": 100,
         "name": "급여",
         "parentId": 1,
         "displayOrder": 1,
         "children": []
       },
       {
         "id": 101,
         "name": "상여",
         "parentId": 1,
         "displayOrder": 2,
         "children": []
       }
     ]
   }
   ```

---

## 전체 흐름 정리

### 1. 평면 데이터 조회

```java
List<Category> allCategories = categoryRepository.findByCategoryTypeIn(types);
```

**DB에서 가져온 평면 리스트**:
```
[
  {id: 1, name: "수입", parent: null},
  {id: 100, name: "급여", parent: 1},
  {id: 101, name: "상여", parent: 1},
  {id: 3, name: "식비", parent: null},
  {id: 300, name: "외식", parent: 3},
  {id: 301, name: "장보기", parent: 3}
]
```

### 2. 타입별 그룹핑

```java
Map<CategoryType, List<Category>> categoryByType = 
    allCategories.stream()
        .collect(Collectors.groupingBy(Category::getCategoryType));
```

### 3. 최상위 카테고리 필터링

```java
categories.stream()
    .filter(Category::isTopLevel)
```

**필터링 결과**:
```
[
  {id: 1, name: "수입", parent: null},
  {id: 3, name: "식비", parent: null}
]
```

### 4. 각 최상위 카테고리에 자식 붙이기

```java
.map(category -> CategoryResponse.fromWithChildren(category, categories))
```

**최종 트리 구조**:
```json
[
  {
    "id": 1,
    "name": "수입",
    "children": [
      {"id": 100, "name": "급여", "children": []},
      {"id": 101, "name": "상여", "children": []}
    ]
  },
  {
    "id": 3,
    "name": "식비",
    "children": [
      {"id": 300, "name": "외식", "children": []},
      {"id": 301, "name": "장보기", "children": []}
    ]
  }
]
```

---

## 성능 최적화 포인트

### 1. 한 번의 쿼리로 모든 데이터 조회

```java
// ✅ Good: 1번의 쿼리
List<Category> allCategories = categoryRepository.findByCategoryTypeIn(types);

// ❌ Bad: N+1 문제 발생
for (Category parent : parents) {
    List<Category> children = categoryRepository.findByParent(parent);
}
```

전체 카테고리를 한 번에 조회한 후 메모리에서 필터링했다. 재귀 쿼리를 사용하지 않아 데이터베이스 부하가 줄었다.

### 2. Stream API를 활용한 메모리 내 처리

```java
allCategories.stream()
    .filter(c -> c.getParent() != null && 
               c.getParent().getId().equals(category.getId()))
    .map(CategoryResponse::from)
    .toList();
```

데이터베이스가 아닌 Java 메모리에서 필터링과 변환을 수행했다. 카테고리 수가 수천 개 수준이면 메모리 처리가 훨씬 빠르다.

### 3. Lazy Loading 설정

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "parent_id")
private Category parent;
```

부모 카테고리를 지연 로딩으로 설정하여 불필요한 조인을 방지했다. 필요할 때만 `parent.getId()`를 호출한다.

---

## 한계와 대안

### 현재 방식의 한계

**1. 3단계 이상 깊은 계층에서 비효율**
```
대분류 (depth 1)
└── 중분류 (depth 2)
    └── 소분류 (depth 3)
        └── 세분류 (depth 4)
```

현재 코드는 2단계까지만 효율적이다. 더 깊은 계층에서는 재귀 로직이 필요하다.

**2. 대용량 데이터에서 메모리 사용**

수만 개의 카테고리를 한 번에 메모리에 올리면 부담이 될 수 있다.

### 대안 1: 재귀 함수 사용

```java
public static CategoryResponse fromWithChildrenRecursive(
        Category category, 
        List<Category> allCategories) {
    
    List<CategoryResponse> children = allCategories.stream()
            .filter(c -> c.getParent() != null && 
                       c.getParent().getId().equals(category.getId()))
            .map(child -> fromWithChildrenRecursive(child, allCategories))  // 재귀 호출
            .toList();
    
    return new CategoryResponse(
            category.getId(),
            category.getName(),
            category.getParent() != null ? category.getParent().getId() : null,
            category.getDisplayOrder(),
            children
    );
}
```

재귀 함수를 사용하면 N단계 계층도 처리할 수 있다.

### 대안 2: 데이터베이스 재귀 쿼리 (CTE)

```sql
-- PostgreSQL의 Recursive CTE
WITH RECURSIVE category_tree AS (
    -- 최상위 카테고리
    SELECT id, name, parent_id, 1 as depth
    FROM category
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- 재귀: 자식 카테고리들
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM category c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

데이터베이스에서 직접 계층 구조를 만들 수 있다. 하지만 JPA로 매핑하기 복잡하고, 네이티브 쿼리를 사용해야 한다.

### 대안 3: Materialized Path 패턴

```java
@Entity
public class Category {
    
    @Id
    private Long id;
    
    private String name;
    
    private String path;  // 예: "/1/100/1001"
}
```

경로를 문자열로 저장하여 조회를 단순화할 수 있다. 하지만 업데이트 비용이 크다.

---

## 실무 적용 시 고려사항

### 1. 캐싱 전략

카테고리는 자주 변경되지 않으므로 캐싱이 효과적이다:

```java
@Service
public class CategoryService {
    
    @Cacheable(value = "categories", key = "#typeNames")
    public List<CategoryResponse> getCategoryTree(List<String> typeNames) {
        // ...
    }
    
    @CacheEvict(value = "categories", allEntries = true)
    public CategoryResponse createCategory(CategoryRequest request) {
        // 캐시 무효화
    }
}
```

### 2. 페이지네이션 불가

트리 구조는 페이지네이션과 맞지 않는다. 전체를 한 번에 보여주거나, 클라이언트에서 필요한 부분만 펼치는 방식을 사용했다.

### 3. 정렬 순서

```java
@Entity
public class Category {
    private Integer displayOrder;  // 순서 보장
}
```

형제 카테고리 간의 순서를 보장하기 위해 `displayOrder` 필드를 추가했다.

---

## 테스트 코드

```java
@SpringBootTest
@Transactional
class CategoryServiceTest {
    
    @Autowired
    private CategoryService categoryService;
    
    @Test
    void 계층형_카테고리_트리_조회() {
        // given
        CategoryType type = new CategoryType(1L, "가계부");
        
        Category income = Category.of("수입", type, null, 1);
        Category salary = Category.of("급여", type, income, 1);
        Category bonus = Category.of("상여", type, income, 2);
        
        categoryRepository.saveAll(List.of(income, salary, bonus));
        
        // when
        List<CategoryResponse> tree = categoryService.getCategoryTree(
            List.of("가계부")
        );
        
        // then
        assertThat(tree).hasSize(1);
        CategoryResponse root = tree.get(0);
        assertThat(root.name()).isEqualTo("수입");
        assertThat(root.children()).hasSize(2);
        assertThat(root.children())
            .extracting("name")
            .containsExactly("급여", "상여");
    }
}
```

---

## 마치며

계층형 데이터를 다루는 것은 처음에는 복잡해 보였지만, Self Join과 Stream API를 조합하니 깔끔하게 해결되었다. 특히 "모든 데이터를 한 번에 조회한 후 메모리에서 가공한다"는 접근 방식이 N+1 문제를 예방하면서도 성능을 유지하는 핵심이었다.

`fromWithChildren()` 메서드의 핵심은 **전체 리스트를 순회하며 `parent.id`가 일치하는 자식들을 필터링**하는 것이다. 이 간단한 로직으로 평면 데이터가 트리 구조로 변환되었다.

다만 3단계 이상의 깊은 계층이나 수만 개의 대용량 데이터에서는 재귀 함수나 데이터베이스 CTE 같은 다른 방식을 고려해야 한다. 프로젝트의 요구사항과 데이터 규모에 맞는 방법을 선택하는 것이 중요하다.

---

## 요약

- **Self Join**으로 같은 테이블 내에서 부모-자식 관계를 표현했다
- **한 번의 쿼리**로 모든 카테고리를 조회하여 N+1 문제를 방지했다
- **Stream API의 filter**로 최상위 카테고리(`parent == null`)만 선택했다
- **fromWithChildren()** 메서드가 전체 리스트에서 자식들을 찾아 트리 구조로 변환했다
- `filter(c -> c.getParent().getId().equals(category.getId()))`로 특정 부모의 자식들만 필터링했다
- **Collectors.groupingBy**로 타입별 그룹핑을 수행했다
- 메모리 내 처리로 데이터베이스 부하를 줄였다
- 2단계 계층까지는 효율적이지만, 더 깊은 계층에서는 재귀 로직이 필요하다
- 카테고리는 자주 변경되지 않으므로 **캐싱**이 효과적이다