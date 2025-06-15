---
title : "Spring Boot의 Bean, 생명 주기, Bean Scope"
date : 2025-04-07 19:00:00 +0900
categories : [Spring Boot]
tags : [java, spring boot, bean, 스프링 부트, 빈]
---

## 📌 Bean이란?

`Bean` 은 스프링 프레임워크에서 `IoC(Inversion of Control)` 컨테이너에 의해 관리되는 객체를 말한다.

> Spring Boot의 특징 중 하나는`IoC(Inversion of Control)`이다. IoC는 객체의 생성 및 제어권, 생명주기 관리 등을 개발자가 아닌 Spring Boot Framework에게 맡기는 것이다. 이를 통해 개발자는 new를 통해 생성한 객체를 사용하는 것이 아닌 Spring Boot에 의해 관리되는 bean을 사용하면 되는 것이다.
> 

IoC 컨테이너는 bean으로 등록할 클래스들을 자동으로 탐색 및 관리하는데, 이를 `Component Scan` 이라고 한다.

bean을 사용함으로써 얻는 장점은 다음과 같다.

1. 애플리케이션의 비즈니스 로직을 캡슐화할 수 있다.
2. `Dependency Injection, DI (의존성 주입)` 을 통해 객체 간 결합도를 낮추고 재사용성을 높일 수 있다.

> `Dependency Injection` 은 객체 간 의존성을 관리하는 방법이다. 의존성이란 클래스가 다른 클래스나 객체를 사용하는 것이다.
> 
1. 애플리케이션을 모듈화할 수 있으며 유지보수성을 향상시킬 수 있다.

## 📌 Bean의 구성 요소

- `ApplicationContext` 는 IoC 컨테이너의 인터페이스이다. Spring Boot 애플리케이션을 실행하면 자동으로 생성된다. bean에 대한 전반적인 관리와 의존성 주입을 수행한다.

- `Bean` 은 IoC 컨테이너에 의해 생성 및 관리되는 객체이다. 애플리케이션의 핵심 로직을 캡슐화한다.

- `Dependency Injection(DI)` 는 클래스 간 의존성을 외부에서 주입받는 방법이다. IoC 컨테이너가 bean 간의 의존성을 자동으로 연결한다.

### Bean을 정의하는 방법

1. XML

```xml
<bean id="myBean" class="com.example.MyClass">
    <property name="propertyName" value="propertyValue"/>
</bean>
```

`<bean>` 태그를 통해 클래스와 속성을 명시적으로 지정할 수 있다. 자주 사용되는 방법은 아니다.

1. Java 기반

```java
@Configuration
public class AppConfig {
    @Bean
    public MyBean myBean() {
        return new MyBean();
    }
}

```

`@Configuration`, `@Bean` 어노테이션을 통해 Java 코드로 bean을 정의할 수 있다.

1. 어노테이션 기반

```java
@Component
public class MyService {
    // ...
}
```

`@Component`, `@Service`, `@Repository`, `@Controller` 등의 어노테이션을 통해 자동으로 bean을 등록할 수 있다.

참고로 @Service, @Repository, @Controller와 같은 어노테이션은 @Component의 상속을 받고 있다.

### DI를 구현하는 방법

1. 생성자 주입

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

객체의 의존 관계가 한 번 주입되면 다시 변경되지 않음을 보장하며, 테스트 시 Mock 객체를 쉽게 주입할 수 있다.

1. 필드 주입

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}
```

의존성이 명확하게 드러나지 않으며 테스트 시 Mock 객체를 주입하기 어럽다. 권장하지 않는 방법이다.

1. Setter 주입

```java
@Service
public class UserService {
    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

선택적인 의존성이나 런타임에 변경 가능해야 하는 의존성에 적합한 방법이다.

## 📌 Bean의 생명 주기

bean의 생명 주기 단계는 다음과 같다.

### 객체 생성(Instantiation)

애플리케이션에 등록된 bean을 IoC 컨테이너가 탐색한다. 생성자는 의존성 주입과 함께 호출된다. bean인 기본적으로 singleton 스코프에서 한 번만 생성된다.

### 의존성 설정(Dependency Injection)

IoC 컨테이너가 필요한 의존성을 주입하여 bean을 구성한다.

### 초기화(Initialization)

bean이 초기화된 후 추가적인 작업을 수행할 수 있는 단계이다.

### 사용(Ready to Use)

bean이 초기화되었다면 애플리케이션에서 사용할 수 있다.

### 소멸(Destruction)

IoC 컨테이너가 종료될 때 bean의 리소스를 정리하고 소멸한다.

## 📌 Bean의 스코프

bean의 스코프에 따라 IoC 컨테이너가 bean을 관리하는 방식이 달라진다. 

### Singleton Scope

컨테이너 당 하나의 인스턴스만 생성되며, 모든 요청에서 동일한 객체를 리턴한다. 또한 애플리케이션 전체에서 같은 state를 공유한다.

애플리케이션이 시작되면 생성되며 컨테이너가 종료되면 소멸된다.

### Prototype Scope

매 요청마다 새로운 인스턴스를 생성한다. state를 유지해야 하는 객체에 적합하다. 주의해야 할 점은, Singleton bean에 Prototype bean을 주입하면 Prototype bean이 Singleton bean처럼 동작한다.

bean 요청이 들어오면 생성되며 애플리케이션이 아니라 GC에 의해 소멸된다.

### Request Scope

HTTP 요청마다 새로운 인스턴스를 생성한다. 동일한 요청 내에서만 state를 공유한다.

HTTP 요청이 시작되면 생성되며, 요청이 종료되면 소멸된다.

### Session Scope

사용자 세션 당 하나의 인스턴스를 생성한다. 로그인한 사용자별로 state를 관리한다.

세션이 생성되면 생성되며 세션이 만료되면 소멸한다.

### Application Scope

`ServiceContext` 당 하나의 인스턴스가 생성되며, 애플리케이션 전역에서 공유된다.

애플리케이션이 시작되면 생성되며 종료 시 소멸된다.

### WebSocket Scope

`WebSocket` 세션별로 인스턴스가 생성된다.

WebSocket 연결 시 생성되며 연결이 종료되면 소멸된다.