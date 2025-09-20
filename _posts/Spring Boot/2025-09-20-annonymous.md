---
title : "[Spring Security] anonymous"
date : 2025-09-20 10:50:00 +0900
categories : [Spring Boot]
tags : [java, 자바, spring boot, 스프링 부트, spring security, annonymous]
---

## 📌 익명 사용자 표현 방법의 변화

이전 인증 시스템에서는 인증되지 않는 사용자를 `null` 로 표현하는 경우가 많았다. 그러나 이는 코드에 `user == null` 과 같은 수많은 검증 로직 작성의 배경이 되었으며, `NullPointerException` 을 항상 고려해야 했다.

이러한 문제를 Spring Security는 ‘존재하지 않음’이라는 상태를 아무런 동작을 하지 않지만 실제 객체인 `AnonymousAuthenticationToken` 으로 표현한다.

> 인증된 사용자는 `UsernamePasswordAuthenticationToken` 으로 확인한다.

이를 통해 `SecurityContextHolder` 가 항상 `Authentication` 객체를 포함하게 된다. 즉, 인증 객체를 가져올 때 그 리턴 값이 절대 `null` 이 되지 않는다는 불변성을 보장하게 된다.

## 📌 AnonymousAuthenticationFilter

`AnonymousAuthenticationFilter` 는 인증 메커니즘 이후 다른 인증 필터들이 인증을 수행하지 못했을 때, 즉 `SecurityContext` 에 인증 객체가 존재하지 않는 경우 동작한다. `AnonymousAuthenticationFilter` 는 `SecurityContext` 가 비어있다면 `AnonymousAuthenticationToken`을 생성하여 context에 저장한다.

```java
public AnonymousAuthenticationFilter(String key, Object principal, List<GrantedAuthority> authorities) {
	Assert.hasLength(key, "key cannot be null or empty");
	Assert.notNull(principal, "Anonymous authentication principal must be set");
	Assert.notNull(authorities, "Anonymous authorities must be set");
	this.key = key;
	this.principal = principal;
	this.authorities = authorities;
}
```

초기에 필터가 생성되면 세 가지 정보가 설정된다. `key` 는 `AnonymousAuthenticationToken` 을 식별하는 고유한 키이다. `principal` 에는 어떤 사용자가 존재하는지 명시하며, 기본값으로 `anonymous` 가 세팅된다. `authorities` 에는 익명 사용자가 가지는 권한을 명시하며, 기본값으로 `ROLE_ANONYMOUS` 를 가진다.

```java
@Override
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
		throws IOException, ServletException {
	Supplier<SecurityContext> deferredContext = this.securityContextHolderStrategy.getDeferredContext();
	this.securityContextHolderStrategy
		.setDeferredContext(defaultWithAnonymous((HttpServletRequest) req, deferredContext));
	chain.doFilter(req, res);
}
```

이후 사용자의 요청은 `doFilter` 메서드로 들어오게 된다. 필터의 핵심 로직은 `defaultWithAnonymous` 메서드에 존재한다.

```java
private SecurityContext defaultWithAnonymous(HttpServletRequest request, SecurityContext currentContext) {
	Authentication currentAuthentication = currentContext.getAuthentication();
	if (currentAuthentication == null) {
		Authentication anonymous = createAuthentication(request);
		
		// ...
		
		SecurityContext anonymousContext = this.securityContextHolderStrategy.createEmptyContext();
		anonymousContext.setAuthentication(anonymous);
		return anonymousContext;
	}
	else {
	
		// ...
	}
	return currentContext;
}
```

먼저 `getAuthentication` 메서드를 통해 `SecurityContext` 에 이미 `Authentication` 객체가 존재하는지 확인한다. 앞선 필터들이 이미 사용자를 인증했는지 확인하게 위해서이다. 만약 존재한다면 필터는 아무런 동작을 수행하지 않고 넘어간다.

만약 존재하지 않는다면, 즉 해당 사용자가 인증되지 않았다면 `createAuthentication` 메서드를 호출하여 `AnonymousAuthenticationToken` 을 생성한다,

```java
protected Authentication createAuthentication(HttpServletRequest request) {
	AnonymousAuthenticationToken token = new AnonymousAuthenticationToken(this.key, this.principal,
			this.authorities);
	token.setDetails(this.authenticationDetailsSource.buildDetails(request));
	return token;
}
```

필터의 생성자에서 설정된 `key`, `principal`, `authorities` 를 통해 토큰을 생성하고 다른 세부 정보들을 토큰에 추가한 후 만들어진 토큰을 리턴한다.

```java
private SecurityContext defaultWithAnonymous(HttpServletRequest request, SecurityContext currentContext) {
	Authentication currentAuthentication = currentContext.getAuthentication();
	if (currentAuthentication == null) {
		// ...
		
		SecurityContext anonymousContext = this.securityContextHolderStrategy.createEmptyContext();
		anonymousContext.setAuthentication(anonymous);
		return anonymousContext;
	}
	else {
	
		// ...
	}
	return currentContext;
}
```

이후 토큰을 넣을 `SecurityContext` 를 생성한 후 토큰을 넣는다. 이렇게 생성된 `SecurityContext` 를 리턴하여 현재 요청의 `SecuritContextHolder` 에 설정된다.

## 📌 AnonymousAuthenticationToken

```java
// AnonymousAuthenticationFilter.java
protected Authentication createAuthentication(HttpServletRequest request) {
	AnonymousAuthenticationToken token = new AnonymousAuthenticationToken(this.key, this.principal,
			this.authorities);
	token.setDetails(this.authenticationDetailsSource.buildDetails(request));
	return token;
}

private AnonymousAuthenticationToken(Integer keyHash, Object principal,
		Collection<? extends GrantedAuthority> authorities) {
	super(authorities);
	Assert.isTrue(principal != null && !"".equals(principal), "principal cannot be null or empty");
	Assert.notEmpty(authorities, "authorities cannot be null or empty");
	this.keyHash = keyHash;
	this.principal = principal;
	setAuthenticated(true);
}
```

필터에 명시된 `key`, `principal`, `authorities` 를 세팅하고, `authenticated` 필드를 설정한다. 이 필드의 값은 `true` 로 설정되는데, 이는 사용자가 성공적으로 로그인하였다는 의미가 아니라 해당 `Authentication` 객체가 Spring Security가 신뢰하는 유효한 객체라는 것을 의미한다.

```java
public Object getCredentials() {
	return "";
}
```

토큰의 `credentials` 에는 빈 문자열이 세팅되는데, 이는 증명할 자격 증명이 없음을 의미한다.

## 📌 주의사항 및 특징

익명 사용자에게 과도한 권한을 부여하지 않도록 한다. 당연한 부분이긴 하다. 인증된 사용자와의 경계를 명확히 해야 한다.

`AnonymousAuthenticationToken`은 최소한의 정보만 보관하므로 메모리적으로 효율적이다. 또한 토큰이 세션에서 관리되지 않으므로 세션 공간을 차지하지 않는다.

익명 사용자 관련 처리는 `instanceof` 나 `authorities` 를 확인으로 간단히 이루어질 수 있다.