---
title : "[Spring Security] HttpSecurity와 WebSecurity"
date : 2025-08-21 18:45:00 +0900
categories : [Spring Boot]
tags : [java, 자바, spring boot, 스프링 부트, spring security, HttpSecurity, WebSecurity]
---

## 📌 WebSecurity

`WebSecurity` 는 전역적인 보안을 담당한다. `SecurityBuilder` 의 구현체로 `FilterChainProxy` 를 생성 및 설정하는 역할을 한다.

`FilterChainProxy` 는 모든 웹 요청을 가장 먼저 받아 어떤 `SecurityFilterChain` 을 적용할지 결정한다.

```java
@Bean
public WebSecurityCustomizer webSecurityCustomizer() {
    return (web) -> web.ignoring()
                       .requestMatchers("/css/**", "/js/**", "/images/**", "/favicon.ico");
}
```

현재는 `WebSecurityCustomizer` 빈을 등록하여 설정하는 것이 일반적이다.

`WebSecurity` 를 통해 특정 경로의 요청을 필터 체인에서 완전히 제외할 수있는데, 이때 사용하는 것이 `ignoring` 메서드이다. 즉, `ignoring` 메서드에 특정 경로를 명시하면 해당 경로들에 대해 `FilterChainProxy` 가 아예 동작하지 않도록 한다.

이는 곧 설정된 경로들에 대해 인증/인가 절차를 진행하지 않고, 여러가지 보안 기능들을 거치지 않는다는 의미이다. 따라서 기본적인 보안 절차는 거치면서 모든 사용자가 접근 가능하도록 하기 위해서 후술할 `HttpSecurity`의 `permitAll` 메서드를 사용한다.

```java
@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
public Filter springSecurityFilterChain() throws Exception {
	
	// ...
	return this.webSecurity.build();
}
```

`build` 메서드는 최종적으로 `FilterChainProxy` 라는 단일 `Filter` 객체를 생성한다.

## 📌 HttpSecurity

`HttpSecurity` 는 HTTP 요청 단위의 세부 보안을 담당한다. `WebSecurity` 를 통과한 요청이 실제로 어떤 인증/인가 규칙을 따라야 하는지 정의한다.

`HttpSecurity` 역시 `SecurityBuilder` 의 구현체로, `WebSecurity` 보다 더 세부적인 규칙을 정의한다. 다르게 말하면 `WebSecurity` 가 `HttpSecurity` 보다 상위 개념이라고 할 수 있다. 하나의 `HttpSecurity` 설정은 하나의 `SecurityFilterChain` 을생성한다. 역시 이는 `build` 메서드를 통해 생성된다.

`authorizeHttpRequests` 메서드를 통해 URL 패턴에 따른 역할, 권한 기반의 접근 허용 또는 거부 규칙을 설정할 수 있다.

`formLogin`, `oauth2Login` 등 여러 인증 메커니즘을 설정할 수 있다.

`csrf`, `cors` 등으로 각종 보안 기능을 설정할 수 있다.

`permitAll` 에 특정 경로 명시하면 인증 결과 무시

## 📌 WebSecurity vs. HttpSecurity

표로 정리해보았다.

| 구분 | WebSecurity | HttpSecurity |
| --- | --- | --- |
| 역할 | 전역 보안 설정 | HTTP 요청 기반 보안 설정 |
| 적용 레벨 | Spring Security 필터 체인 진입 전 | Spring Security 필터 체인 내부 |
| 주요 목적 | 특정 요청을 `SecurityFilterChain` 에서 완전히 무시 | 인증/인가 규칙 정의 |
| 최종 결과물 | `FilterChainProxy` | `SecurityFilterChain`  |
| 설정 방법 | `WebSecurityCustomizer` 빈 등록 | 빈으로  `SecurityFilterChain`을 리턴하는 메서드 정의 |

## 📌 참고

https://soojae.tistory.com/52