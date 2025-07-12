---
title : "커스텀 어노테이션과 이를 사용한 검증 방법"
date : 2025-07-12 11:30:00 +0900
categories : [Spring Boot]
tags : [java, spring boot, annotation, 어노테이션]
---

## 📌 개요

값에 대한 검증을 `@Valid` 어노테이션이나 `@Column` 어노테이션 등을 통해 구현할 수 있으나, 그 이상의 검증을 요구하는 경우 커스텀 어노테이션을 만들어 구현하는 것이 좋다.

요구사항은 다음과 같다.

1. `Player` 엔티티는 `name`, `role`, `information` 필드를 가진다.
2. `Role` 은 `ADMIN`, `USER` 가 존재한다.
3. `role` 이 `USER` 인 `player` 는 반드시 `information` 을 가져야 한다.

플레이어를 생성할 때, 생성된 객체의 `role` 이 `USER` 이면 `information` 을 가지고 있는지 확인하는 어노테이션을 작성해보자.

## 📌 필요한 어노테이션과 속성

그 전에, 커스텀 어노테이션을 만들기 위해 필요한 어노테이션과 속성을 살펴보자.

### @Target

`@Target` 은 커스텀 어노테이션이 사용될 위치를 결정하는 어노테이션이다. `ElementType` 이라는 `enum` 배열을 인자로 받는데, 종류는 다음과 같다.

| ElementType | 설명 |
| --- | --- |
| TYPE | 클래스, 인터페이스 |
| METHOD | 메서드 |
| FIELD | 필드 |
| PARAMETER | 메서드 파라미터 |
| CONSTRUCTOR | 생성자 |

```java
@Target(ElementType.TYPE)
public @interface UserInfoRequired {
    // ...
}
```

이렇게 작성하면 위 어노테이션을 클래스나 인터페이스에 적용할 수 있게 된다.

### @Retention

`@Retention` 은 어노테이션의 생명 주기를 결정하는 어노테이션이다. `RetentionPolicy` 라는 `eum` 배열을 인자로 받고, 종류는 다음과 같다.

| RetentionPolicy | 설명 |
| --- | --- |
| SOURCE | 소스 코드에만 존재하며, 컴파일 시 해당 어노테이션을 무시한다. |
| CLASS | 클래스 파일까지 존재한다. 즉, 런타임에 이 정보를 로드하지 않는다. |
| RUNTIME | 런타임까지 존재한다. |

보통 커스텀 어노테이션을 런타임에 특정 로직을 수행하도록 작성하므로 대부분의 경우 `RUNTIME` 을 사용한다.

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UserInfoRequired {
    // ...
}
```

이제 위 어노테이션은 런타임까지 정보가 유지된다.

### @Constraint

`@Constraint` 은 커스텀 어노테이션과 실제 검증 로직을 담고 있는 `Validator` 클래스를 연결하는 어노테이션이다.

우리는  `Role.USER` 이고, `information` 필드의 값이 `null` 이 아니라면 유효하다고 판단하는 validator를 만들어야 한다. 코드는 아래와 같다.

```java
public class UserInfoValidator implements ConstraintValidator<UserInfoRequired, PlayerCreateRequest> {
    @Override
    public boolean isValid(PlayerCreateRequest value, ConstraintValidatorContext context) {
        if (value.role() == Role.USER) {
            return value.information() != null && !value.information().isBlank();
        }
        return true;
    }
}
```

`ConstraintValidator` 인터페이스를 구현한다. `UserInfoRequired` 는 validator가 짝을 이루는 어노테이션, `PlayerCreateRequest` 은 검증할 수 있는 클래스를 의미한다. 다시 정리하면, `@UserInfoRequired` 어노테이션이 붙은 필드는 `UserInfoValidator` 가 동작하게 되며 해당 validator는 `PlayerCreateRequest` 타입을 검증한다.

`isValid` 는 `ConstraintValidator` 가 요구하는 메서드로, 실제로 검증 로직을 이 곳에 작성한다.

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UserInfoValidator.class)
public @interface UserInfoRequired{
    // ...
}
```

`validateBy` 옵션을 통해 validator를 어노테이션에 매핑시킨다.

### groups

`groups` 는 동일한 객체에 대해 상황에 따라 다른 검증 로직을 적용하고 싶을 때 사용한다. 보통 DTO를 재사용하는 경우가 잦으므로, DTO에서 이를 사용하는 경우가 많다.

예를 들어, 등록과 수정에 동시에 사용하는 DTO가 있다고 하자. 보통 `id` 필드를 가지고 있으며, 등록 시에는 `id` 는 `null` 이 되어야 하지만, 수정 시 `id` 는 존재해야 하는 것이 일반적이다. 이처럼 하나의 필드에 상반된 검증 로직을 적용해야 할 때 `groups` 를 사용한다.

```java
// 등록 시 사용할 인터페이스(마커)
public interface SaveCheck {
}

// 수정 시 사용할 인터페이스
public interface UpdateCheck {
}

```

먼저 검증 로직 그룹을 식별하기 위해 빈 인터페이스를 생성한다. ‘이름표’ 같은 역할을 한다.

```java
public class ItemDto {

    @NotNull(message = "수정 시에는 ID가 필수입니다.", groups = UpdateCheck.class)
    private Long id;

    @NotBlank(message = "상품 이름은 공백일 수 없습니다.", groups = {SaveCheck.class, UpdateCheck.class})
    private String itemName;
}

```

각 검증 로직이 어떤 그룹에 속하는지 `groups` 속성을 통해 명시한다.

```java
@RestController
@RequestMapping("/items")
public class ItemController {

    @PostMapping
    public ResponseEntity<> addItem(
            @Validated(SaveCheck.class) @RequestBody ItemDto itemDto) {
        return ResponseEntity.ok("상품 등록 성공");
    }

    @PutMapping("/{itemId}")
    public ResponseEntity<String> editItem(
            @PathVariable Long itemId,
            @Validated(UpdateCheck.class) @RequestBody ItemDto itemDto) {
        return ResponseEntity.ok("상품 수정 성공");
    }
}

```

`@Validated` 어노테이션을 통해 실행하고 싶은 마커 인터페이스를 지정한다. 해당 그룹에 속한 검증 로직만 수행된다. `@Valid` 를 사용하게 되면 모든 어노테이션을 검증하려고 시도하니 유의해야 한다.

### payload

보통 `message` 속성만으로 오류 메시지를 전달하지만, 단순한 경고인지 심각한 오류인지 구분하는 것처럼 그 이상의 동작을 구현하고 싶은 경우 `payload` 속성을 사용한다. `payload` 는 검증 로직에 추가적인 메타데이터를 포함시키기 위해 사용된다.

```java
public interface ErrorLevel extends Payload {
}
```

```java
public class ItemDto {

    @NotNull(message = "ID는 필수입니다.", payload = ErrorLevel.class)
    private Long id;
}
```

사용하는 방법은 `groups` 속성과 유사하다.

`payload` 에 담긴 정보는 검증 실패 시 생성되는 `ConstraintViolation` 객체를 통해 접근할 수 있다.