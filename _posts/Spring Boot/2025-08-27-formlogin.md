---
title : "[Spring Security] formLogin"
date : 2025-08-27 18:00:00 +0900
categories : [Spring Boot]
tags : [java, 자바, spring boot, 스프링 부트, spring security, formlogin]
---

## 📌 formLogin 메서드란?

`formLogin` 메서드는 로그인 폼을 통한 인증을 쉽게 구현할 수 있도록 하는 기능이다.

## 📌 formLogin API

### loginPage

`loginPage` 는 Spring Security가 기본적으로 제공하는 로그인 폼 대신 개발자가 만든 커스텀 로그인 폼을 사용하려고 할 때 사용한다.

### loginProcessingUrl

`loginProcessingUrl` 메서드에는 인증 절차를 가로채서 수행할 엔드포인트를 작성한다. 해당 URL로 자격 증명이 들어오면 `UsernamePasswordAuthenticationFilter` 가 이를 가로채서 `AuthenticationManager` 에게 전달한다. 이후 `UserDetailService` 구현체를 찾아 `loadUserByUsername` 메서드를 호출하고, 리턴된 `UserDetails` 객체의 비밀번호를 비교하여 일치하는지 검증한다. 인증에 성공하면 사용자 정보와 권한을 담아 `Authentication` 객체를 생성할 후 `SecurityContext` 에 저장한다.

### defaultSuccessUrl

`defaultSuccessUrl` 은 로그인에 성공했을 때 이동할 기본 URL을 지정하는 메서드이다. 다만 로그인이 성공했을 때 항상 `defaultSuccessUrl` 에 명시된 URL로 이동하는 것이 아니다.

사용자가 로그인을 시도할 때, 원래 가려고 한 URL을 세션의 `RequestCache` 에 `SavedRequest` 객체로 저장하는데, 만약 `SavedRequest` 가 없다면 `defaultSuccessUrl` 에 명시된 URL로 이동하는 것이다.

만약 `SavedRequest` 의 존재 여부를 확인하지 않고 무조건 `defaultSuccessUrl` 의 URL로 리디렉션하고 싶다면 두 번째 파라미터인 `alwaysUse` 의 값을 `true` 로 설정하면 된다.

다만 `defaultSuccessUrl` 설정은 후술할 `successHandler` 가 설정되었다면 완전히 무시된다.

### failureUrl

`failureUrl` 은 로그인 시도가 실패했을 때 어느 URL로 리디렉션할지 지정하는 메서드이다.

`failureUrl` 또한 `failureHandler` 가 설정되었다면 완전히 무시된다.

### usernameParameter

`usernameParameter` 메서드는 사용자 아이디를 어떤 HTTP 파라미터로 받을지 알려준다. 이 설정에 따라 추출된 값이 `loadUserByUsername` 메서드의 `username` 파라미터로 전달된다.

### passwordParameter

`passwordParameter` 또한 비밀번호를 어떤 파라미터로 받을지 알려주는 메서드이다. 추출된 값은 `UserDetails` 에 저장된 비밀번호 해시 값과 비교하여 인증한다.

### failureHandler

`failureUrl` 메서드가 단순히 URL을 리디렉션하는 것에 비해  `failureHandler` 는 더 다양한 동작을 수행할 수 있다. `AuthenticationFailureHandler` 인터페이스를 구현하거나 `SimpleUrlAuthenticationFailureHandler` 를 상속하여 클래스를 만들고 인자로 넘겨준다.

```java
@Override
public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
		AuthenticationException exception) throws IOException, ServletException {
	
	// ...
}
```

보통 `onAuthenticationFailure` 메서드를 구현하여 사용하는데, `HttpServletRequest` 에서 요청에 대한 정보를 가져올 수 있고, `HttpServletResponse` 에서 리디렉션같은 응답을 제어할 수 있다. `exception` 을 통해 예외 타입을 확인하여 분기 처리를 할 수 있다.

### successHandler

`successHandler` 또한 `defaultSuccessUrl` 보다 정교한 작업을 수행한다. `AuthenticationSuccessHandler` 인터페이스를 구현하거나 `SimpleAuthenticationSuccessHandler` 를 상속받아 클래스를 구현한다.