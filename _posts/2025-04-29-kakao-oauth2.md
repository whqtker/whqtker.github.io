---
title : "[React, Spring Boot] 카카오 로그인 구현"
date : 2025-04-29 17:50:00 +0900
categories : [Spring Boot]
tags : [spring boot, network, 스프링 부트, 네트워크, OAuth]
---

![image.png](assets/img/kakao_oauth2/1.png)

Spring Boot로 RESTful API를 사용한 OAuth 카카오 로그인을 구현하자. 

## 📌 개발 환경

JDK: 21

Java: 21

Spring Boot: 3.4.4

## 📌 Kakao Developer

https://developers.kakao.com/console/app

![image.png](assets/img/kakao_oauth2/2.png)

새로운 애플리케이션을 추가한다. ‘애플리케이션 추가하기’ 버튼을 클릭한다.

![image.png](assets/img/kakao_oauth2/3.png)

정보를 입력하고 ‘저장’ 버튼을 클릭한다.

![image.png](assets/img/kakao_oauth2/4.png)

플랫폼으로 이동하여 ‘Web 플랫폼 등록’ 버튼을 클릭한다.

![image.png](assets/img/kakao_oauth2/5.png)

백엔드 서버 주소를 입력하고 ‘저장’ 버튼을 클릭한다.

![image.png](assets/img/kakao_oauth2/6.png)

도메인 주소가 잘 지정되면 위와 같이 보이게 된다.

![image.png](assets/img/kakao_oauth2/7.png)

카카오 로그인 탭에서 상태를 ON으로 설정한다.

![image.png](assets/img/kakao_oauth2/8.png)

하단에 ‘Redirect URI 등록’을 클릭한다.

![image.png](assets/img/kakao_oauth2/9.png)

원하는 Redirect URI를 등록한다.

![image.png](assets/img/kakao_oauth2/10.png)

기본적으로 가져올 수 있는 정보는 닉네임과 프로필 사진이다. 두 값 모두 식별자로써 사용하기에는 어려움이 있다. 따라서 카카오계정을 추가로 가져오는 작업을 수행한다.

![image.png](assets/img/kakao_oauth2/11.png)

앱 권한 신청에서 ‘비즈 앱 전환’ 버튼을 클릭한다.

![image.png](assets/img/kakao_oauth2/12.png)

‘개인 개발자 비즈 앱 전환’ 버튼을 클릭한다.

![image.png](assets/img/kakao_oauth2/13.png)

이메일 필수 동의를 선택하고 ‘전환’ 버튼을 클릭한다.

![image.png](assets/img/kakao_oauth2/14.png)

동의항목에서 필요 항목들을 필수 동의로 설정한다.

## 📌 카카오 로그인 동작

![image.png](assets/img/kakao_oauth2/15.png)

OAuth2.0 동작에 맞게 코드를 살펴보자.

1. Resource Owner → Client: 로그인 요청

```tsx
// src/components/Home.tsx
const handleLogin = () => {    
  const encodedRedirectUrl = encodeURIComponent(`${frontUrl}/auth/kakao/callback`);
  const loginUrl = `${backUrl}${kakaoUrl}?redirectUrl=${encodedRedirectUrl}`;
  window.location.href = loginUrl;
};

// ...
<button
  onClick={handleLogin}
  className="w-full flex items-center justify-center gap-2 bg-[#FEE500] text-black py-3 rounded-lg hover:bg-[#FDD835] transition-colors"
>
  <IconKakaoLargeWide />
</button>
```

사용자가 로그인 화면에서 ‘로그인’ 버튼을 클릭한다. 버튼이 클릭되면 설정된 리다이렉트 URI `loginUrl` 로 이동한다.

1. Client → Authorization Server: 로그인 요청

Spring Security의 OAuth2 Client가 카카오 인증 서버로 인증 요청을 만든다.

```java
// CustomAuthorizationRequestResolver.java
private OAuth2AuthorizationRequest customizeAuthorizationRequest(OAuth2AuthorizationRequest authorizationRequest, HttpServletRequest request) {
    if (authorizationRequest == null || request == null) {
        return null;
    }

    // 쿼리 파라미터에서 redirectUrl 추출
    String redirectUrl = request.getParameter("redirectUrl");
    Map<String, Object> additionalParameters = new HashMap<>(authorizationRequest.getAdditionalParameters());

    // state에 반드시 값이 들어가도록 보장
    String stateValue = (redirectUrl != null && !redirectUrl.isEmpty())
        ? redirectUrl
        : "http://localhost:5173/";

    additionalParameters.put("state", stateValue);

    return OAuth2AuthorizationRequest.from(authorizationRequest)
            .additionalParameters(additionalParameters)
            .state(stateValue)
            .build();
}
```

`CustomAuthorizationRequestResolver` 가 동작하여 요청을 커스터마이징한다.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          kakao:
            client-id: <카카오 REST API 키>
            redirect-uri: "${custom.site.backUrl}/{action}/oauth2/code/{registrationId}"
            authorization-grant-type: authorization_code
            scope:
              - profile_nickname
              - profile_image
              - account_email
```

카카오에 전달되는 파라미터는 application.yml에 명시되어 있다. 이 파라미터들이 쿼리스트링에 포함되어 전송된다.

`redirect_uri` 는 카카오 디벨롭퍼 애플리케이션에서 설정한 uri를 사용해야 한다.

![image.png](assets/img/kakao_oauth2/16.png)

`client_id` 에는 ‘REST API 키’를 작성하면 된다.

1. Authorization Server → Resource Owner: 로그인 페이지 제공

![image.png](assets/img/kakao_oauth2/17.png)

해당 로그인 페이지는 카카오가 제공하며, 서비스 개발자 코드에는 해당 UI가 없다.

1. Resource Owner → Authorization Server: ID/PW 제공

사용자는 카카오가 제공한 로그인 페이지를 통해 아이디와 비밀번호를 입력한다. 

1. Authorization Server → Resource Owner: Authorization Code 발급

사용자 인증에 성공하면 Authorization Server는 사용자에게 Authorization Code를  발급한다.

1. Resource Owner → Client: Redirect URI로 리디렉션

카카오는 등록된 redirect_uri로 사용자를 리디렉션하며, 해당 URL의 쿼리 파라미터에 Authorization Code, state 등이 포함된다.

```java
// CustomOAuth2AuthenticationSuccessHandler.java
@SneakyThrows
@Override
public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
    // ...existing code...
    String redirectUrl = request.getParameter("state");
    response.sendRedirect(redirectUrl);
}
```

인증이 성공하면 redirectUrl로 사용자를 리디렉션한다.

```tsx
// src/components/auth/KakaoCallback.tsx
const [searchParams] = useSearchParams();
const code = searchParams.get("code"); // ← Authorization Code 추출

useEffect(() => {
  if (!code) {
    navigate("/");
    return;
  }
  // ...이후 code를 이용한 인증 처리...
}, [code, login, navigate]);
```

프론트엔드에서 콜백 URL에서 code 파라미터를 추출한다. 이후 해당 코드를 통해 인증 처리를 진행한다.

1. Client → Authorization Server: Access Token 요청

Spring Security를 사용하고 있으므로 별도의 코드 없이 인증 플로우가 동작한다.

1. Authorization Server → Client: Access Token 발급

클라이언트의 Access Token 요청이 유효하면 카카오는 Access Token을 리턴한다. Spring Security가 자동으로 처리하며, 내부적으로 JWT를 생성하여 쿠키에 저장하고 이후 요청에 사용한다.

```java
// Rq.java
public String makeAuthCookies(Member member) {
    String accessToken = memberService.genAccessToken(member);
    setCookie("apiKey", member.getApiKey());
    setCookie("accessToken", accessToken);
    return accessToken;
}
```

```java
// AuthTokenService.java
String genAccessToken(Member member) {
    // JWT 생성
    return Ut.jwt.toString(
        jwtSecretKey,
        accessTokenExpirationSeconds,
        Map.of(
            "id", member.getId(),
            "username", member.getUsername(),
            "nickname", member.getNickname(),
            "avatar", member.getAvatar()
        )
    );
}
```

```java
// CustomOAuth2AuthenticationSuccessHandler.java
@SneakyThrows
@Override
public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
    String username = ((SecurityUser) authentication.getPrincipal()).getUsername();
    Member actor = memberService.findByUsername(username)
        .orElseThrow(() -> new IllegalStateException("Member not found: " + username));
    rq.makeAuthCookies(actor); // JWT(Access Token) 쿠키 저장
    String redirectUrl = request.getParameter("state");
    response.sendRedirect(redirectUrl);
}
```

1. Client → Resource Owner: 로그인 성공 알림

사용자가 카카오 인증을 마치고 redirect_uri로 돌아오면 프론트엔드는 로그인 상태로 전환한다.

```tsx
// src/components/auth/KakaoCallback.tsx
useEffect(() => {
  if (!code) {
    navigate("/");
    return;
  }

  const fetchAuthData = async () => {
    try {
      const response = await axios.get(`${backUrl}/api/v1/members/me`, {
        withCredentials: true,
      });

      if (response.data) {
        login();         // 로그인 상태로 전환
        navigate("/");   // 홈 등으로 이동 (로그인 성공 알림)
      }
    } catch (error) {
      console.error("인증 실패:", error);
      navigate("/");
    }
  };

  fetchAuthData();
}, [code, login, navigate]);
```

1. Resource Owner → Client: 서비스 요청

사용자가 Client를 통해 리소스를 요청한다.

1. Client → Resource Server: Access Token으로 API 호출

특정 엔드포인트로 요청을 보낸다. 프론트엔드는 JWT Access Token이 담긴 쿠키와 함께 엔드포인트로 GET 요청을 보낸다.

```tsx
// src/contexts/AuthContext.tsx
const checkAuthStatus = useCallback(async (): Promise<boolean> => {
    try {
        const response = await axios.get(`${backUrl}/api/v1/members/me`, {
            withCredentials: true,
        });

        const isAuthenticated = response.data && response.data.statusCode === 200;
        setIsLoggedIn(isAuthenticated);

        if (isAuthenticated) {
            setUserData(response.data.data);
        } else {
            setUserData(null);
        }
        return isAuthenticated;
    } catch (error) {
        setIsLoggedIn(false);
        setUserData(null);
        return false;
    }
}, []);
```

LoginUser 어노테이션과 `LoginUserArgumentResolver` 가 accessToken 쿠키에서 JWT를 추출하고 토큰에서 사용자 정보를 복원하여 컨트롤러에 주입한다.

```java
// ApiV1MemberController.java
@GetMapping("/me")
public GlobalResponse<?> me(@LoginUser Member loginUser) {
    log.debug("Received /me request for user: {}", loginUser);

    if (loginUser == null) {
        log.debug("No authenticated user found");
        return GlobalResponse.error(ErrorCode.ACCESS_DENIED);
    }

    MemberInfoDto userInfo = memberService.me(loginUser);
    return GlobalResponse.success(userInfo);
}
```

```java
// LoginUserArgumentResolver.java
@Override
public Object resolveArgument(
    MethodParameter parameter, ModelAndViewContainer mavContainer,
    NativeWebRequest webRequest, WebDataBinderFactory binderFactory
) {
    // accessToken 쿠키에서 JWT 추출
    HttpServletRequest request = (HttpServletRequest) webRequest.getNativeRequest();
    String accessToken = null;
    if (request.getCookies() != null) {
        for (Cookie cookie : request.getCookies()) {
            if ("accessToken".equals(cookie.getName())) {
                accessToken = cookie.getValue();
                break;
            }
        }
    }

    if (accessToken == null) {
        log.debug("accessToken cookie not found");
        return null;
    }

    // JWT에서 사용자 정보 추출
    Member member = memberService.getMemberFromAccessToken(accessToken);
    if (member == null) {
        log.debug("JWT payload invalid or missing id");
        return null;
    }

    return member;
}
```

1. Resource Server → Client: Access Token 검증 및 서비스 제공 승인

`LoginUserArgumentResolver` 에서 JWT를 쿠키에서 추출하고 유효성을 검증한다.

```java
@Override
public Object resolveArgument(
    MethodParameter parameter, ModelAndViewContainer mavContainer,
    NativeWebRequest webRequest, WebDataBinderFactory binderFactory
) {
    // accessToken 쿠키에서 JWT 추출
    HttpServletRequest request = (HttpServletRequest) webRequest.getNativeRequest();
    String accessToken = null;
    if (request.getCookies() != null) {
        for (Cookie cookie : request.getCookies()) {
            if ("accessToken".equals(cookie.getName())) {
                accessToken = cookie.getValue();
                break;
            }
        }
    }

    if (accessToken == null) {
        log.debug("accessToken cookie not found");
        return null;
    }

    // JWT에서 사용자 정보 추출
    Member member = memberService.getMemberFromAccessToken(accessToken);
    if (member == null) {
        log.debug("JWT payload invalid or missing id");
        return null;
    }

    return member;
}
```

또한 JWT의 유효성, 만료, payload를 검증한다.

```java
Map<String, Object> payload(String accessToken) {
    Map<String, Object> parsedPayload = Ut.jwt.payload(jwtSecretKey, accessToken);

    if (parsedPayload == null) return null;

    long id = ((Number) parsedPayload.get("id")).longValue();
    String username = (String) parsedPayload.get("username");
    String nickname = (String) parsedPayload.get("nickname");
    String avatar = (String) parsedPayload.get("avatar");

    return Map.of(
        "id", id,
        "username", username,
        "nickname", nickname,
        "avatar", avatar
    );
}
```

```java
public static Map<String, Object> payload(String secret, String jwtStr) {
    SecretKey secretKey = Keys.hmacShaKeyFor(secret.getBytes());

    try {
        return (Map<String, Object>) Jwts.parser()
                .verifyWith(secretKey)
                .build()
                .parse(jwtStr)
                .getPayload();
    } catch (Exception e) {
        return null;
    }
}
```

1. Resource Server → Client: 리소스 제공

LoginUser 어노테이션으로 인증된 Member 객체가 컨트롤러에 주입된다. 

## 📌 깃허브

https://github.com/whqtker/practice_oauth2_kakao