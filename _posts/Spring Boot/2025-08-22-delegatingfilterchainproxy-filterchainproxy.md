---
title : "[Spring Security] DelegatingFilterProxy와 FilterChainProxy"
date : 2025-08-22 18:00:00 +0900
categories : [Spring Boot]
tags : [java, 자바, spring boot, 스프링 부트, spring security, DelegatinFilterChainProxy, Filter, FilterChainProxy]
---

## 📌 Filter

Spring Security는 `Filter` 를 기반으로 동작한다. `Filter` 란 무엇일까?

![image.png](attachment:968926c5-5305-4c9b-a3db-85ff12d09022:image.png)

클라이언트가 `WAS, Web Application Server`에 요청(`HttpServletRequest`)을 보내면 서블릿 컨테이너는 URI와 `Servlet` 을 보고 `FilterChain` 을 생성한다.

요청은 최종적으로 `Servlet` 에 도착하는데, 이 과정에서 여러 개의 `Filter` 를 거치게 된다. 이러한 일련의 단계들을 `FilterChain` 이라고 한다.

`Servlet` 은 실질적인 비즈니스 로직을 수행하는 작업자이며, Spring MVC에서는 대부분이 `DispatcherServlet` 이 담당한다. 하나의 요청은 단 하나의 `Servlet` 에 의해 처리된다.

`Filter` 는 중간 작업자로, 진짜 이름 그대로 `Servlet` 에 도달하기까지 ‘필터’ 역할을 한다. 받은 요청을 거절하거나 조작할 수 있다.

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response,
		FilterChain chain) throws IOException, ServletException {
		
	chain.doFilter(request, response);
}
```

수행한 작업의 결과를 다음 필터에 넘기기 위해 `doFilter` 메서드는 `FilterChain` 을 파라미터로 받는다. `chain.doFilter(request, response)` 은 현재 작업의 요청과 응답을 다음 필터로 넘기는 동작을 수행한다.

이는 메서드 내부에서 `doFilter` 메서드를 호출하기 전후로 작업을 정의하여 전/후처리 작업을 수행할 수 있다는 것을 의미한다.

필터는 자신 뒤에 있는 필터와 서블릿에만 영향을 주기 때문에, 실행 순서가 중요하다.

## 📌 DelegatingFilterProxy

앞서 살펴 본 필터는 서블릿 컨테이너에서 생성하고 관리한다. 반면 개발자가 작성한 객체나 설정은 스프링의 IoC 컨테이너가 빈으로 관리한다. 이는 두 컨테이너의 생명 주기가 다르다는 것을 의미한다. 필터에서 `@Autowired` 를 통해 스프링 빈을 주입받을 수 없다는 것이다.

![image.png](attachment:4ef8d9fb-4275-472a-974a-a561f71b4ac9:image.png)

이 문제를 해결하기 위한 필터가 `DelegatingFilterProxy` 이다. `DelegatingFilterProxy` 에는 어떠한 보안 로직도 없으며, 유일하게 하는 동작은 스프링 컨테이너에서 ‘특별한 빈’을 찾아 해당 빈에게 모든 처리를 위임하는 것이다.

## 📌 FilterChainProxy

![image.png](attachment:e4b65c6e-275d-4a26-88fe-a3179bce56ae:image.png)

보안 처리를 위임받는 특별한 빈이 무엇일까? 바로 `FilterChainProxy` 이다. `FilterChainProxy` 는 스프링 IoC 컨테이너에 등록되어 있다.

```java
public class FilterChainProxy extends GenericFilterBean {

	// ...

	private List<SecurityFilterChain> filterChains;
}
```

`FilterChainProxy` 는 내부적으로 `List<SecurityFilterChain>` 를 가지고 있다. `DelegatingFilterProxy` 로부터 요청을 전달받으면 요청의 URI, HTTP 메서드 등을 보고 `filterChains` 에서 해당 요청에 가장 적합한 `SecurityFilterChain` 을 하나 선택한다. 이후 선택한 `SecurityFilterChain` 에 등록된 보안 필터들을 순차적으로 실행시키고, 성공적으로 통과되면 요청을 `DispatcherServlet` 으로 전달한다.

## 📌 참고

https://docs.spring.io/spring-security/reference/servlet/architecture.html