---
layout: post
title: "JPA Auditing - CreatedDate와 LastModifiedDate로 엔티티 생성/수정 시간 자동 관리하기"
date: 2025-10-29 19:00:00 +0900
categories: [JPA, Spring Boot]
tags: [jpa, auditing, createddate, lastmodifieddate, spring-data-jpa]
---

## 들어가며

엔티티를 설계하다 보면 "이 데이터가 언제 생성되었는지", "마지막으로 언제 수정되었는지"를 추적해야 하는 경우가 많았다. 초기에는 매번 수동으로 `LocalDateTime.now()`를 호출해서 시간을 저장했는데, 이는 번거로울 뿐만 아니라 실수로 누락되기도 쉬웠다.

Spring Data JPA의 Auditing 기능을 사용하면 이러한 시간 정보를 자동으로 관리할 수 있다. 이번 글에서는 `@CreatedDate`와 `@LastModifiedDate` 애노테이션의 동작 원리와 실제 사용 방법을 정리했다.

---

## JPA Auditing이란?

JPA Auditing은 엔티티의 생성 시간, 수정 시간, 생성자, 수정자 등의 메타데이터를 자동으로 관리해주는 기능이다. Spring Data JPA에서 제공하며, 반복적인 코드를 줄이고 일관성 있는 데이터 관리를 가능하게 했다.

### 지원하는 애노테이션

- `@CreatedDate`: 엔티티 생성 시간
- `@LastModifiedDate`: 엔티티 수정 시간
- `@CreatedBy`: 엔티티 생성자
- `@LastModifiedBy`: 엔티티 수정자

이번 포스팅에서는 시간 관련 애노테이션인 `@CreatedDate`와 `@LastModifiedDate`에 집중했다.

---

## 기본 설정

### 1. Application 클래스에 Auditing 활성화

```java
@SpringBootApplication
@EnableJpaAuditing  // JPA Auditing 활성화
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@EnableJpaAuditing` 애노테이션을 추가하는 것만으로 Auditing 기능이 활성화되었다. 이 애노테이션은 Spring에게 "JPA Auditing을 사용하겠다"고 선언하는 역할을 한다.

### 2. 기본 엔티티 클래스 작성

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    // Getter만 제공 (Setter는 제공하지 않음)
    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public LocalDateTime getUpdatedAt() {
        return updatedAt;
    }
}
```

#### 주요 애노테이션 설명

**@MappedSuperclass**
- 이 클래스가 직접 테이블로 매핑되지 않고, 상속받는 엔티티에게 필드를 제공하는 부모 클래스임을 나타낸다
- BaseEntity 자체는 테이블이 생성되지 않고, 상속받은 자식 엔티티의 컬럼으로 포함된다

**@EntityListeners(AuditingEntityListener.class)**
- JPA의 라이프사이클 이벤트를 감지하는 리스너를 등록한다
- `AuditingEntityListener`는 Spring Data JPA에서 제공하는 리스너로, `@CreatedDate`와 `@LastModifiedDate` 필드를 자동으로 업데이트한다

**@CreatedDate**
- 엔티티가 처음 저장될 때(persist) 현재 시간이 자동으로 설정된다
- `updatable = false` 옵션으로 수정 불가능하게 했다

**@LastModifiedDate**
- 엔티티가 저장되거나 수정될 때마다 현재 시간으로 업데이트된다

---

## 실제 엔티티에 적용

```java
@Entity
public class Category extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;
    
    // 비즈니스 로직...
}
```

`Category` 엔티티가 `BaseEntity`를 상속받으면, 자동으로 `createdAt`과 `updatedAt` 필드가 포함되었다. 별도의 코드 없이도 생성 시간과 수정 시간이 관리되었다.

---

## 동작 원리 분석

### AuditingEntityListener 내부 동작

Spring Data JPA의 `AuditingEntityListener`는 JPA 라이프사이클 콜백을 사용해 동작한다:

```java
public class AuditingEntityListener {

    @PrePersist  // 엔티티가 영속화되기 전
    public void touchForCreate(Object target) {
        // @CreatedDate와 @LastModifiedDate 필드에 현재 시간 설정
    }

    @PreUpdate   // 엔티티가 수정되기 전
    public void touchForUpdate(Object target) {
        // @LastModifiedDate 필드만 현재 시간으로 업데이트
    }
}
```

#### 라이프사이클 이벤트

**@PrePersist**
- 엔티티가 처음 저장되기 직전에 호출된다
- `createdAt`과 `updatedAt` 모두 현재 시간으로 설정된다

**@PreUpdate**
- 엔티티가 수정되기 직전에 호출된다
- `updatedAt`만 현재 시간으로 갱신된다
- `createdAt`은 `updatable = false` 설정으로 변경되지 않는다

---

## 실제 사용 예시

### 카테고리 생성

```java
@Service
public class CategoryService {
    
    private final CategoryRepository categoryRepository;
    
    public CategoryResponse createCategory(CategoryRequest request) {
        Category category = Category.of(
            request.getName(),
            categoryType,
            parent,
            request.getDisplayOrder()
        );
        
        // save() 호출 시 자동으로 createdAt, updatedAt이 설정됨
        Category saved = categoryRepository.save(category);
        
        return CategoryResponse.from(saved);
    }
}
```

`save()` 메서드가 호출되면:
1. JPA가 `@PrePersist` 이벤트를 발생시킨다
2. `AuditingEntityListener`가 이벤트를 감지한다
3. 자동으로 `createdAt`과 `updatedAt`에 현재 시간이 설정된다

### 카테고리 수정

```java
public CategoryResponse updateCategory(Long id, CategoryRequest request) {
    Category category = categoryRepository.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("카테고리를 찾을 수 없습니다"));
    
    category.update(
        request.getName(),
        parent,
        request.getDisplayOrder()
    );
    
    // update() 호출 시 자동으로 updatedAt만 갱신됨
    Category updated = categoryRepository.save(category);
    
    return CategoryResponse.from(updated);
}
```

수정 시에는:
1. JPA가 `@PreUpdate` 이벤트를 발생시킨다
2. `AuditingEntityListener`가 `updatedAt`만 갱신한다
3. `createdAt`은 변경되지 않는다

---

## 데이터베이스 스키마

실제 테이블에는 다음과 같이 컬럼이 생성되었다:

```sql
CREATE TABLE category (
    id BIGINT PRIMARY KEY,
    name VARCHAR(20) NOT NULL,
    parent_id BIGINT,
    category_type_id BIGINT NOT NULL,
    display_order INT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_id) REFERENCES category(id),
    FOREIGN KEY (category_type_id) REFERENCES category_type(id)
);
```

`created_at`과 `updated_at` 컬럼이 자동으로 추가되고, JPA Auditing에 의해 관리되었다.

---

## 장점과 주의사항

### 장점

**1. 코드 간결화**
```java
// Before (수동 관리)
category.setCreatedAt(LocalDateTime.now());
category.setUpdatedAt(LocalDateTime.now());

// After (자동 관리)
// 아무 코드도 필요 없음!
```

**2. 일관성 보장**
- 모든 엔티티에서 동일한 방식으로 시간 정보가 관리되었다
- 개발자의 실수로 인한 누락이 방지되었다

**3. 유지보수 용이**
- 시간 관리 로직이 한 곳(BaseEntity)에 집중되어 수정이 쉬웠다

### 주의사항

**1. Auditing이 활성화되어야 한다**
```java
@EnableJpaAuditing  // 필수!
```

**2. EntityListener 등록 필요**
```java
@EntityListeners(AuditingEntityListener.class)  // 필수!
```

**3. 영속성 컨텍스트를 거쳐야 한다**
```java
// ✅ 동작함
categoryRepository.save(category);

// ❌ 동작하지 않음 (bulk update는 영속성 컨텍스트를 거치지 않음)
em.createQuery("UPDATE Category c SET c.name = :name")
  .setParameter("name", "새이름")
  .executeUpdate();
```

Bulk 연산은 데이터베이스에 직접 쿼리를 전송하므로 JPA 라이프사이클 이벤트가 발생하지 않는다. 이런 경우에는 수동으로 시간을 관리해야 했다.

---

## 실무 팁

### 1. BaseEntity 패턴 사용

모든 엔티티가 공통으로 사용하는 필드는 BaseEntity에 모아두었다:

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String updatedBy;
}
```

### 2. 시간대(TimeZone) 설정

```java
@Configuration
public class JpaConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return new AuditorAwareImpl();
    }
    
    @PostConstruct
    public void init() {
        TimeZone.setDefault(TimeZone.getTimeZone("Asia/Seoul"));
    }
}
```

서버의 기본 시간대를 명시적으로 설정하여 혼란을 방지했다.

### 3. 테스트 코드 작성

```java
@SpringBootTest
@Transactional
class CategoryAuditingTest {
    
    @Autowired
    private CategoryRepository categoryRepository;
    
    @Test
    void 카테고리_생성시_생성시간과_수정시간이_자동설정된다() {
        // given
        Category category = Category.of("테스트", categoryType, null, 1);
        
        // when
        Category saved = categoryRepository.save(category);
        
        // then
        assertNotNull(saved.getCreatedAt());
        assertNotNull(saved.getUpdatedAt());
        assertEquals(saved.getCreatedAt(), saved.getUpdatedAt());
    }
    
    @Test
    void 카테고리_수정시_수정시간만_갱신된다() throws InterruptedException {
        // given
        Category category = categoryRepository.save(
            Category.of("원래이름", categoryType, null, 1)
        );
        LocalDateTime originalCreatedAt = category.getCreatedAt();
        
        Thread.sleep(1000); // 1초 대기
        
        // when
        category.update("새이름", null, 1);
        Category updated = categoryRepository.save(category);
        
        // then
        assertEquals(originalCreatedAt, updated.getCreatedAt());
        assertTrue(updated.getUpdatedAt().isAfter(originalCreatedAt));
    }
}
```

---

## 마치며

JPA Auditing을 사용하면서 엔티티의 시간 정보를 관리하는 것이 훨씬 간편해졌다. `@EnableJpaAuditing` 한 줄과 `BaseEntity` 클래스 하나로 모든 엔티티에서 일관되게 생성/수정 시간을 추적할 수 있었다.

특히 `@CreatedDate`에 `updatable = false` 옵션을 추가하여 생성 시간이 실수로 변경되는 것을 방지한 점이 유용했다. 이러한 작은 설정 하나가 데이터 무결성을 지키는 데 큰 도움이 되었다.

다만 bulk 연산처럼 영속성 컨텍스트를 거치지 않는 작업에서는 동작하지 않는다는 점을 주의해야 한다. 이런 경우에는 수동으로 시간을 관리하거나, 데이터베이스 트리거를 사용하는 방법을 고려해볼 수 있다.

---

## 요약

- **JPA Auditing**은 엔티티의 생성/수정 시간을 자동으로 관리해주는 기능이다
- **@EnableJpaAuditing**으로 Auditing을 활성화하고, **@EntityListeners**로 리스너를 등록해야 한다
- **@CreatedDate**는 엔티티 생성 시 한 번만 설정되며, `updatable = false`로 보호할 수 있다
- **@LastModifiedDate**는 엔티티가 수정될 때마다 자동으로 갱신된다
- **AuditingEntityListener**가 @PrePersist와 @PreUpdate 이벤트를 감지하여 동작한다
- **BaseEntity 패턴**을 사용하면 모든 엔티티에서 일관되게 시간 정보를 관리할 수 있다
- **Bulk 연산**은 영속성 컨텍스트를 거치지 않으므로 Auditing이 동작하지 않는다