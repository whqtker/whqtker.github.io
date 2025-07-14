---
title : "정적 팩토리 메서드 패턴과 메서드명 규칙"
date : 2025-07-14 14:05:00 +0900
categories : [Design Pattern]
tags : [static factory method, 정적 팩토리 메서드, java, 자바]
---

## 📌 정적 팩토리 메서드 패턴

`Static Factory Method` (정적 팩토리 메서드) 패턴은 객체 생성의 역할을 **생성자 대신 정적 메서드에 위임**하는 디자인 패턴이다.

```java
User user1 = new User("John", 30);

User user2 = User.createWithNameAndAge("John", 30);
```

정적 팩토리 메서드란 단순히 객체 생성을 요청하는 메서드를 말한다. 생성자를 사용하면 바로 `new` 를 통해 객체를 직접 생성하는 반면, 정적 팩토리 메서드는 받은 파라미터를 통해 내부적으로 `new` 를 통해 객체를 생성하거나, 이미 존재하는 객체를 리턴한다.

**“생성자 대신 정적 팩토리 메서드를 통해 객체를 생성하라.”**

이펙티브 자바의 저자 조슈아 블로크가 가장 먼저 추천할 정도로 유용한 방법이다.

## 📌 특징

### 메서드명을 통해 객체 생성 목적을 명확히 드러낼 수 있다

생성자는 명확한 한계를 가지고 있다. 생성자의 이름은 반드시 클래스 이름과 같아야 하며, 여러 생성자를 만들려면 파라미터의 타입, 개수 등을 다르게 해야 한다. 그러나 정적 팩토리 메서드는 이름만으로 객체가 어떤 맥락으로 생성되는지 명확히 드러낼 수 있다.

```java
Order order1 = new Order(cart, user);

Order order2 = new Order(user, previousOrderId);
```

위 코드를 보고 어떤 종류의 주문이 생성되었는지 직관적으로 파악할 수 있는가? 직접 생성자의 내부 구현 코드를 봐야 정확한 동작을 알 수 있을 것이다.

```java
Order order1 = Order.fromCart(cart, user);

Order order2 = Order.basedOnPreviousOrder(user, previousOrderId);
```

정적 팩토리 메서드는 이름에 의미 있는 이름을 붙여 객체 생성 목적을 분명히 할 수 있다.

```java
public User(String email) { ... }
public User(String username) { ... } // 컴파일 에러

User userByEmail = User.fromEmail("test@example.com");
User userByUsername = User.fromUsername("testuser");

```

정적 팩토리 메서드는 생성자로는 불가능한 일도 할 수 있다. `User` 가 `email` 또는 `username` 으로 생성할 수 있다고 하자. 이를 생성자로 구현하면 시그니처가 겹치기 때문에 구현이 불가능하다. 이 문제를 정적 팩토리 메서드가 해결할 수 있다.

### 객체 생성에 대한 통제권을 가질 수 있다

생성자는 호출될 때마다 새로운 객체를 메모리에 할당하지만 정적 팩토리 메서드는 내부 로직을 통해 객체를 재사용하는 등 생성 방식을 조절할 수 있다.

자주 사용되는 불변 객체를 미리 만들고, 객체 생성 요청이 들어오면 새로 생성하는 것이 아닌 캐시에서 꺼내서 리턴하게 되면, 불필요한 객체 생성을 막을 수 있어 성능이 향상된다.

```java
Boolean b1 = new Boolean(true);
Boolean b2 = new Boolean(true);
System.out.println(b1 == b2);   // false

Boolean b3 = Boolean.valueOf(true);
Boolean b4 = Boolean.valueOf(true);
System.out.println(b3 == b4);   // true
```

정적 팩토리 메서드 패턴을 따르는 `Boolean` 클래스의 예를 보자. `valueOf` 메서드는 내부적으로 `Boolean.TRUE` 와 `Boolean.FALSE` 라는 `static final` 상수를 미리 만든다. 메서드가 호출되면 파라미터의 값에 따라 두 객체 중 하나를 리턴할 뿐, 새로운 메모리를 할당하지 않는다.

단 하나의 유일한 객체만 사용하도록 강제할 수 있다. 이러한 디자인 패턴을 `SingleTon Pattern` (싱글톤 패턴)이라고 하는데, 정적 팩토리 메서드는 이를 구현하는 가장 일반적인 방법이다.

```java
public class Printer {
    private static final Printer INSTANCE = new Printer();

    private Printer() {
        // private 생성자이므로 외부에서 호출할 수 없음
    }

    public static Printer getInstance() {
        return INSTANCE;
    }
}

Printer p1 = Printer.getInstance();
Printer p2 = Printer.getInstance();
System.out.println(p1 == p2); // true

```

`getInstance` 메서드는 클래스가 로드될 때 생성된 단 하나의 인스턴스만 리턴한다.

정적 팩토리 메서드로 `Flyweight Pattern` 또한 구현할 수 있는데, `Flyweight Pattern` 은 속성이 같은 여러 객체를 공유하여 메모리 사용량을 줄이는 패턴이다.

```java
public class FontFactory {
    private static final Map<String, Font> cache = new HashMap<>();

    public static Font getFont(String fontName, int size, boolean isBold) {
        String key = fontName + size + isBold;
        return cache.computeIfAbsent(key, k -> new Font(fontName, size, isBold));
    }
}

```

매 글자마다 `Font` 객체를 생성하면 수많은 객체가 생성되고, 메모리가 낭비된다. 대신, 폰트 설정을 키로 사용하여 이미 만들어진 `Font` 객체가 있다면 그것을 재사용하고, 없다면 새로 만들어 캐시에 저장한다. 이를 통해 불필요한 메모리 사용을 막을 수 있다.

### 하위 타입의 객체를 리턴할 수 있다

생성자는 항상 자신과 같은 타입의 객체만 생성할 수 있다. 그러나 정적 팩토리 메서드는 리턴 타입을 인터페이스로 지정하고, 해당 인터페이스를 구현하는 하위 타입 객체를 리턴하도록 할 수 있다. 이를 통해 불필요한 내부 구현 클래스를 숨길 수 있으며, 유연성이 증가한다.

```java
public interface PaymentMethod {

    void pay(int amount);

    public static PaymentMethod createCreditCardPayment() {
        return new CreditCard();
    }

    public static PaymentMethod createPayPalPayment() {
        return new PayPal();
    }
}

class CreditCard implements PaymentMethod {
    @Override
    public void pay(int amount) {
        System.out.println("신용카드로 " + amount + "원을 결제합니다.");
    }
}

class PayPal implements PaymentMethod {
    @Override
    public void pay(int amount) {
        System.out.println("페이팔로 " + amount + "원을 결제합니다.");
    }
}

```

외부에서 `createCreditCardPayment` 나 `createPayPalPayment` 메서드를 사용할 수는 있지만 `CreditCard` 과 `PayPal` 같은 구체적인 클래스는 알지 못한다.

### 파라미터에 따라 다른 클래스의 객체를 리턴할 수 있다

‘하위 타입의 객체를 리턴할 수 있다’의 확장 개념이다. 정적 팩토리 메서드가 분기점처럼 작동하여 파라미터에 따라 적합한 구현체를 선택하여 리턴한다.

```java
public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
    Enum<?>[] universe = getUniverse(elementType);
    if (universe == null)
        throw new ClassCastException(elementType + " not an enum");

    if (universe.length <= 64)
        return new RegularEnumSet<>(elementType, universe);
    else
        return new JumboEnumSet<>(elementType, universe);
}
```

대표적인 예시가 `EnumSet` 인데, `EnumSet` 은 추상 클래스로, 내부적으로 `RegularEnumSet`, `JumboEnumSet` 클래스를 가지고 있다. `RegularEnumSet` 은 전체 원소 개수가 64개 이하일 때, `JumboEnumSet` 은 전체 원소 개수가 65개 이상일 때 사용하는 구현체이다.

### 객체 생성을 캡슐화할 수 있다

정적 팩토리 메서드를 통해 객체를 생성하는 내부 구현 사항을 외부로부터 숨길 수 있다. 생성자는 특정 클래스의 객체를 만들겠다는 의도를 드러내지만, 정적 팩토리 메서드는 그렇지 않다.

상위 인터페이스 타입으로 객체를 리턴하게 되면 실제 구현과 독립적으로 정의된 메서드와 상호작용하면 된다. 또한 팩토리 클래스가 리턴하는 실제 구현 클래스를 `private` 로 선언하여 해당 클래스에 직접 접근하거나 `new` 로 생성하는 것을 원천적으로 막을 수 있다. 즉, 팩토리 메서드를 통해서만 객체를 얻도록 강제할 수 있다.

## 📌 메서드명

정적 팩토리 메서드의 이름을 짓는 컨벤션이 존재한다.

### from

```java
Date date = Date.from(instant);
```

`from` 메서드는 하나의 파라미터를 받아 해당 타입의 객체를 리턴한다.

### of

```java
Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
```

`of` 메서드는 여러 개의 파라미터를 받아 해당 타입의 객체를 리턴한다.

### valueOf

```java
BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
```

`valueOf` 는 `from` 이나 `of` 보다 자세하게 값을 변환하여 객체를 생성한다.

### instance / getInstance

```java
Calendar cal = Calendar.getInstance();
```

캐싱된 객체가 존재하거나 싱글톤 객체라면 해당 객체를 리턴하고, 그렇지 않다면 객체를 새로 생성하여 리턴한다. 즉, 리턴된 객체가 항상 동일한 객체임을 보장하지 않는다.

### create / newInstance

```java
Object newArray = Array.newInstance(classObject, arrayLen);
```

항상 새로운 객체를 생성하여 리턴한다.

### getXXX

```java
FileStore fs = Files.getFileStore(path);
```

`getInstance` 와 유사하지만 팩토리 메서드가 서로 다른 클래스에 있을 때 사용한다. 즉, 객체를 생성하는 주체와 생성되는 객체 타입이 다를 때 사용한다.

### newXXX

```java
BufferedReader br = Files.newBufferedReader(path);
```

`newInstance` 와 유사하지만 팩토리 메서드가 서로 다른 클래스에 있을 때 사용한다.

## 📌 참고

[https://mimah.tistory.com/entry/Effective-Java-정적-팩토리-메서드-Static-factory-method#google_vignette](https://mimah.tistory.com/entry/Effective-Java-%EC%A0%95%EC%A0%81-%ED%8C%A9%ED%86%A0%EB%A6%AC-%EB%A9%94%EC%84%9C%EB%93%9C-Static-factory-method#google_vignette)

[https://inpa.tistory.com/entry/GOF-💠-정적-팩토리-메서드-생성자-대신-사용하자#static_factory_method_pattern](https://inpa.tistory.com/entry/GOF-%F0%9F%92%A0-%EC%A0%95%EC%A0%81-%ED%8C%A9%ED%86%A0%EB%A6%AC-%EB%A9%94%EC%84%9C%EB%93%9C-%EC%83%9D%EC%84%B1%EC%9E%90-%EB%8C%80%EC%8B%A0-%EC%82%AC%EC%9A%A9%ED%95%98%EC%9E%90#static_factory_method_pattern)