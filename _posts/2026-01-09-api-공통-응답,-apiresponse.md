---
share: true
categories:
  - Spring Boot
title: API 공통 응답, ApiResponse
date: 2026-01-09 23:37:27 +0900
slug: 2026-01-09-api-공통-응답,-apiresponse
tags:
  - Spring_Boot
  - 스프링부트
description: API 공통 응답, ApiResponse
---

## 왜 공통 응답 객체를 사용할까?

가장 큰 이유는 **클라이언트와 협업 시 유지보수의 효율성이 증가**하기 때문이다. 클라이언트 입장에서 API가 동일한 구조를 가지므로, 공통 인터셉터 등을 만들어 처리할 수 있게 된다.

**단순히 HTTP 상태 코드만으로 구체적인 비즈니스 로직 결과를 표현하기 어렵다.** 예를 들어 이메일 형식이 잘못된 경우와, 이미 가입된 이메일인 경우 모두 `400 Bad Request`일 수 있으나, 클라이언트는 다른 메시지를 띄워야 한다. 서버에서 에러 코드를 유지하고 있는 경우, 응답에 에러 코드를 넣어 보내면 된다. 예를 들어 이메일 형식 오류는 `code: U001`, 중복 가입 오류는 `code: U002`로 설정하면 된다.

또한 **확장성** 측면에서도 유리하다. 만약 모든 API 응답에 서버 응답 시간을 넣어 전송해야 한다고 하자. 공통 응답 객체가 없다면 모든 컨트롤러 메서드를 수정해야 하지만, 있다면 `ApiResponse` 클래스에 필드를 추가하면 된다.

> `ApiResponse`는 객체 이름일 뿐이다. 이름으로 `ApiResponse`를 사용해야 한다고 정해진 것이 아니다.

## 공통 응답 객체를 생성했다. 그 다음은?

공통 응답 객체만 생성하면 끝일까? 모든 컨트롤러의 응답으로 `ApiResponse`를 사용해도록 구현해야 할까? 그보단, **클라이언트에 응답을 전달하기 전 `ApiResponse` 형식으로 변환**하여 전달하면 될 것이다. 이때 사용하는 것이 `ResponseBodyAdvice` 이다.

> `Advice`는 AOP에서 사용되는 개념으로, 자세한 내용은 AOP를 참고.

비즈니스 로직을 수행한 후, 컨트롤러가 DTO를 리턴하면 `ResponseBodyAdvice` 구현체가 이를 잡아 `ApiResponse` 구조로 변환한다.

즉, `ResponseBodyAdvice`를 사용하면 **컨트롤러를 작성할 때 리턴 타입에 대해 신경쓰지 않아도 된다.** 원래 구현하던 방식으로 DTO를 리턴하면 알아서 `ApiResponse` 구조로 변환해주기 때문이다.

## 이게 끝일까?

아니다. 만약 예외를 던지는 경우 Spring Boot의 기본 에러 처리기가 동작하게 되는데, 이 또한 `ApiResponse` 구조로 바꾸기 위해 예외를 잡아야 한다. 이때 사용하는 어노테이션이 `@ExceptionHandler`이다. Spring은 컨트롤러 밖으로 예외가 던져진 경우 먼저 `@ExceptionHandler`가 붙은 메서드가 있는지 확인하고, 그렇지 않다면 기본 에러 처리기가 동작한다.

`@ExceptionHandler` 메서드를 따로 관리하는 클래스를 생성하고(보통 이름은 `GlobalExceptionHandler` 라고 명명한다.), 해당 메서드에서 예외를 적절히 `ApiResponse`로 변환해주면 된다.

## 예제

```java
public record ExampleResponse(  
        String message  
) {  
    public static ExampleResponse from(String message) {  
        return new ExampleResponse(message);  
    }  
}
```

```java
@Service  
@RequiredArgsConstructor  
public class ExampleService {  
  
    public ExampleResponse getNormalResponse() {  
        return ExampleResponse.from("Hello World!");  
    }  
}
```

```java
@RestController  
@RequestMapping("/api")  
@RequiredArgsConstructor  
public class ExampleController {  
  
    private final ExampleService exampleService;  
  
    @GetMapping("/normal")  
    public ResponseEntity<ExampleResponse> getNormalResponse() {  
        return ResponseEntity.ok(exampleService.getNormalResponse());  
    }  
}
```

`/normal` 에 접근하면 정상적인 응답을 받는 간단한 예제이다. 실제로 응답이 어떤 형식으로 받아지는지 확인해보자.

---

```json
// http://localhost:8080/api/normal
{
    "message": "Hello World!"
}
```

`ExampleResponse` 와 동일한 구조로 응답이 오는 것을 확인할 수 있다.

---

이번엔 공통 응답 객체 `ApiResponse`를 작성하고, 모든 응답이 `ApiResponse` 구조를 따르도록 설정해보자.

```java
@JsonInclude(JsonInclude.Include.NON_NULL)  
public record ApiResponse<T>(  
        boolean success,  
        T data  
) {  
  
    public static <T> ApiResponse<T> of(T data) {  
        return new ApiResponse<>(true, data);  
    }  
  
    public static ApiResponse<Void> empty() {  
        return new ApiResponse<>(true, null);  
    }  
}
```

위와 같이 `ApiResponse`를 생성한다.

`success`는 성공 여부, `data`는 실제 응답 데이터를 담는 변수이다. 추후 작성할 어드바이스에서 `of`, `empty` 메서드를 적절히 사용하여 응답 구조를 변경할 것이다.

`empty` 메서드에서 `data`는 `null`인데, `@JsonInclude(JsonInclude.Include.NON_NULL)` 에 의해 JSON 응답에서 생략된다.

---

```java
@RestControllerAdvice  
public class ApiResponseAdvice implements ResponseBodyAdvice<Object> {  
  
    @Override  
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {  
        // 이미 ApiResponse인 경우 래핑하지 않음  
        return !returnType.getParameterType().equals(ApiResponse.class);  
    }  
  
    @Override  
    public Object beforeBodyWrite(  
            Object body,  
            MethodParameter returnType,  
            MediaType selectedContentType,  
            Class<? extends HttpMessageConverter<?>> selectedConverterType,  
            ServerHttpRequest request,  
            ServerHttpResponse response  
    ) {  
        // 이미 ApiResponse인 경우 그대로 반환  
        if (body instanceof ApiResponse) {  
            return body;  
        }  
  
        // null인 경우 빈 성공 응답  
        if (body == null) {  
            return ApiResponse.empty();  
        }  
  
        // 일반 응답을 ApiResponse로 래핑  
        return ApiResponse.of(body);  
    }  
}
```

`ResponseBodyAdvice`의 구현체 `ApiResponseAdvice`는 위와 같다.

`supports` 메서드는 이 어드바이스 로직을 실행할지 여부를 결정한다. 리턴 타입이 `ApiResponse`가 아닌 경우에만 실행되도록 설정하였다.

`beforeBodyWrite`는 실제 응답 객체를 `ApiResponse`로 감싸는 동작을 수행한다. 혹여나 `supports` 메서드를 통과했으나 응답 객체가 `ApiResponse`인 경우 그대로 리턴하도록 설정했다.

> 이런 상황이 가능할까? 메서드 리턴 타입이 `Object`이고, `ApiResponse`를 리턴하는 경우 가능하다.

이후 컨트롤러가 `null`을 리턴한 경우 `empty`를 호출하고, 그렇지 않은 일반 응답이라면 `ApiResponse`로 래핑하여 리턴한다.

---

다시 `/normal`에 접근해보자.

```json
{
    "success": true,
    "data": {
        "message": "Hello World!"
    }
}
```

`ApiResponse`의 구조로 래핑되어 리턴된다!

---

```java
@Service  
@RequiredArgsConstructor  
public class ExampleService {  
  
    public ExampleResponse getNormalResponse() {  
        return ExampleResponse.from("Hello World!");  
    }  
  
    public ExampleResponse getErrorResponse() {  
        throw new IllegalArgumentException("올바르지 않은 요청입니다.");  
    }  
}
```

```java
@RestController  
@RequestMapping("/api")  
@RequiredArgsConstructor  
public class ExampleController {  
  
    private final ExampleService exampleService;  
  
    @GetMapping("/normal")  
    public ResponseEntity<ExampleResponse> getNormalResponse() {  
        return ResponseEntity.ok(exampleService.getNormalResponse());  
    }  
  
    @GetMapping("/error")  
    public ResponseEntity<ExampleResponse> getErrorResponse() {  
        return ResponseEntity.ok(exampleService.getErrorResponse());  
    }  
}
```

예외를 던지는 `/error` 엔드포인트를 추가하였다. 호출하면 어떤 응답을 받게 될까?

```json
{
    "success": true,
    "data": {
        "timestamp": "2026-01-09T13:58:25.162Z",
        "status": 500,
        "error": "Internal Server Error",
        "trace": "java.lang.IllegalArgumentException: 올바르지 않은 요청입니다.\r\n\tat org.ll.apiresponse.service.ExampleService.getErrorResponse(ExampleService.java:16)...",
        "message": "올바르지 않은 요청입니다.",
        "path": "/api/error"
    }
}
```

Spring 기본 예외 처리기가 생성한 응답을 `data`에 담아 전송한다. 그러나 의도와 다르게 `success`가 `true`로 설정되었다. `data` 필드 또한 보다 친화적인 형태로 바꿀 필요가 있어 보인다.

---

```java
@JsonInclude(JsonInclude.Include.NON_NULL)  
public record ErrorResponse(  
        String code,  
        String message,  
        int status,  
        List<FieldError> errors  
) {  
  
    public record FieldError(String field, String message) {}  
  
    public static ErrorResponse of(String code, String message, int status) {  
        return new ErrorResponse(code, message, status, null);  
    }  
}
```

먼저 예외에 대한 공통 응답 객체 `ErrorResponse`를 생성한다.

`code`는 앞에서 언급한 `U001`, `U002`와 같이 개발자가 따로 정의한 에러 식별 코드, `message` 역시 정의한 에러 메시지이다. `status`는 HTTP 상태 코드, `error`는 `@Valid` 에러 발생 시 구체적으로 어떤 필드에서 어떤 문제가 발생했는지 내역을 담는 리스트이다. 이번 예제에서는 다루지 않는다.

`of` 메서드는 이후 핸들러에서 `ErrorResponse` 객체를 생성할 때 사용된다.

---

```java
@JsonInclude(JsonInclude.Include.NON_NULL)  
public record ApiResponse<T>(  
        boolean success,  
        T data,  
        ErrorResponse error  
) {  
  
    public static <T> ApiResponse<T> of(T data) {  
        return new ApiResponse<>(true, data, null);  
    }  
  
    public static ApiResponse<Void> empty() {  
        return new ApiResponse<>(true, null, null);  
    }  
  
    public static ApiResponse<Void> error(ErrorResponse error) {  
        return new ApiResponse<>(false, null, error);  
    }  
}
```

```java
@RestControllerAdvice  
public class GlobalExceptionHandler {  
  
    @ExceptionHandler(IllegalArgumentException.class)  
    public ResponseEntity<ApiResponse<Void>> handleIllegalArgumentException(IllegalArgumentException ex) {  
        ErrorResponse error = ErrorResponse.of(  
                "BAD_REQUEST",  
                ex.getMessage(),  
                HttpStatus.BAD_REQUEST.value()  
        );  
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)  
                .body(ApiResponse.error(error));  
    }  
}
```

`IllegalArgumentException` 예외를 잡아 `ErrorResponse` 로 변환한 후, 다시 `ApiResponse`로 변환하는 핸들러를 생성한다.

`@RestControllerAdvice`를 통해 전역 예외 핸들러임을 선언한다.

---

```json
// http://localhost:8080/api/error
{
    "success": false,
    "error": {
        "code": "BAD_REQUEST",
        "message": "올바르지 않은 요청입니다.",
        "status": 400
    }
}
```

이후 `/error`에 접근하면, `ApiResponse` 형식에 맞게, 에러 또한 `ErrorResponse` 형태를 가지도록 응답이 리턴된다.