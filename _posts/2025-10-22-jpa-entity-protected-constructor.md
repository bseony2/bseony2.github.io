---
title: "JPA Entity의 기본 생성자와 접근 제한자"
date: 2025-10-22 21:30:00 +0900
categories: [Backend, JPA]
tags: [jpa, hibernate, entity, lombok, proxy]
---

## 기본 생성자의 필요성

JPA Entity를 작성할 때 관례적으로 사용하는 패턴이 있다.
```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Category {
    // ...
}
```

기본 생성자가 필요한 이유와 protected를 사용하는 이유를 확인해보려고 한다.

## 기본 생성자 없이 만들면

먼저 기본 생성자 없이 엔티티를 만들어봤다.
```java
@Entity
public class TestEntity {
    @Id
    private Long id;
    private String name;
    
    // 기본 생성자 없음
    public TestEntity(String name) {
        this.name = name;
    }
}
```

실행 결과:
```
org.hibernate.InstantiationException: No default constructor for entity: TestEntity
```

기본 생성자가 없으면 엔티티 생성에 실패한다.

## JPA의 객체 생성 방식

JPA는 DB에서 데이터를 가져와서 객체로 만들 때 리플렉션(Reflection)을 사용한다.
```java
// JPA가 내부적으로 하는 일
Class<?> clazz = Class.forName("com.example.Member");
Constructor<?> constructor = clazz.getDeclaredConstructor();
Object instance = constructor.newInstance();

// 그 다음 필드에 값 주입
Field nameField = clazz.getDeclaredField("name");
nameField.setAccessible(true);
nameField.set(instance, "홍길동");
```

일단 빈 객체를 만들고, 필드에 값을 채워넣는 방식이다. 그래서 기본 생성자가 필요하다.

## 프록시 객체와의 관계

지연 로딩(Lazy Loading) 설정을 사용하면 JPA는 프록시 객체를 생성한다.
```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    private Member member;
}
```

이 경우 `member`는 실제 Member 객체가 아니라 Member를 상속받은 프록시 클래스의 인스턴스다.
```java
// Hibernate가 생성하는 프록시 (개념적 표현)
public class Member$HibernateProxy extends Member {
    private Member target;
    
    public Member$HibernateProxy() {
        super();  // 부모의 기본 생성자 호출 필요
    }
    
    @Override
    public String getName() {
        if (target == null) {
            target = loadFromDB();
        }
        return target.getName();
    }
}
```

프록시는 원본 클래스를 상속받는다. 자식 클래스 생성자에서 부모 클래스의 생성자를 호출해야 하는데, private 생성자는 자식 클래스에서 접근할 수 없다.

## 접근 제한자별 테스트

각 접근 제한자로 테스트해봤다.

### private 생성자
```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class Member {
    // ...
}
```

프록시 생성 시 에러 발생:
```
org.hibernate.InstantiationException: could not instantiate proxy
```

### public vs protected
```java
// public
@NoArgsConstructor(access = AccessLevel.PUBLIC)

// protected  
@NoArgsConstructor(access = AccessLevel.PROTECTED)
```

두 경우 모두 JPA 동작에는 문제가 없었다. 그렇다면 protected를 선택하는 이유는 무엇인지 확인해봤다.

## 도메인 무결성 관점

public 생성자가 있으면 외부에서 무분별하게 객체를 생성할 수 있다.
```java
// public 생성자가 있다면
Member member = new Member();  // 유효하지 않은 상태의 객체 생성 가능
member.setName(null);  // 비즈니스 규칙 위반

// protected 생성자 + 정적 팩토리 메서드
Member member = Member.create("홍길동");  // 유효성 검증 포함
```

protected는 같은 패키지 내에서만 접근 가능하므로, 외부에서는 정적 팩토리 메서드를 통해서만 객체를 생성하도록 강제할 수 있다.
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Category {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    // 정적 팩토리 메서드에서 유효성 검증
    public static Category createTopLevel(String name, CategoryType categoryType) {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("카테고리 이름은 필수");
        }
        
        Category category = new Category();
        category.name = name;
        category.categoryType = categoryType;
        return category;
    }
}

// 다른 패키지에서
Category category = new Category();  // 컴파일 에러
Category category = Category.createTopLevel("급여", incomeType);  // OK
```

## 테스트로 확인
```java
@Test
void 정적_팩토리_메서드로만_생성() {
    CategoryType incomeType = CategoryType.of("수입");
    
    Category category = Category.createTopLevel("급여", incomeType);
    
    assertThat(category.getName()).isEqualTo("급여");
    assertThat(category.getCategoryType()).isEqualTo(incomeType);
}

@Test
void 유효하지_않은_이름_검증() {
    CategoryType incomeType = CategoryType.of("수입");
    
    assertThatThrownBy(() -> Category.createTopLevel(null, incomeType))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("카테고리 이름은 필수");
}
```

## 접근 제한자별 비교

| 접근 제한자 | JPA 엔티티 생성 | 프록시 생성 | 외부 무분별 생성 차단 | DDD 적합성 |
|------------|----------------|------------|---------------------|-----------|
| private | 실패 | 실패 | 가능 | 불가능 |
| protected | 성공 | 성공 | 가능 (같은 패키지 제외) | 적합 |
| public | 성공 | 성공 | 불가능 | 제한적 |

## 정리

1. JPA는 리플렉션으로 엔티티를 생성하므로 기본 생성자가 필요하다
2. 프록시 생성을 위해 private은 사용할 수 없다
3. protected를 사용하면 외부 무분별 생성을 막고 정적 팩토리 메서드로 생성을 강제할 수 있다

`@NoArgsConstructor(access = AccessLevel.PROTECTED)`는 JPA의 기술적 요구사항과 도메인 무결성을 모두 만족시킨다.