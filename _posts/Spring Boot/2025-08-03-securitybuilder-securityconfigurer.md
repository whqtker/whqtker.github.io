---
title : "[Spring Security] SecurityBuilder와 SecurityConfigurer"
date : 2025-08-03 11:35:00 +0900
categories : [Spring Boot]
tags : [java, 자바, spring boot, 스프링 부트, spring security, securitybuilder, securityconfigurer]
---

## 📌 개요

`Spring Security` 는 인증/인가 메커니즘을 `filter chain` 을 기반으로 처리한다.

`SecurityBuilder`, `SecurityConfigurer` 는 필터 체인과 관련된 빈들을 빌더 패턴을 사용하여 생성한다.

## 📌 SecurityBuilder, SecurityConfigurer

```java
public interface SecurityBuilder<O> {

	O build() throws Exception;

}
```

`SecurityBuilder` 는 웹 보안을 구성하는 데 필요한 빈 객체와 설정 클래스들을 생성하는 빌더이다. 제네릭 인터페이스이며, 빌더 패턴으로 보안 객체를 생성한다.

대표적인 구현체가 있다. `WebSecurity` 는 `FilterChainProxy` 를 생성한다. 전체 웹 보안 필터 체인을 관리하는 최상위 프록시이다. `ignoring` 과 같이 특정 요청을 보안 필터에서 아예 제외하는 전역적인 설정을 할 때 사용한다.

또 다른 구현체인 `HttpSecurity`는 `SecurityFilterChain` 을 생성하며, HTTP 요청에 대한 세부적인 보안 규칙(인증/인가, 세션, CSRF 방어 등)을 설정한다.

```java
public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {

	void init(B builder) throws Exception;
	void configure(B builder) throws Exception;

}
```

`SecurityConfigurer` 는 `SecurityBuilder` 에 구체적인 보안 설정을 추가하고 커스터마이징한다. `formLogin` , `csrf` 와 같은 세부적인 설정이 가능하며 실제 필터를 생성하고 초기화한다.

`init` 메서드는 `configure` 전에 필요한 공유 객체를 생성하거나 상태를 초기화하는 데 사용된다. 

`configurer` 메서드는 `init` 이 모두 끝난 후 호출되며 `SecurityBuilder` 에 필요한 속성을 설정하고 실제 필터를 생성하여 필터 체인에 추가하는 등 구체적인 보안 구성을 적용한다.

`.formLogin()`, `.csrf()` 와 같은 메서드는 내부적으로 `FormLoginConfigurer` 나 `CsrfConfigurer` 와 같은 `SecurityConfigurer` 의 구현체를 생성하고 `SecurityBuilder` 에 추가한다.

```java
@Override
public final O build() throws Exception {
	if (this.building.compareAndSet(false, true)) {
		this.object = doBuild();
		return this.object;
	}
	throw new AlreadyBuiltException("This object has already been built");
}
```

`build` 메서드는 내부적으로 `doBuild` 메서드를 호출하며,

```java
@Override
protected final O doBuild() throws Exception {
	synchronized (this.configurers) {
		this.buildState = BuildState.INITIALIZING;
		beforeInit();
		init();
		this.buildState = BuildState.CONFIGURING;
		beforeConfigure();
		configure();
		this.buildState = BuildState.BUILDING;
		O result = performBuild();
		this.buildState = BuildState.BUILT;
		return result;
	}
}
```

`doBuild` 메서드는 내부에서 `init` 과 `configure` 메서드를 호출한다.

```java
@SuppressWarnings("unchecked")
private void init() throws Exception {
	Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
	for (SecurityConfigurer<O, B> configurer : configurers) {
		configurer.init((B) this);
	}
	for (SecurityConfigurer<O, B> configurer : this.configurersAddedInInitializing) {
		configurer.init((B) this);
	}
}

@SuppressWarnings("unchecked")
private void configure() throws Exception {
	Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
	for (SecurityConfigurer<O, B> configurer : configurers) {
		configurer.configure((B) this);
	}
}
```

`init` 와 `configure` 메서드는 각각 내부적으로 `SecurityConfigurer` 의 `init` 과 `configure` 메서드를 호출한다. 

```java
public <C extends SecurityConfigurer<O, B>> C apply(C configurer) throws Exception {
	add(configurer);
	return configurer;
}
```

`AbstractConfiguredSecurityBuilder` 에는 `apply` 메서드가 있는데, `SecurityConfigurer` 를 `SecurityBuilder` 구현체에 추가한다.

---

`SecurityBuilder` 의 대표적인 구현체로 `WebSecurity` 와 `HttpSecurity` 가 있었다.

`WebSecurity` 의 `build` 메서드는 `FilterChainProxy` 를 리턴하며, `HttpSecurity` 의 `build` 메서드는 `SecurityFilterChain` 을 리턴한다.

```java
public class FilterChainProxy extends GenericFilterBean {

	// ...

	private List<SecurityFilterChain> filterChains;
}
```

`FilterChainProxy` 는 내부적으로 `SecurityFilterChain` 을 가지고 있다. 요청이 들어오면 `SecurityFilterChain`을 통해 인증을 수행한다.

```java
@Autowired(required = false)
public void setFilterChainProxySecurityConfigurer(ObjectPostProcessor<Object> objectPostProcessor,
		ConfigurableListableBeanFactory beanFactory) throws Exception {
	this.webSecurity = objectPostProcessor.postProcess(new WebSecurity(objectPostProcessor));
	if (this.debugEnabled != null) {
		this.webSecurity.debug(this.debugEnabled);
	}
	List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers = new AutowiredWebSecurityConfigurersIgnoreParents(
			beanFactory)
		.getWebSecurityConfigurers();
	webSecurityConfigurers.sort(AnnotationAwareOrderComparator.INSTANCE);
	Integer previousOrder = null;
	Object previousConfig = null;
	for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
		Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
		if (previousOrder != null && previousOrder.equals(order)) {
			throw new IllegalStateException("@Order on WebSecurityConfigurers must be unique. Order of " + order
					+ " was already used on " + previousConfig + ", so it cannot be used on " + config + " too.");
		}
		previousOrder = order;
		previousConfig = config;
	}
	for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
		this.webSecurity.apply(webSecurityConfigurer);
	}
	this.webSecurityConfigurers = webSecurityConfigurers;
}
```

`WebSecurityConfiguration` 빈이 생성되고 초기화될 때, `setFilterChainProxySecurityConfigurer` 가 호출된다. 모든 `SecurityConfigurer` 를 `apply` 메서드를 통해 `WebSecurity` 에 추가한다.

```java
@Bean(HTTPSECURITY_BEAN_NAME)
@Scope("prototype")
HttpSecurity httpSecurity() throws Exception {
	LazyPasswordEncoder passwordEncoder = new LazyPasswordEncoder(this.context);
	AuthenticationManagerBuilder authenticationBuilder = new DefaultPasswordEncoderAuthenticationManagerBuilder(
			this.objectPostProcessor, passwordEncoder);
	authenticationBuilder.parentAuthenticationManager(authenticationManager());
	authenticationBuilder.authenticationEventPublisher(getAuthenticationEventPublisher());
	HttpSecurity http = new HttpSecurity(this.objectPostProcessor, authenticationBuilder, createSharedObjects());
	WebAsyncManagerIntegrationFilter webAsyncManagerIntegrationFilter = new WebAsyncManagerIntegrationFilter();
	webAsyncManagerIntegrationFilter.setSecurityContextHolderStrategy(this.securityContextHolderStrategy);
	// @formatter:off
	http
		.csrf(withDefaults())
		.addFilter(webAsyncManagerIntegrationFilter)
		.exceptionHandling(withDefaults())
		.headers(withDefaults())
		.sessionManagement(withDefaults())
		.securityContext(withDefaults())
		.requestCache(withDefaults())
		.anonymous(withDefaults())
		.servletApi(withDefaults())
		.apply(new DefaultLoginPageConfigurer<>());
	http.logout(withDefaults());
	// @formatter:on
	applyDefaultConfigurers(http);
	return http;
}
```

`WebSecurityConfiguration` 를 통해 `FilterChainProxy` 객체를 만들기 위해 `SecurityFilterChain` 빈을 수집해아 한다. 이 과정에서 `HttpSecurity` 가 필요하므로,  `HttpSecurityConfiguration` 의 `httpSecurity` 메서드가 호출된다. 메서드 체이닝을 통해 `HttpSecurity` 객체에 여러 `SecurityConfigurer` 를 추가한다.

```java
public HttpSecurity csrf(Customizer<CsrfConfigurer<HttpSecurity>> csrfCustomizer) throws Exception {
	ApplicationContext context = getContext();
	csrfCustomizer.customize(getOrApply(new CsrfConfigurer<>(context)));
	return HttpSecurity.this;
}
```

`csrf` 메서드를 보면, `getOrApply` 메서드를 통해 `CsrfConfigurer` 를 추가하는 것을 볼 수 있다. 나머지 메서드도 마찬가지로 적절한 `SecurityConfigurer` 를 추가한다.

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnDefaultWebSecurity
static class SecurityFilterChainConfiguration {

	@Bean
	@Order(SecurityProperties.BASIC_AUTH_ORDER)
	SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
		http.authorizeHttpRequests((requests) -> requests.anyRequest().authenticated());
		http.formLogin(withDefaults());
		http.httpBasic(withDefaults());
		return http.build();
	}

}
```

이후 `httpSecurity` 메서드에서 리턴된 `HttpSecurity` 는 `SecurityFilterChainConfiguration` 의 `build` 메서드를 통해 `SecurityFilterChain` 에 주입된다.

```java
@Autowired(required = false)
void setFilterChains(List<SecurityFilterChain> securityFilterChains) {
	this.securityFilterChains = securityFilterChains;
}
```

만들어진 `SecurityFilterChain` 은 `WebSecurityConfiguration` 의 `setFilterChains` 메서드를 통해 설정된다.

```java
@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
public Filter springSecurityFilterChain() throws Exception {
	boolean hasFilterChain = !this.securityFilterChains.isEmpty();
	if (!hasFilterChain) {
		this.webSecurity.addSecurityFilterChainBuilder(() -> {
			this.httpSecurity.authorizeHttpRequests((authorize) -> authorize.anyRequest().authenticated());
			this.httpSecurity.formLogin(Customizer.withDefaults());
			this.httpSecurity.httpBasic(Customizer.withDefaults());
			return this.httpSecurity.build();
		});
	}
	for (SecurityFilterChain securityFilterChain : this.securityFilterChains) {
		this.webSecurity.addSecurityFilterChainBuilder(() -> securityFilterChain);
	}
	for (WebSecurityCustomizer customizer : this.webSecurityCustomizers) {
		customizer.customize(this.webSecurity);
	}
	return this.webSecurity.build();
}
```

설정된 `SecurityFilterChain` 은 `springSecurityFilterChain` 메서드에서 빌더 형태로 감싸서 `WebSecurity` 에 추가한다.

```java
@Override
protected Filter performBuild() throws Exception {

	// ...
	
	for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : this.securityFilterChainBuilders) {
		SecurityFilterChain securityFilterChain = securityFilterChainBuilder.build();
		securityFilterChains.add(securityFilterChain);
		requestMatcherPrivilegeEvaluatorsEntries
			.add(getRequestMatcherPrivilegeEvaluatorsEntry(securityFilterChain));
	}
	if (this.privilegeEvaluator == null) {
		this.privilegeEvaluator = new RequestMatcherDelegatingWebInvocationPrivilegeEvaluator(
				requestMatcherPrivilegeEvaluatorsEntries);
	}
	FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
	
	// ...
}
```

`performBuild` 메서드는 `WebSecurity` 객체의 `build` 메서드 내부에서 호출된다. `addSecurityFilterChainBuilder` 메서드를 통해 수집한 `SecurityFilterChain` 의 빌더를 다시 `SecurityFilterChain` 객체로 리턴받고, 이들을 통해 `FilterChainProxy` 객체를 생성한다.

## 📌 동작 과정

전체적인 동작 과정을 정리하여 살펴보자.

1. `WebSecurityConfiguration`, `HttpSecurityConfiguration` 과 같은 설정 파일이 IoC 컨테이너에 로드된다.
2. `WebSecurityConfiguration` 빈을 초기화하는 과정에서 `setFilterChainProxySecurityConfigurer` 메서드가 호출되고, `WebSecurity` 객체가 생성된다.
3. 존재하는 모든 `WebSecurityConfigurer` 타입의 빈을 찾고, `apply` 메서드를 통해 `WebSecurity` 빌더에 적용한다.
4. 빈으로 등록된 `SecurityFilterChain` 을 찾는다. 단, `SecurityFilterChain` 빈을 생성하기 위해 파라미터로 `HttpSecurity` 객체가 필요하므로 `HttpSecurityConfiguration` 의 `httpSecurity` 메서드를 통해 새로운 `HttpSecurity` 객체를 생성하여 파라미터에 주입한다.
5. 주입받은 `http` 객체에 메서드 체이닝을 통해 보안 설정을 적용하고, `build` 메서드를 호출한다.
6. `build` 는 내부적으로 `doBuild` 메서드를 호출하여 `init` 과 `configure` 메서드를 호출하여 보안 필터를 구성하고, 이들을 담은 `SecurityFilterChain` 객체를 생성하여 리턴한다. 생성된 `SecurityFilterChain` 객체는 빈으로 등록된다.
7. 다시 3번 동작에서, `setFilterChains` 메서드를 호출하여 존재하는 모든 `SecurityFilterChain` 빈들을 하나의 `List` 로 묶는다.
8. 이후 `springSecurityFilterChain` 메서드에서 주입받은 `securityFilterChains` 리스트를 순회하며 `SecurityFilterChain` 객체를 빌더 형태로 감싸 `WebSecurity` 객체에 등록한다.
9. 모두 등록되면 `WebSecurity` 의 `build` 메서드를 호출한다. 이 메서드는 내부적으로 `performBuild` 메서드를 호출한다.
10. `performBuild` 메서드에서 빌더 형태로 감싼 `SecurityFilterChain` 을 다시 원래 `SecurityFilterChain` 객체로 변환한 후, 이들을 통해 최종 `FilterChainProxy` 객체를 생성한다. 생성된 `FilterChainProxy` 객체는 스프링 컨테이너의 빈으로 등록된다.

## 📌 참고

https://ttl-blog.tistory.com/1165