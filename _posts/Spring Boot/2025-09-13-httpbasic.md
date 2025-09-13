---
title : "[Spring Security] httpBasic, HTTP Basic Authentication"
date : 2025-09-13 21:30:00 +0900
categories : [Spring Boot]
tags : [java, 자바, spring boot, 스프링 부트, spring security, httpBasic]
---

## 📌 인증 메커니즘 원리

`HTTP Basic Authentication` 은 특정 리소스에 대한 접근을 요청할 때 사용자에게 `username` 과 `password` 를 확인하여 인가를 수행하는 방법이다.

클라이언트가 보호된 리소스에 접근하면 서버는 `Authorization` 헤더의 존재 유무를 확인하고, 응답의 `WWW-Authenticate` 헤더에 `Basic` 스킴과 `realm` 매개변수를 포함하여 다시 클라이언트로 전달한다.

클라이언트는 사용자의 자격 증명을 `username:password` 형태의 문자열로 만든 후 이를 `Base64` 로 인코딩하여 `Authrization` 헤더에 담아 서버에 전송한다.

### Spring Boot에서 HTTP Basic Authentication 구현

먼저 사용자가 보호된 리소스에 처음 접근을 시도하면 서버는 `SecurityFilterChain` 에서 `BasicAuthenticationFilter` 가 요청을 받는다. `BasicAuthenticationFilter` 는 `OncePerRequestFilter` 를 상속받아 구현되는데, 요청 당 한 번의 실행을 보장할 수 있도록 한다.

> 이는 `HTTP Basic Authentication이 stateless한 특성을 가진다는 것을 의미한다.

`BasicAuthenticationFilter` 는 form 기반 인증과 달리 매 요청마다 `Authorization` 헤더의 존재 여부를 확인 및 인증을 시도한다. 헤더가 없다면 인증되지 않은 요청으로 간주하고, `401 Unauthorized` 와 함께 응답을 클라이언트에 전달한다. 헤더가 존재하면 `Base64` 로 인코딩된 토큰을 추출하고 디코딩하여 `username` 과 `password` 를 분리하여 `UsernamePasswordAuthenticationToken` 을 생성한 후, `AuthenticationManager` 에게 인증을 위임한다. 인증이 성공하면 `SecurityContextHolder` 에 인증 정보를 저장한다.

전달되는 응답의 `WWW-Authenticate` 헤더에는 `Basic` 스킴을 지정하고, `realm` 매개변수에 ‘보호 영역’을 지정한다. 클라이언트는 `Basic` 스킴을 확인하고, 자격 증명을 `Base64` 로 인코딩한다. 여기서 ‘보호 영역’이란 사용자가 어떤 시스템에 로그인하려는지를 알려준다.

## 📌 HTTP Basic Authentication의 장/단점

```jsx
const username = 'admin';
const password = 'secret123';
const credentials = btoa(`${username}:${password}`);

fetch('/api/data', {
    headers: {
        'Authorization': `Basic ${credentials}`
    }
});
```

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.httpBasic(withDefaults());
    return http.build();
}
```

`HTTP Basic Authentication`의 가장 큰 장점은 단순한 구현이다. 동작에 있어 토큰만 사용되고, 쿠키, 세션 등은 필요하지 않다. 또한 HTTP 표준 프로토콜에 정의된 방식이므로 대부분의 클라이언트 및 서버에 기본적으로 지원된다.

다만 사용자의 자격 증명이 `Base64` 인코딩이 되어 서버로 전달되는데, **인코딩은 암호화가 아니다.** 즉, 인코딩된 자격 증명이 탈취되면 디코딩을 통해 자격 증명을 쉽게 얻을 수 있다.

HTTPS 통신을 사용한다면 `Man-in-the-Middle` 공격에 대한 노출 위험을 크게 줄일 수 있긴 하다.

## 📌 HTTP Basic Authentication vs. formLogin

`formLogin` 은 세션 방식이고, `HTTP Basic Authentication` 은 stateless 방식이다. 따라서 `formLogin` 은 최초 로그인 후 서버에서 세션 상태를 유지하여 같은 사용자에 대한 인증을 생략할 수 있으나, `httpBasic` 은 그렇지 않다.