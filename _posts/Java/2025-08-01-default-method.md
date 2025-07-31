---
title : "[Java] Default Method"
date : 2025-08-01 00:30:00 +0900
categories : [Java]
tags : [java, 자바, default method, interface]
---

## 📌 Default Method란?

```java
interface MyInterface {
    void normalMethod();

    default void printHello() {
        System.out.println("Hello World");
    }
}
```

`default` 메서드는 Java 8부터 도입된 기능으로, 기존에는 인터페이스에 추상 메서드만 작성할 수 있었으나 default 메서드의 등장으로 로직이 포함된 메서드를 작성할 수 있게 되었다.

왜 등장하였을까? 인터페이스를 구현한 N개의 클래스가 있다고 하자. 인터페이스의 새로운 추상 메서드가 추가되었고, 모든 클래스가 해당 메서드를 필수로 구현해야 한다고 하자. 그렇다면 N개의 클래스가 일일히 추상 메서드를 구현해야 하다. 이는 매우 불편하기에, 추상 메서드의 기본적인 구현을 제공하는 default 메서드가 등장한 것이다. 필요한 경우 오버라이딩하여 구현을 재정의할 수 있다.

## 📌 이점

- 하위 호환성이 좋다. 이전에 언급한 내용이지만, 기존에 구현된 클래스들을 변경하지 않고 새로운 메서드를 추가하는 것만으로 해당 메서드를 사용할 수 있게 한다.
- 인터페이스가 단순히 메서드를 선언하는 것을 넘어 기본 구현을 가질 수 있게 되면서 인터페이스의 역할이 유연해진다.

```java
interface Loggable {
    default void log(String message) {
        System.out.println("[LOG] " + message);
    }
}

interface Notifiable {
    default void notify(String event) {
        System.out.println("[NOTIFICATION] " + event);
    }
}

class ServiceProcessor implements Loggable, Notifiable {
    public void processData(String data) {
        log("데이터 처리 시작: " + data);
        
        System.out.println("데이터 " + data + " 처리 중...");
        
        notify("데이터 처리 완료: " + data);
    }
}
```

- 자바는 기본적으로 다중 상속을 지원하지 않는다. 대신 다중 상속과 유사한 효과를 얻기 위해 `implementes` 키워드를 통해 여러 인터페이스를 구현할 수 있다. 여기에 default 메서드 개념을 추가하면 하나의 클래스가 여러 인터페이스의 default 메서드가 제공하는 다양한 로직들을 조합하여 가질 수 있다.

## 📌 주의

클래스의 다중 상속에서 발생하는 `Diamond Problem` 이 동일하게 발생할 수 있다. 여러 인터페이스를 `implements` 한 상황에서 만약 동일한 이름의 (default) 메서드가 존재하는 경우 어떤 (default) 메서드를 취해야 할 지 모호해진다.

자바는 이 문제를 해결하기 위해 우선순위 규칙을 제공한다.

1. 구현하는 클래스 또는 상위 클래스가 동일한 시그니처의 메서드를 가지고 있다면 클래스에 정의된 메서드는 인터페이스의 default 메서드보다 항상 우선권을 가진다.
2. 인터페이스 간 상속 계층 구조가 있는 경우 가장 하위 인터페이스에 정의된 default 메서드가 우선권을 가진다.
3. 동일한 레벨의 여러 인터페이스에서 동일한 시그너치의 default 메서드가 충돌하는 경우 컴파일 에러가 발생한다. 인터페이스를 구현하는 클래스에서 충돌하는 default 메서드를 오버라이딩하여 사용해야 한다.

## 📌 default 메서드를 사용한 인터페이스 vs. 추상 클래스

한 가지 의문이 들어야 한다. default 메서드와 추상 클래스 모두 ‘추상화’를 제공하고, 구현을 부분적으로 포함할 수 있는데 어떤 차이가 있을까?

가장 큰 차이점은 클래스는 여러 인터페이스를 구현할 수 있으나, 추상 클래스는 단 하나만 상속할 수 있다는 점이다. 즉, 이전에 언급한 바와 같이 특정 동작을 다중 상속하는 효과를 가져올 수 있다.

## 📌 참고

https://whybk.tistory.com/155