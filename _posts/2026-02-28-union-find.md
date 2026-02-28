---
share: true
categories:
  - PS
title: Union Find
date: 2026-02-28 16:41:16 +0900
slug: 2026-02-28-union-find
tags:
  - UnionFind
description: Union Find
---

`Union Find (Disjoint Set Union, DSU)`는 **서로소 집합들을 관리하는 자료구조**로, 두 원소가 같은 집합에 속하는지 판별하고 두 집합을 합치는 연산을 효율적으로 수행한다.

> 서로소 집합(Disjoint Set)이란 공통 원소가 없는 두 집합, 즉 **교집합이 공집합**인 집합들을 말한다.

## 핵심 연산

- `Find(x)`: 원소 $x$가 속한 집합의 **대표 원소**(루트)를 리턴한다.
- `Union(a, b)`: 원소 $a$가 속한 집합과 원소 $b$가 속한 집합을 **하나로 합친다.**

## 일반적인 구현

각 원소의 부모를 저장하는 배열 `parent`를 사용한다. 초기에는 **모든 원소가 자기 자신을 부모**로 가진다. 즉, **각 원소가 독립적인 집합을 이루는 상태**이다.

```cpp
int parent[10001];

void init(int n) {
    for (int i = 1; i <= n; i++) {
        parent[i] = i;
    }
}

int Find(int x) {
    if (parent[x] == x) return x;
    return Find(parent[x]);
}

void Union(int a, int b) {
    a = Find(a);
    b = Find(b);
    if (a != b) parent[a] = b;
}
```

---

이 구현은 직관적이지만, 트리가 한쪽으로 치우쳐 **편향 트리**가 될 수 있다. 이 경우 Find 연산이 $O(N)$까지 걸릴 수 있어, 전체 $M$번의 연산에 대해 $O(NM)$의 시간복잡도를 가진다.

## 경로 압축

Find 연산 시 거쳐간 모든 노드의 부모를 **직접 루트로 갱신**하는 최적화 기법이다. 이후 동일한 노드에 대해 Find를 호출하면, 한 번에 루트에 도달할 수 있다.

```cpp
int Find(int x) {
    if (parent[x] == x) return x;
    return parent[x] = Find(parent[x]);
}
```

---

경로 압축만 적용한 경우, $M$번의 연산에 대해 amortized $O(M \log N)$의 시간복잡도를 가진다.

## Union by Rank

Union 연산 시 트리의 **높이**가 낮은 쪽을 높은 쪽 아래에 붙여, 트리의 높이 증가를 최소화하는 기법이다.

```cpp
int parent[10001];
int rnk[10001];

void init(int n) {
    for (int i = 1; i <= n; i++) {
        parent[i] = i;
        rnk[i] = 0;
    }
}

int Find(int x) {
    if (parent[x] == x) return x;
    return Find(parent[x]);
}

void Union(int a, int b) {
    a = Find(a);
    b = Find(b);
    if (a == b) return;

    // rank가 낮은 쪽을 높은 쪽 아래에 붙인다.
    if (rnk[a] < rnk[b]) swap(a, b);
    parent[b] = a;

    // rank가 같은 경우에만 높이가 증가한다.
    if (rnk[a] == rnk[b]) rnk[a]++;
}
```

rank가 다른 경우 rank가 큰 쪽에 트리가 붙게 되므로 전체 트리의 rank는 변하지 않는다. 단, rank가 같은 경우 필연적으로 전체 트리의 깊이가 1 증가하기 때문에 rank를 1 증가시켜야 한다.

---

Union by Rank만 적용한 경우, 트리의 높이가 $O(\log N)$으로 제한되므로, $M$번의 연산에 대해 $O(M \log N)$의 시간복잡도를 가진다.

## 경로 압축 + Union by Rank

두 최적화를 동시에 적용하면, 사실상 상수 시간에 근사하는 성능을 얻을 수 있다.

```cpp
int parent[10001];
int rnk[10001];

void init(int n) {
    for (int i = 1; i <= n; i++) {
        parent[i] = i;
        rnk[i] = 0;
    }
}

int Find(int x) {
    if (parent[x] == x) return x;
    return parent[x] = Find(parent[x]);
}

void Union(int a, int b) {
    a = Find(a);
    b = Find(b);
    if (a == b) return;

    if (rnk[a] < rnk[b]) swap(a, b);
    parent[b] = a;
    if (rnk[a] == rnk[b]) rnk[a]++;
}
```

---

경로 압축과 Union by Rank를 함께 적용하면, $M$번의 연산에 대해 amortized $O(M \cdot \alpha(N))$의 시간복잡도를 가진다. 여기서 $\alpha(N)$은 **아커만 함수의 역함수**로, 현실적인 모든 입력 범위에서 $4$ 이하의 값을 가진다. 따라서 사실상 $O(M)$, 즉 연산당 **상수 시간**으로 간주할 수 있다.

## 언제 무엇을 사용해야 하는가?

일반적인 코테나 PS같은 경우는 경로 압축만 사용해도 대부분의 문제를 해결할 수 있다. 성능을 높이기 위해서는 경로 압축에 Union By Rank를 적용하면 된다. 롤백 시 `parent`와 `rnk` 를 롤백하면 된다.

단, **Union 연산을 롤백하는 경우가 있다면 Union By Rank만 적용**해야 한다.

## 시간복잡도 정리

| 기법                    | Find                     | $M$번 연산                             |
| --------------------- | ------------------------ | ----------------------------------- |
| Naive                 | $O(N)$                   | $O(NM)$                             |
| 경로 압축                 | amortized $O(\log N)$    | $O(M \log N)$                       |
| Union by Rank         | $O(\log N)$              | $O(M \log N)$                       |
| 경로 압축 + Union by Rank | amortized $O(\alpha(N))$ | $O(M \cdot \alpha(N)) \approx O(M)$ |