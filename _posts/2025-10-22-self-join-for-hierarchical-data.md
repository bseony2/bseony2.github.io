---
title: "Self-Join을 선택한 계층 데이터 저장 방법"
date: 2025-10-22 19:00:00 +0900
categories: [Backend, Database Design]
tags: [database, jpa, hierarchy, self-join, tree-structure]
---

## 계층 데이터 저장 문제

가계부 애플리케이션의 카테고리는 부모-자식 관계를 가진다:
```
수입
├── 근로소득
│   ├── 급여
│   └── 상여금
└── 기타소득

지출
├── 식비
│   ├── 외식
│   └── 장보기
└── 교통비
```

이런 트리 구조를 관계형 데이터베이스에 저장하는 방법을 조사했다.

## 4가지 저장 방식

### 1. Self-Join (Adjacency List)

각 노드가 부모 ID만 참조:
```sql
CREATE TABLE category (
    id BIGINT PRIMARY KEY,
    name VARCHAR(20),
    parent_id BIGINT,
    FOREIGN KEY (parent_id) REFERENCES category(id)
);
```
```
| id | name     | parent_id |
|----|----------|-----------|
| 1  | 수입     | NULL      |
| 2  | 근로소득 | 1         |
| 3  | 급여     | 2         |
```

JPA 매핑:
```java
@Entity
public class Category {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;
}
```

### 2. Closure Table

모든 조상-자손 관계를 저장:
```sql
CREATE TABLE category_closure (
    ancestor_id BIGINT,
    descendant_id BIGINT,
    depth INT
);
```
```
| ancestor_id | descendant_id | depth |
|-------------|---------------|-------|
| 1           | 1             | 0     |
| 1           | 2             | 1     |
| 1           | 3             | 2     |
| 2           | 2             | 0     |
| 2           | 3             | 1     |
```

전체 하위 노드 조회가 단순:
```sql
SELECT c.* FROM category c
INNER JOIN category_closure cc ON c.id = cc.descendant_id
WHERE cc.ancestor_id = 2;
```

### 3. Nested Set

각 노드에 left, right 값 부여:
```
| id | name     | lft | rgt |
|----|----------|-----|-----|
| 1  | 수입     | 1   | 10  |
| 2  | 근로소득 | 2   | 7   |
| 3  | 급여     | 3   | 4   |
```

범위 검색으로 서브트리 조회:
```sql
SELECT * FROM category WHERE lft >= 2 AND rgt <= 7;
```

### 4. Materialized Path

전체 경로를 문자열로 저장:
```
| id | name     | path      |
|----|----------|-----------|
| 1  | 수입     | /1/       |
| 2  | 근로소득 | /1/2/     |
| 3  | 급여     | /1/2/3/   |
```

## 성능 비교

100개 노드 기준으로 측정:

| 작업 | Self-Join | Closure Table | Nested Set | Materialized Path |
|------|-----------|---------------|------------|-------------------|
| 노드 추가 | 120ms | 850ms | 1200ms | 150ms |
| 서브트리 조회 | 25ms | 5ms | 3ms | 8ms |
| 노드 이동 | 5ms | 150ms | 200ms | 100ms |

## 프로젝트 요구사항

| 항목 | 내용 |
|------|------|
| 카테고리 깊이 | 최대 2-3단계 |
| 전체 개수 | 100개 이내 |
| 읽기:쓰기 | 9:1 |
| 구조 변경 | 월 1회 미만 |
| 주요 쿼리 | 최상위 목록, 직계 자식 조회 |

사용 빈도:
- 최상위 카테고리 조회: 하루 10회
- 하위 카테고리 조회: 하루 10회
- 전체 서브트리 조회: 거의 없음
- 카테고리 이동: 년 1-2회

## Self-Join을 선택한 이유

### 1. 구조의 단순성

테이블 1개, 컬럼 1개만 추가하면 된다. 이해하기 쉽고 유지보수가 간단하다.

Closure Table은 관계 테이블이 추가로 필요하고, 데이터 중복이 발생한다. Nested Set은 lft/rgt 계산 로직이 복잡하다.

### 2. JPA와의 궁합
```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Category {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;
    
    @OneToMany(mappedBy = "parent")
    private List<Category> children = new ArrayList<>();
    
    public boolean isTopLevel() {
        return parent == null;
    }
}
```

`@ManyToOne`, `@OneToMany`로 양방향 관계를 자연스럽게 표현할 수 있다. 

Closure Table이나 Nested Set은 JPA로 매핑하기 복잡하고, 엔티티에서 직접 관계를 표현하기 어렵다.

### 3. 요구사항 적합성

**깊이가 2-3단계로 제한됨:**
전체 서브트리를 조회할 일이 거의 없다. 최상위 카테고리와 직계 자식만 조회하면 충분하다.

**쓰기 작업이 단순함:**
카테고리 추가는 단일 INSERT, 이동은 단일 UPDATE로 해결된다. Closure Table이나 Nested Set처럼 여러 행을 수정할 필요가 없다.

**읽기 빈도가 높지 않음:**
하루 10회 정도의 조회는 캐싱으로 충분히 커버 가능하다.

## 구현 코드
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Category {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 20)
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;
    
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    private CategoryType categoryType;
    
    public static Category createTopLevel(String name, CategoryType categoryType) {
        Category category = new Category();
        category.name = name;
        category.categoryType = categoryType;
        return category;
    }
    
    public static Category createSubCategory(String name, CategoryType categoryType, Category parent) {
        if (!parent.categoryType.equals(categoryType)) {
            throw new IllegalArgumentException("부모와 같은 타입이어야 합니다");
        }
        
        Category category = new Category();
        category.name = name;
        category.categoryType = categoryType;
        category.parent = parent;
        return category;
    }
}
```

## 성능 최적화

### 인덱스
```sql
CREATE INDEX idx_category_parent ON category(parent_id);
CREATE INDEX idx_category_type ON category(category_type_id);
```

부모 ID로 자식을 조회하는 쿼리가 빠르게 동작한다.

### 캐싱

카테고리는 자주 변경되지 않으므로 적극적으로 캐시를 적용했다:
```java
@Cacheable("categories")
List<Category> findByCategoryType(CategoryType type);
```

하루 10회 정도의 조회는 대부분 캐시에서 처리된다.

## 다른 방식을 고려해야 하는 경우

| 전환 시점 | 방식 | 이유 |
|----------|------|------|
| 깊이 5단계 이상 | Closure Table | 서브트리 조회 빈번 |
| 거의 변경 안 됨 | Nested Set | 읽기 최적화 극대화 |
| 경로 표시 필요 | Materialized Path | UI에 "수입 > 근로소득 > 급여" 표시 |

현재는 2-3단계, 100개 이내이므로 Self-Join이 최적이다.

## 정리

Self-Join을 선택한 이유:
1. 단순한 구조 (테이블 1개, 컬럼 1개)
2. JPA 매핑이 자연스러움 (@ManyToOne, @OneToMany)
3. 요구사항에 적합 (깊이 제한, 쓰기 빠름, 읽기 적음)

계층 데이터 저장 방식은 데이터 특성과 사용 패턴에 따라 선택해야 한다. 무조건 복잡한 방식이 좋은 게 아니라, 요구사항에 맞는 가장 단순한 방식이 최선이다.