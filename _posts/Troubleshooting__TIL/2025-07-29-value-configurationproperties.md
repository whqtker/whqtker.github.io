---
title : "Value와 ConfigurationProperties, 어느 어노테이션을 사용하여 프로퍼티를 주입할 것인가"
date : 2025-07-29 09:55:00 +0900
categories : [Troubleshooting/TIL]
tags : [spring boot, value, configurationProperties, 프로퍼티, 스프링 부트]
---

![image.png](assets/img/til/4.png)

## 📌 회고

여태까지 내가 프로퍼티를 주입하기 위해 알고 있던 방법은 `@Value` 어노테이션을 사용하는 것이었다. 여느 개발자가 그렇듯이 더 나은 방법을 알지 못하면 기존의 방식을 계속 사용하게 되기 마련이다. 그리고 난 최근에서야 ‘더 우아한 대안’의 존재에 대해 알게 되었다.

일단 `@Value` 어노테이션과 `@ConfigurationProperties` 어노테이션에 대해 알아보고, `@ConfigurationProperties` 이 어떤 장점을 가지고 있는지 탐구하고자 한다.

## 📌 @Value

`@Value` 어노테이션은 `Spring Expression Language(SpEL)` 을 사용하여 설정 파일의 특정 키에 해당하는 값을 필드 또는 메서드 파라미터에 직접 주입하는 방법이다.

```yaml
app:
  name: My App
  storage:
    upload-dir: /data/uploads
    max-file-size: 10MB
```

예를 들어 `application.yml` 에 위와 같은 설정이 있다고 가정하자.

```java
@Value("${app.storage.upload-dir}")
private String uploadDir;
```

`@Value` 어노테이션을 사용하여 설정 파일의 키 경로를 사용하여 값을 `uploadDir` 에 주입할 수 있다.

```java
@Value("${app.aws.s3.bucket-name:default-bucket}")
private String bucketName;
```

만약 특정 키가 존재하지 않는 경우 ‘:’를 사용하여 기본 값을 지정할 수도 있다.

SpEL을 잘 사용하면 다른 표현식도 작성할 수 있으나, 여기서는 더 살펴보지 않으려고 한다.

간단한 단일 설정 값을 가져올 때 직관적이고 편하다. 이건 부정할 수 없는 사실이다.

다만 주입한 필드에 `final` 키워드를 붙일 수 없다.

왜 그럴까? `final` 로 선언된 필드는 선언 시점 또는 객체 생성자 내부에서 반드시 초기화되어야 한다. 그러나 `@Value` 를 사용한 필드 주입은 객체가 먼저 생성된 후 필드에 값을 설정하는 방식이다. 즉, 값 주입이 객체 생성 이후에 일어나기 때문이다.

이 문제는 생성자 주입을 통해 해결할 수 있긴 하다. 생성자 파라미터에 `@Value` 어노테이션을 붙이게 되면 스프링은 객체를 생성하는 시점에 값을 가져와 생성자의 파라미터로 전달한다.

## 📌 @ConfigurationProperties

`@ConfigurationProperties` 의 핵심은 프로퍼티 값들을 하나의 객체로 묶어 관리할 수 있게 한다는 것이다. 특정 `prefix` 를 가진 설정들을 `POJO` 에 자동으로 바인딩한다.

```java
@ConfigurationProperties(prefix = "app.storage")
public record StorageProperties(
    String uploadDir,
    int maxFileSize,
    List<String> supportedFormats
) {}
```

먼저 프로퍼티를 주입할 클래스를 생성한다. `record` 를 사용하면 불변 객체를 간단하게 만들 수 있다. `prefix` 로 시작하는 설정을 찾아 객체 필드에 값을 할당한다. 참고로 `@ConfigurationProperties` 에는 SpEL을 사용할 수 없다.

```java
@ConfigurationProperties(prefix = "app.storage")
@Validated
public record StorageProperties(
    @NotEmpty
    String uploadDir,

    @Min(1024)
    int maxFileSize
) {}
```

프로퍼티 클래스에 `@Validated` 를 추가하면 각 필드에 대해 검증 어노테이션을 사용할 수 있게 된다. `@Value` 와 비교되는 차이점이다.

```java
@SpringBootApplication
@EnableConfigurationProperties(StorageProperties.class)
public class SpringBootApplication {

    // ...
    }
}
```

작성한 프로퍼티 클래스를 인식하고 빈으로 등록하기 위해서 메인 애플리케이션 클래스 또는 설정 클래스에 `@EnableConfigurationProperties` 을 추가해야 한다.

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class SpringBootApplication {

    // ...
    }
}
```

다만 여러 개의 프로퍼티 클래스가 존재하는 경우, 모든 클래스를 일일히 등록하는 것은 번거로우므로, `@ConfigurationPropertiesScan` 어노테이션을 사용한다. `@ConfigurationPropertiesScan` 어노테이션은 모든 `@ConfigurationProperties` 클래스를 자동으로 스캔하여 자동으로 빈으로 등록한다.

```java
@Service
@RequiredArgsConstructor
public class AnotherFileService {

    private final StorageProperties storageProperties;
    
    // ...
}
```

의존성을 주입한 후, 프로퍼티 클래스의 객체를 사용하면 된다.

`@ConfigurationProperties` 는 프로퍼티 값들이 지정된 타입의 필드로 바인딩되므로 컴파일 시점에 타입 오류를 잡을 수 있다. 비슷한 설정들을 하나의 객체로 묶어 관리하므로 응집성이 좋다.

`@Value` 와 비교되는 가장 큰 차이점은 `Relaxed Binding` 이다. `@Value` 어노테이션을 사용하면 프로퍼티 키를 정확히 일치시켜야 값을 가져올 수 있었다. 그러나 `@ConfigurationProperties` 는 다양한 이름 형태를 알아서 카멜 케이스로 바꿔 바인딩된다.

## 📌 결론

가장 큰 차이점은 `@Value` 어노테이션은 단일 값을 바인딩하는 반면 `@ConfigurationProperties` 어노테이션은 여러 값들을 한 번에 바인딩할 수 있다.

또한 유지보수 측면에서 `@ConfigurationProperties` 가 더 좋다. 특정 키를 여러 곳에서 사용하는 경우, 만약 이름이 변경된다면 일일히 키를 변경해주어야 한다.

`@Value` 는 간단한 한 두 개의 단일 값만 가져올 때 사용하면 좋을 것 같다. 사실 대부분의 프로젝트는 그렇지 않기에, 일반적인 경우 `@ConfigurationProperties` 를 우선적으로 생각하는 것이 유리할 것 같다.

## 📌 참고

https://j-dong.tistory.com/5