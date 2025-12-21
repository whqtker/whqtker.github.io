---
share: true
categories:
  - Redis
title: Redis의 자료형과 명령어 - Hash
date: 2025-12-21 15:33:51 +0900
slug: 2025-12-21-redis의-자료형과-명령어---hash
tags:
  - redis
  - 레디스
description: Redis의 자료형과 명령어 - Hash
---

## Hash

Redis의 기본 구조는 key-value 구조이다. `Hash` 타입은 value에 또 다른 key-value 쌍의 집합이 들어간다. 해시 타입의 key를 field라고 부른다.

Java의 `HashMap`, Python의 `dict`과 유사하다.

하나의 hash key에 저장할 수 있는 field의 수는 $2^{32}-1$개로, 많은 양의 데이터를 보관할 수 있다.

### 인코딩 방식

`Set`과 `ZSet`과 마찬가지로 데이터 크기에 따라 인코딩 방식이 동적으로 변화한다.

#### listpack (ziplist)

field 개수가 적고 value의 크기가 작을 때 `listpack` 방식으로 인코딩한다(기본값 512개, 64바이트).

헤더에 전체 길이와 원소의 개수를 저장하고, field와 value를 교대로 저장한다. 각각의 엔트리는 본인의 데이터 크기를 기록하고 있다.

포인터가 없기 때문에 메모리 측면에서 효율적이며, 메모리 할당 시 오버헤드가 한 번만 발생한다.

`jemalloc` 등을 통해 메모리를 할당한 후, 다시 해제하기 위해 할당한 메모리의 크기를 헤더에 기록한다. `listpack`은 각 엔트리에 대해 메모리를 할당하는 방법이 아니라 전체 `Hash` 자료구조에 대해 딱 한 번 할당하므로 하나의 헤더만 생기게 된다.

또한 엔트리가 빈틈없이 붙어서 저장되기 때문에 단편화 현상이 발생하지 않는다.

단, 엔트리 검색 시 선형 탐색을 사용하므로 시간복잡도는 $O(N)$이다.

#### hashtable

기본적으로 `listpack`을 사용하다 임계값을 넘으면 `hashtable` 방식으로 변환된다. 한 번 변경되면 다시 `1istpack`으로 돌아가지 않는다.

메모리는 `listpack` 방식보다 더 많이 사용하지만 속도 측면에서 유리하다.

내부적으로 `dict` 구조체를 그대로 사용한다. 입력된 field를 해시 함수를 통해 해시값을 얻고, 연산을 통해 인덱스를 계산하여 어느 버킷에 데이터를 넣을지 결정한다.

해시 충돌이 발생하면 체이닝 방식을 통해 linked list로 같은 버킷에 배정된 데이터들을 연결한다.

`Hash` 자료형에서 엔트리 구조체(`dictEntry`)는 field를 가리키는 key 포인터, value를 가리키는 value 포인터, 다음 엔트리를 가리키는 next 포인터가 존재한다.

$O(1)$의 시간복잡도로 데이터를 조회할 수 있으며, 테이블이 꽉 차게 되면 테이블 크기를 2배로 늘리는 방식으로 리해싱한다.

### 명령어

#### HSET

`HSET key field value [field value ...]`

`HSET`은 `Hash` 자료형에 field와 value를 저장 및 수정하는 명령이다.

`key`는 해시의 키, `field`는 내부 속성의 이름, `value`는 해당 속성에 저장할 값이다.

한 번의 명령으로 여러 쌍의 엔트리를 동시에 저장할 수 있다.

리턴값은 새로 추가된 field의 개수이다. 실제로 저장 또는 수정된 field의 개수가 아니다. 즉, 0이 리턴되더라도 명령이 실패한 것이 아닐 수 있다.

시간복잡도는 $O(N)$으로, $N$은 field의 개수이다. 다만 실무에서는 `HSET` 명령으로 비교적 적은 수의 field를 저장하므로 $O(1)$에 가깝다.

먼저 `key`가 존재하는지 확인하고, 없으면 새로운 `Hash` 객체를 생성한다. `Hash` 객체가 존재하고 `listpack` 방식으로 인코딩된 경우, 만약 `listpack` 의 임계값을 넘으면 `hashtable`

로 변환 후 저장한다.

#### HGET

`HGET key field`

`HGET`은 `field`의 value를 조회하는 명령이다. field가 존재하면 value를 리턴하고, field가 없거나 key 자체가 없을 때 `nil`을 리턴한다.

`Hash` 자료형이므로 조회 시의 시간복잡도는 $O(1)$이다.

인코딩 방식에 따라 value를 찾는 방법이 조금 다르다.

`hashtable` 방식이면 field를 해시 함수에 넣어 계산된 인덱스로 버킷을 찾은 후, value를 찾는다.

`listpack` 방식이면 메모리 시작점부터 데이터를 훑어 field와 일치하는 key를 찾을 때까지 선형 탐색을 진행한다. 이론 상 $O(N)$이나, $N$의 크기가 작으므로 $O(1)$만큼 빠르다.

#### HDEL

`HDEL key field [field ...]`

`HDEL` 은 `field`를 지정하여 엔트리를 삭제하는 명령이다. 한 번의 명령으로 여러 개의 field를 동시에 삭제할 수 있다 시간복잡도는 $O(N)$이다.

실제로 삭제된 개수를 리턴한다.

`hashtable`로 인코딩된 경우 해시 함수로 버킷을 찾아 해당 엔트리를 메모리에서 해제하고 linked list를 끊는다.

`listpack` 으로 인코딩된 경우 선형 탐색으로 엔트리를 찾은 뒤 해당 엔트리를 삭제하고 뒤에 있는 엔트리들을 앞으로 당긴다. 이 과정에서 메모리 이동 비용이 발생한다.

`HDEL`로 field를 삭제한 후, `Hash` 에 남은 field가 없게 되면 해당 hash key를 메모리에서 삭제한다.

#### HEXISTS

`HEXISTS key field`

`HEXISTS`는 `Hash` 내에 `field`가 존재하는지 여부를 확인하는 명령이다. 존재하면 1, 그렇지 않거나 key 자체가 없으면 0을 리턴한다. 역시 시간복잡도는 $O(1)$이다.

`HGET`으로 확인해도 되는데 왜 굳이 `HEXISTS`를 사용할까? 만약 value의 크기가 아주 크다면 존재 여부를 알기 위해 크기가 큰 value를 로드해야 하므로 네트워크 낭비가 발생한다. `HEXISTS`는 오직 field가 존재하는지만 확인하므로 네트워크 비용이 거의 없다.

#### HMGET

`HMGET key field [field ...]`

`HMGET` 명령은 여러 `field`의 값을 한 번의 요청으로 조회하는 명령이다. value가 담긴 리스트를 리턴한다. 리스트 내의 value들은 순서가 보장된다. 만약 요청한 field가 존재하지 않으면 그 자리에 `nil`을 채워 리턴한다.

시간복잡도는 $O(N)$이나, 여러 번의 `HGET` 명령보다 훨씬 빠르다. RTT 비용이 줄어들기 때문이다.

#### HGETALL

`HGETALL key`

`HGETALL` 명령은 해시 `key`에 저장된 모든 field와 value를 전부 가져오는 명령이다.

JSON이나 Map 형태가 아니라 1차원 리스트 형태로 결과가 리턴된다. field와 value가 번갈아가며 등장한다.

시간복잡도는 $O(N)$이다. 따라서 field의 수가 많은 경우 사용에 유의해야 한다. 대안으로 `HSCAN`을 사용해야 하는데, 후술한다.

#### HINCRBY, HINCRBYFLOAT

`HINCRBY/HINCRBYFLOAT key field increment`

`HINCRBY`와 `HINCRBYFLOAT` 명령은 `field`값을 숫자로 간주하고 연산하는 명령이다. 기존의 value를 읽어 연산한 후 다시 저장하는 과정을 원자적으로 수행한다.

해당하는 `field`가 없다면 value를 0으로 간주하고 연산을 수행한다. 만약 value가 정수로 변환 불가능한 문자열이라면 에러를 리턴한다.

field를 찾는 작업은 $O(1)$, 연산 후 다시 저장하는 작업은 매우 빠르므로 전체 시간복잡도는 $O(1)$이다.

#### HKEYS, HVALS

```bash
HKEYS key
HVALS key
```

`HKEYS`와 `HVALS`는 저장된 데이터를 field 또는 value만 추출하는 명령이다. 저장된 모든 field 또는 value를 리스트 형태로 리턴한다.

field의 수가 $N$개일 때, 시간복잡도는 $O(N)$이다. 역시 데이터가 많은 경우 사용에 유의해야 하며, `HSCAN`을 사용하는 것이 권장된다.

`listpack`으로 인코딩된 경우 `[key, value, key, value, ...]` 형태로 저장되어 있으므로, `HKEYS`는 홀수 번째 원소만, `HVALS`는 짝수 번째 원소만 골라 리턴한다. 또한 입력 순서가 보장된다.

`hashtable` 형태로 인코딩된 경우 모든 버킷을 돌며 `HEKYS`는 엔트리의 `key` 포인터가 가리키는 값을, `HVALS`는 `value` 포인터가 가리키는 값을 리턴한다. 해시 함수의 결과에 따라 저장되므로 순서를 보장하지 않는다.

#### HLEN

`HLEN key`

`HLEN` 은 field의 총 개수를 리턴하는 명령이다.

`Hash` 자료형은 field의 수를 메타데이터로 관리하기 때문에 시간복잡도는 $O(1)$이다.

`hashtable` 로 인코딩된 경우 Redis의 `dict` 구조체의 `used` 변수의 값을 리턴한다.

`listpack`으로 인코딩된 경우 헤더에 저장된 총 원소의 개수를 리턴한다. 조금 정확히 표현하면 key-value 쌍이므로 2로 나누어 리턴한다.

#### HSCAN

`HSCAN key cursor [MATCH pattern] [COUNT count]`

`HSCAN`은 `Hash`에 저장된 모든 field와 value를 커서 기반으로 조회하는 명령이다.

`HGETALL`와 같은 $O(N)$으로 조회하는 명령의 경우 블로킹 문제가 발생할 수 있기 때문에 데이터의 수가 많은 경우 `HSCAN`을 사용하는 것이 좋다.

`SSCAN`, `ZSCAN`과 마찬가지로 `cursor`는 탐색을 시작할 인덱스이며, `MATCH` 옵션은 특정 패턴에 맞는 field를 필터링할 때, `COUNT` 옵션은 가져올 데이터의 힌트를 지정할 때 사용한다.

> `COUNT`는 가져올 데이터의 수와 일치하지 않는다. 검사할 버킷의 수를 의미한다.

리턴값은 다음 커서와 데이터가 담긴 리스트이다.

역시 순회 도중 다른 클라이언트가 데이터를 추가 및 삭제하여 해시 테이블 크기가 변경되면 데이터가 중복되어 조회될 수 있다.

### Hash vs. String

예시 상황을 통해 데이터 모델링 시 `String`과 `Hash` 중 어느 타입을 선택해야 할지 알아보자.

![Pasted image 20251219210245.png](../assets/img/Pasted%20image%2020251219210245.png)

> 게임 캐릭터의 정보를 Redis에 저장하려고 한다. 캐릭터는 `id`, `name`, `job`, `hp`, `gold` 속성을 가지고 있다. 캐릭터는 샤냥을 하며 `hp`와 `gold` 는 수시로 변경된다. 현 플레이어의 `id`는 `char:100` 이다.

#### String 타입으로 저장

```bash
SET char:100:name "타락파워전사"
SET char:100:job "Demon Slayer"
SET char:100:hp 5000
SET char:100:gold 10000
```

캐릭터의 속성을 `String` 타입으로 분산하여 저장하는 상황을 가정해보자.

샤낭 중 `hp`가 100이 깎였다. 이 동작은 `DECRBY char:100:hp 100`으로 반영할 수 있으며, 속도 또한 빠르다.

그러나 `타락파워전사`가 캐릭터를 삭제하려고 한다. 그렇다면 `id`가 `char:100`인 key들을 일일이 찾아 삭제해주어야 한다. 누락한 경우 더미 데이터가 된다. 또한 key 하나를 만들 때마다 Redis는 메터데이터를 붙이는데, 생성한 캐릭터의 수가 많을수록 메타데이터를 포함한 캐릭터의 정보가 차지하는 메모리의 크기는 매우 커지게 된다.

#### JSON으로 저장

```bash
SET char:100 '{"name":"타락파워전사", "job":"Demon Slayer", "hp":5000, "gold":10000}'
```

캐릭터 정보를 JSON 형태로 관리해보자. 이전에 발생한 캐릭터 삭제 문제는 간단히 해결할 수 있다.

그러나 `hp`가 100 감소한 경우 어떤 동작을 수행해야 할까? 먼저 `GET char:100`으로 전체 JSON을 가져와야 하고, JSON을 파싱해서 `hp`를 4900으로 수정 후 `SET` 명령으로 다시 저장해야 한다. 과도한 RTT 비용이 발생하게 된다.

또한 서버 A에서 `hp`를 수정하는 순간 다른 서버 B에서 `gold` 를 수정하면, B는 `hp`가 5000인 캐릭터의 정보를 읽게 되고, 서버 B가 수정한 내역으로 덮어쓰게 되면서 서버 A가 수정한 `hp` 감소 내역은 사라지게 된다.

#### Hash로 저장

```bash
HSET char:100 name "타락파워전사" job "Demon Slayer" hp 5000 gold 10000
```

캐릭터의 정보를 `Hash`로 관리해보자.

먼저 `hp`가 감소되면 `HINCRBY char:100 hp -100` 명령을 수행하면 된다. JSON 방식보다 네트워크 비용 측면에서 효율적이다.

동시성 문제는 어떨까? 서버 A가 `HINCRBY char:100 hp -100`, 서버 B가 `HINCRBY char:100 gold 50` 를 수행한다고 하자. Redis는 싱글 쓰레드로 동작하므로 데이터 충돌 없이 순서대로 수행하게 된다.

또한 `char:100`이라는 key 하나만 관리하면 되므로 메모리 측면에서도 효율적이다. 당연히 캐릭터 삭제 측면에서도 유리하다.

---

`Hash`는 데이터 객체는 크지만 세부 속성 변경은 조금씩, 자주 일어나는 경우 유리하다. (JSON vs. `Hash`)

또한 관련 데이터를 그룹핑하고 싶을 때 유리하다. 이 경우 메모리 측면에서도 효율적이다. (`String` vs. `Hash`)

---

과거 인스타그램 역시 이미지 ID와 사용자 ID 매핑 시 `String` 타입을 사용했으나, 메모리 부족으로 `Hash` 구조로 변경하였고, 메모리 사용량을 효과적으로 절감한 [사례](https://instagram-engineering.com/storing-hundreds-of-millions-of-simple-key-value-pairs-in-redis-1091ae80f74c)가 있다.

### 주의사항

`Hash`는 개별 field에 대해 TTL을 설정할 수 없다.

특정 field에 TTL을 설정하면 Redis는 만료 시각을 어딘가에 저장해야 한다. field가 많다면 각각의 TTL에 대한 메타데이터를 관리해야 하므로 메모리 오버헤드가 발생하게 된다. 또한 Redis가 매 순간마다 지워야 하는 field가 있는지 확인해야 하므로 CPU 사용량 또한 증가하게 된다.

---

이전에 제시한 게임 상황으로 돌아가자.

`HSET char:100 buff:attack "8"`

사용자가 '드레이크의 피'를 마셔 공격력이 5분간 8 증가했다고 하자. `Hash`로 버프 정보를 관리하는 경우 TTL을 설정할 수 없기 때문에 관리가 매우 어려워지게 된다.

```bash
SET buff:char:100:attack "8"
EXPIRE buff:char:100:attack 300
```

TTL을 설정해야 하는 field는 별도의 `String` key로 분리하여 관리하는 것이 좋다.