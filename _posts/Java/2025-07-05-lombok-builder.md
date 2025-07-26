---
title : "Builder 어노테이션"
date : 2025-07-05 14:50:00 +0900
categories : [Java]
tags : [java, spring boot, lombok, builder]
---

## 📌 @Builder 란?

`Lombok` 의 `@Builder` 는 빌더 패턴을 자동으로 생성하는 어노테이션이다. 

```java
@Getter
@Builder
public class User {
    private String name;
    private String email;
    private int age;
    private String address;
}
```

```java
User user = User.builder()
    .name("김철수")
    .email("kim@example.com")
    .age(30)
    .address("서울시 강남구")
    .build();
```

`@Builder` 를 클래스에 붙이면 Lombok이 컴파일 시점에 빌더 클래스와 메서드를 생성한다.

```java
private User(String name, String email, int age, String address) {
    this.name = name;
    this.email = email;
    this.age = age;
    this.address = address;
}

public static UserBuilder builder() {
    return new UserBuilder();
}

public static class UserBuilder {
    private String name;
    private String email;
    private int age;
    private String address;

    public UserBuilder name(String name) {
        this.name = name;
        return this;
    }

    public UserBuilder email(String email) {
        this.email = email;
        return this;
    }

    public UserBuilder age(int age) {
        this.age = age;
        return this;
    }

    public UserBuilder address(String address) {
        this.address = address;
        return this;
    }

    public User build() {
        return new User(name, email, age, address);
    }
}
```

Lombok이 자동으로 생성하는 코드는 세 가지이다.

1. 전체 필드를 인자로 받는 `private` 생성자
2. 빌더 객체를 리턴하는 메서드
3. 내부 빌더 클래스

필드의 모든 `setter` 메서드는 `return this;` 를 포함하기에 메서드 체이닝이 가능하다.

## 📌 장점

1. 생성자로 객체를 생성하는 방법보다 가독성이 좋다.
2. 메서드 체이닝을 통해 필요한 필드만 선택하여 객체를 유연하게 생성할 수 있다.
3. 매개변수 순서를 신경쓰지 않아도 된다.
4. `setter` 메서드 없이 불변 객체를 생성할 수 있다.

## 📌 단점

1. 필수로 필요한 필드를 누락할 가능성이 있다.
2. 객체 생성에 여러 메서드를 호출하므로 성능 상 오버헤드가 발생한다.

## 📌 @Builder 관련 옵션

### @Builder.Default

`@Builder.Default` 는 `@Builder` 를 사용할 때 특정 필드에 기본값을 설정하기 위한 옵션이다.

```java
@Builder
public class User {
    private String name;
    private int age = 20;
}

User user = User.builder()
    .name("홍길동")
    .build();
```

빌더는 필드의 초기화 값을 무시하고, 설정되지 않는 필드에는 `null` 을 할당한다. 이러한 문제를 해결하기 위한 방법은 초기 값이 설정된 필드 위에 `@Builder.Default` 를 명시하는 것이다. 이를 통해 빌더가 설정되지 않아도 기본 값을 제대로 할당할 수 있다.

### toBuilder

`toBuilder` 옵션은 기존 객체의 필드 값을 복사하여 새로운 객체를 생성할 때 사용한다. 불변 객체의 일부 값만 변경해야 할 때 유용하다.

`toBuilder` 옵션이 활성화되면 Lombok은 컴파일 시점에 `toBuilder` 메서드를 생성한다. 이 메서드는 기존 객체의 필드 값으로 초기화된 빌더를 리턴한다.

```java
Order firstOrder= Order.builder()
    .orderNumber("12345")
    .productName("노트북")
    .totalPrice(1500000L)
    .build();

Order secondOrder= firstOrder.toBuilder()
    .totalPrice(1450000L)
    .build();
```

단, `toBuilder` 메서드는 기존 객체를 수정하는 것이 아닌 새로운 객체를 생성하는 것이므로 이를 `save` 하게 되면 의도와 다른 동작이 발생할 수 있다. 위 예제에서 `secondOrder` 는 `firstOrder` 와 별개의 인스턴스이다. `save(secondOrder)` 를 호출하면 JPA는 `INSERT` 쿼리를 실행하여 새로운 레코드가 추가된다.

JPA 엔티티를 올바르게 수정하기 위한 방법은 `toBuilder` 메서드를 사용하는 것이 아니라 영속 상태 엔티티를 `setter` 나 메서드를 통해 값을 변경해야 한다.

### @SuperBuilder

`@SuperBuilder` 는 상속 관계에 있는 클래스에서 빌더를 올바르게 사용할 수 있도록 하는 어노테이션이다. 

```java
@Builder
public class Parent {
    private String parentName;
}

@Builder
public class Child extends Parent {
    private String childName;
}
```

상속 관계에 있는 클래스에 `@Builder` 를 사용하면 컴파일 에러가 발생하게 된다. 자식 클래스는 부모 클래스의 정적 멤버를 상속받으며, 따라서 자식 클래스에는 자식의 `builder` 메서드와 부모의 `builder` 메서드가 동시에 존재하게 된다. 자바는 정적 메서드의 오버라이딩을 허용하지 않으며, 자식의 메서드가 부모의 메서드를 숨김 처리한다. 그러나 `@Builder` 는 각 클래스에 대해 서로 관련 없는 내부 빌더 클래스를 생성한다. 따라서 자식 클래스는 이름은 같으너 서로 호환되지 않는 리턴 타입을 가진 두 개의 `builder` 메서드를 가지게 되어 컴파일 에러가 발생한다.

`@SuperBuilder` 를 사용하면 자식 클래스의 빌더에서 부모 클래스 필드까지 설정할 수 있게 된다. 그 이유는 `@Builder` 와 달리 상속 관계를 지원하기 위해 제네릭을 사용한 빌더 클래스 계층을 사용하기 때문이다. 즉, 내부적으로 자식 빌더가 부모 빌더를 상속받는 구조를 만든다.