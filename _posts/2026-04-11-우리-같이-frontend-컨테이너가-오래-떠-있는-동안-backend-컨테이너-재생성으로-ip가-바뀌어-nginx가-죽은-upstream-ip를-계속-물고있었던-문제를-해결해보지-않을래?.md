---
share: true
categories:
  - Troubleshooting/TIL
title: 우리 같이 frontend 컨테이너가 오래 떠 있는 동안 backend 컨테이너 재생성으로 IP가 바뀌어 Nginx가 죽은 upstream IP를 계속 물고있었던 문제를 해결해보지 않을래?
date: 2026-04-11 22:01:24 +0900
slug: 2026-04-11-우리-같이-frontend-컨테이너가-오래-떠-있는-동안-backend-컨테이너-재생성으로-ip가-바뀌어-nginx가-죽은-upstream-ip를-계속-물고있었던-문제를-해결해보지-않을래?
tags:
  - Nginx
  - Docker
description: 우리 같이 frontend 컨테이너가 오래 떠 있 .. (더보기)
---

![IMG_6120.jpg](../assets/img/IMG_6120.jpg)

## 배경

프론트엔드 서버를 Vercel 대신 Nginx를 통해 직접 서버로 배포했다.

```nginx
server {
    listen 80;
    
    # ...

    location / {
        # ...
    }

    location /api/ {
        proxy_pass http://backend:8080;
        
        # ...
    }
}

```

사용자가 80번 포트에 접근하면 라우팅이 이루어지는데, 정적 경로에 접근하면 프론트엔드 빌드 산출물을, `/api`에 접근하면 백엔드 서비스로 프록시를 전달한다.

```yaml
services:
  backend:
    # ...
```

`backend`라는 서비스명으로 통신이 가능한 이유는, Docker Compose가 기본 네트워크를 생성하고 각 서비스 이름을 내부 DNS 이름으로 등록하기 때문이다.

## 문제

![Pasted image 20260411185824.png](../assets/img/Pasted%20image%2020260411185824.png)

프로덕션에서 회원가입 요청이 실패하는 문제가 발생했다. 브라우저 콘솔에는 API 요청이 `502 Bad Gateway`로 표시되었다.

```bash
curl -i http://{서버 주소}/actuator/health

# HTTP/1.1 200 
```

같은 시점에 백엔드 서버 헬스체크 API는 정상 응답을 반환했기에, 백엔드 프로세스 자체의 다운과 관련이 없을 것이라고 판단했다.

따라서 애플리케이션 로직 문제가 아니라 프록시 계층 문제일 가능성이 높다고 판단했다. 최근에 백엔드 컨테이너만 별도로 배포를 진행했었기에, 이와 관련된 문제일 것이라 추측했다.

## 원인 분석

```bash
docker logs -f quiz-frontend-1 --tail 200

# 2026/04/11 05:29:36 [error] 29#29: *602 connect() failed (111: Connection refused) while connecting to upstream, client: XXX.XXX.XXX.XXX, server: _, request: "POST /api/auth/register HTTP/1.1", upstream: "http://172.18.0.5:8080/api/auth/register", host: "XXX.XXX.XXX.XXX", referrer: "http://XXX.XXX.XXX.XXX/"
```

프론트엔드 컨테이너의 로그를 살펴본 결과, Nginx가 특정 컨테이너 IP를 upstream으로 고정하고 있었으며, 해당 IP가 더 이상 유효하지 않음을 확인할 수 있었다.

문제를 정리하면 다음과 같다.

1. 프론트엔드 서버를 배포한 시점에, 당시 백엔드 컨테이너의 IP 주소로 upstream이 설정되었다.
2. 이후 백엔드 서버만 별도로 재배포되었고, 변경된 컨테이너 IP가 Nginx에 반영되지 않았다.
3. 프론트엔드 서버는 캐시된 upstream으로 요청을 보냈고, 502 에러가 발생했다.

## 해결

프론트엔드 컨테이너를 재시작하여 Nginx upstream을 올바르게 재해석할 수 있도록 초기화했다. 이후 정상적으로 응답을 받을 수 있었다.

```nginx
server {
    listen 80;
    
    # ...
    
    resolver 127.0.0.11 valid=30s ipv6=off;

    location / {
        # ...
    }

    location /api/ {
        set $backend_upstream http://backend:8080;
        proxy_pass $backend_upstream;
        
        # ...
    }
}

```

추가로 Nginx 설정을 Docker DNS 재해석 방식으로 변경했다. Docker 내장 DNS resolver를 명시적으로 선언하고, `proxy_pass`를 변수 기반으로 구성하여 요청 시점에 DNS TTL이 만료되어 있다면 이름을 재해석하도록 했다.