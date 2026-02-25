---
share: true
categories:
  - Troubleshooting/TIL
title: GitHub Flow와 Git Flow, 그 너머로
date: 2026-02-25 16:43:23 +0900
slug: 2026-02-25-github-flow와-git-flow,-그-너머로
tags:
  - git
  - 깃
  - 깃허브
  - github
description: GitHub Flow와 Git Flow, 그 너머로
---

![18b3e1d85fd4b421a.png](../assets/img/18b3e1d85fd4b421a.png)

Solid Connection 에서는 초기에 Github Flow 브랜치 전략을 채택했다가 [Git Flow 브랜치 전략으로 변경](https://github.com/solid-connection/solid-connect-server/wiki/%EB%B8%8C%EB%9E%9C%EC%B9%98-%EC%A0%84%EB%9E%B5-%EB%B3%80%EA%B2%BD%EA%B8%B01%E2%80%90Git-Flow%EB%A1%9C-%EB%B3%80%EA%B2%BD)하였고, [Git Flow를 변형한 전략](https://github.com/solid-connection/solid-connect-server/wiki/%EB%B8%8C%EB%9E%9C%EC%B9%98-%EC%A0%84%EB%9E%B5-%EB%B3%80%EA%B2%BD%EA%B8%B02%E2%80%90%EC%9A%B0%EB%A6%AC-%ED%8C%80%EB%A7%8C%EC%9D%98-%EC%A0%84%EB%9E%B5%EC%9C%BC%EB%A1%9C)을 사용했다.

![Pasted image 20260225164148.png](../assets/img/Pasted%20image%2020260225164148.png)

구체적인 브랜치 전략은 다음과 같다.

1. develop 브랜치를 기준으로 feature 브랜치를 생성하여 작업을 진행한다.
2. 작업이 완료된 feature 브랜치는 develop 브랜치에 머지된다.
3. develop 브랜치에 머지가 발생하면 CD 스크립트에 의해 stage 서버에 개발 내용이 반영된다.
4. 구현하고자 하는 기능이 완전히 구현되면 develop에서 release 브랜치로 PR을 생성 및 머지한다.
5. 이상이 없다면 release를 발행한다. 이 시점에 CD 스크립트에 의해 prod 서버에 개발 내용이 반영된다.
6. 이상이 없다면 release에서 master 브랜치로 PR을 생성 및 머지한다.

만약 hotfix를 해야 하는 상황이 발생한다면, 다음과 같은 순서로 진행한다.

1. master 브랜치에서 hotfix 브랜치를 생성한다.
2. 버그를 수정한 후 master 브랜치에 PR을 생성 및 머지한다.
3. 이후 커밋 동기화를 위해 master에서 release, develop으로 PR을 생성 및 머지한다.

---

이러한 전략을 취하면서 느낀 불편함은 다음과 같다.

1. release 브랜치의 존재 의미가 모호하다. 단순히 develop 브랜치와 master 브랜치의 중간 다리 역할을 할 뿐, 그 이상의 의미는 없다. release 브랜치를 없애고 곧장 master에 머지한 후, release를 발행해도 된다.
2. 커밋이 아니라 기능 단위로 prod 서버로의 배포를 진행하기에, 급하게 필요한 기능만 prod에 반영하기 어렵다. 해당 기능만 cherry pick하여 반영하면 좋지만, 대부분의 경우 다른 커밋과 의존적이기 때문에 연관 커밋까지 cherry pick해야 한다.
3. hotfix를 해야 하는 상황에서 롤백하기 어렵다. stage와 prod 서버에서 롤백용 이미지를 관리하고 있었으나, 배포 주기가 길었기 때문에 DB 스키마가 다를 확률이 높고, 롤백하더라도 스키마 불일치가 발생할 가능성이 높았다.

![Pasted image 20260225164157.png](../assets/img/Pasted%20image%2020260225164157.png)

느낀 불편함을 해소하는 전략을 팀원께서 제안해주셔서 정리해보려고 한다. 먼저 각 브랜치의 역할에 대해 알아보자.

master: 완벽히 동작하는 코드들을 관리하는 아카이브 브랜치. 최신 커밋을 기준으로 롤백용 이미지를 하나 관리한다.

feature: 기능을 개발하는 브랜치

develop: stage 서버의 기준이 되는 브랜치

release: prod 서버의 기준이 되는 브랜치

동작 흐름은 다음과 같다.

1. master 브랜치에서 feature 브랜치를 생성하여 기능을 구현한다.
2. 구현한 기능을 develop에 머지하여 stage 서버에 반영한다.
3. stage 서버에서 정상 동작을 확인했다면 feature에서 release로의 PR을 생성 및 머지한다.
4. prod 서버에서 정상 동작을 확인했다면 release에서 master로의 PR을 생성 및 머지한다.

만약 hotfix를 해야 하는 상황이 발생한다면, 다음과 같은 순서로 진행한다.

1. master의 이미지로 롤백한다.
2. release에서 hotfix 브랜치를 생성하고 수정 후 머지한다.
3. 정상 동작을 확인 후, master 및 develop 브랜치에 머지한다.
4. develop에서 release로 머지할 때 여러 기능이 한 번에 반영되기 때문에, prod 서버에서 문제가 발생했을 때 어떤 기능이 원인인지 파악하기 어렵다.

---

기존 불편함이 어떻게 해소되었는지 살펴보자.

> release 브랜치의 존재 의미가 모호하다. 단순히 develop 브랜치와 master 브랜치의 중간 다리 역할을 할 뿐, 그 이상의 의미는 없다. release 브랜치를 없애고 곧장 master에 머지한 후, release를 발행해도 된다.

release 브랜치가 prod 서버의 기준이 된다는 명확한 의미를 가지게 되었다.

> 커밋이 아니라 기능 단위로 prod 서버로의 배포를 진행하기에, 급하게 필요한 기능만 prod에 반영하기 어렵다. 해당 기능만 cherry pick하여 반영하면 좋지만, 대부분의 경우 다른 커밋과 의존적이기 때문에 연관 커밋까지 cherry pick해야 한다.

feature에서 release로 바로 PR을 생성하기에 원하는 feature만 선택적으로 prod 서버에 배포할 수 있다.

> hotfix를 해야 하는 상황에서 롤백하기 어렵다. stage와 prod 서버에서 롤백용 이미지를 관리하고 있었으나, 배포 주기가 길었기 때문에 DB 스키마가 다를 확률이 높고, 롤백하더라도 스키마 불일치가 발생할 가능성이 높았다.

새로운 전략은 더 작은 단위로 자주 배포할 수 있게 되므로 master와 release 간 이미지 차이가 작아지고, 스키마 불일치 가능성이 크게 줄어든다.

> develop에서 release로 머지할 때 여러 기능이 한 번에 반영되기 때문에, prod 서버에서 문제가 발생했을 때 어떤 기능이 원인인지 파악하기 어렵다.

새로운 전략에서는 feature 브랜치에서 release로 개별적으로 PR을 생성하므로, 문제가 발생했을 때 어떤 feature가 원인인지 특정하기 쉽다.