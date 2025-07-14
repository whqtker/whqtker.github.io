---
title : "ResponseEntity, HTTP 상태 코드"
date : 2025-07-15 00:15:00 +0900
categories : [Spring Boot]
tags : [spring boot, 스프링 부트, java, 자바, ResponseEntity, HTTP]
---

## 📌 ResponseEntity란?

`ResponseEntity` 는 HTTP 응답을 제어할 수 있도록 도와주는 클래스이다. 응답 body, 상태 코드, 헤더를 제어할 수 있다. `HttpEntity` 를 상속받아 구현되었다.

`@ResponseBody` 는 응답의 body만 제어할 수 있다는 점에서 차이가 있다.

## 📌 구성 요소

```java
public ResponseEntity(HttpStatusCode status) {
	this(null, null, status);
}

public ResponseEntity(@Nullable T body, HttpStatusCode status) {
	this(body, null, status);
}

public ResponseEntity(MultiValueMap<String, String> headers, HttpStatusCode status) {
	this(null, headers, status);
}

public ResponseEntity(@Nullable T body, @Nullable MultiValueMap<String, String> headers, HttpStatusCode status) {
	this(body, headers, (Object) status);
}

public ResponseEntity(@Nullable T body, @Nullable MultiValueMap<String, String> headers, int rawStatus) {
	this(body, headers, (Object) rawStatus);
}

private ResponseEntity(@Nullable T body, @Nullable MultiValueMap<String, String> headers, Object status) {
	super(body, headers);
	Assert.notNull(status, "HttpStatusCode must not be null");
	this.status = status;
}
```

`ResponseEntity` 는 세 가지 요소를 포함하고 있다.

- `HttpStatus` 는 ‘200 OK’, ‘404 Not Found’와 같은 HTTP 상태 코드 정보를 담고 있다.
- `HttpHeaders` 는 `Content-type'과 같은 HTTP 헤더 정보를 담고 있다.
- `HttpBody` 는 실제 응답 body 데이터를 담고 있다.

## 📌 HTTP 상태 코드

일단 어떤 종류의 HTTP 상태 코드가 존재하는지 확인해보자.

크게 5가지 그룹이 존재한다.

- 1xx: 특정 정보를 제공할 때 사용한다. 요청을 받았으며 프로세스를 계속 진행함을 의미한다.
    - 100 Continue: 서버가 요청의 첫 부분을 받고 아직 수신이 끝나지 않았음을 의미하며 나머지 부분을 계속 보내도 좋다는 것을 의미한다.
    - 101 Switching Protocols: 요청의 `Upgrade` 헤더에 따라 서버가 프로토콜을 변경하고 있는 상태이다.
    - 102 Processing: 서버가 요청을 수신했고, 처리 중이지만 응답을 보낼 수 없는 상태이다.
- 2xx: 요청이 성공적으로 처리되었음을 의미한다.
    - 200 OK: 요청이 성공적으로 처리된 상태이다.
    - 201 Created: 요청이 성공하여 새로운 리소스가 생성된 상태이다.
    - 202 Accepted: 요청이 접수되었으나 아직 처리되지 않은 상태이다.
    - 204 No Content: 요청은 성공했으나 클라이언트에게 보낼 응답 body가 없음을 의미한다.
    - 206 Partial Content: 클라이언트가 `Range` 헤더를 통해 리소스의 일부만 요청했을 때 해당 콘텐츠가 성공적으로 전송되었음을 의미한다.
- 3xx: 요청을 완료하기 위해 리다이렉션해야 함을 의미한다.
    - 301 Moved Permanently: 요청한 리소스의 URI가 영구적으로 변경되었음을 의미한다. 응답의 `Location` 헤더에 새로운 URI을 포함시킨다.
    - 302 Found: 요청한 리소스가 일시적으로 다른 URI에 있는 상태이다.
    - 304 Not Modified: 클라이언트는 캐시된 파일의 버전이 변경되었는지 확인하기 위해 서버에 조건부 요청을 보내는데, 해당 버전이 변경되지 않았음을 의미한다.
    - 307 Temporary Redirect: 302와 유사하나 리다이렉트 시 기존 요청의 HTTP 메서드를 변경하지 않고 그대로 사용해야 함을 의미한다.
    - 308 Permanent Redirect: 301과 유사하나 리다이렉트 시 기존 요청의 HTTP 메서드를 변경하지 않고 그대로 사용해야 함을 의미한다.
- 4xx: 클라이언트의 오류로 서버가 요청을 처리할 수 없음을 의미한다.
    - 400 Bad Request: 잘못된 문법으로 인해 서버가 요청을 이해할 수 없음을 의미한다.
    - 401 Unauthorized: 인증되지 않은 사용자의 요청임을 의미한다.
    - 403 Forbidden: 서버가 요청을 수신했으나 해당 리소스에 접근할 권한이 없음을 나타낸다.
    - 404 Not Found: 요청한 리소스를 찾을 수 없음을 의미한다.
    - 405 Method Not Allowed: 요청한 URI는 존재하나 해당 URI에서 허용되지 않는 HTTP 메서드를 사용한 경우이다.
    - 409 Conflict: 요청이 서버의 현재 상태와 충돌하여 완료될 수 없음을 의미한다.
    - 415 Unsupported Media Type: 요청의 `Content-Type` 을 지원하지 않음을 의미한다.
    - 429 Too Many Requests: 특정 시간 동안 너무 많은 요청이 보낸 상태이다.
- 5xx: 서버 측 오류로 요청을 처리할 수 없음을 의미한다.
    - 500 Internal Server Error: 서버에 여상치 못한 문제가 발생한 상태이다.
    - 502 Bad Gateway: 개이트웨이 또는 프록시 서버가 실제 요청을 처리하는 서버로부터 잘못된 응답을 받았음을 의미한다.
    - 503 Service Unavailable: 서버가 과부하 또는 유지보수 등으로 인해 요청을 처리할 수 없음을 의미한다.
    - 504 Gatewat Timeout: 게이트웨이 또는 프록시 서버가 실제 요청을 처리하는 서버로부터 시간 내 응답을 받지 못했음을 의미한다.

## 📌 생성 방법

```java
// 생성자를 사용하는 방법
HttpHeaders headers = new HttpHeaders();
headers.add("Custom-Header", "value");
return new ResponseEntity<>(user, headers, HttpStatus.OK);

// 빌더 패턴을 사용하는 방법
return ResponseEntity
    .status(HttpStatus.CREATED)
    .header("Location", "/users/123")
    .body(createdUser);

```

생성자와 빌더 패턴 두 가지 방법이 존재하는데, 보통 빌더 패턴을 많이 사용한다.

## 📌 장점

`ResponseEntity` 를 사용하면 요청 처리 결과에 따라 적절한 상태 코드를 설정할 수 있다. 또한 서비스 계층에서 발생한 예외에 따라 컨트롤러 계층에서 구체적인 에러 응답을 생성할 수 있다.