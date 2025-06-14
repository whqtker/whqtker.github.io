---
title : "DB 트랜잭션의 종류"
date : 2025-03-22 12:00:00 +0900
categories : [Database]
tags : [트랜잭션, cs, transaction]
---

## 📌 트랜잭션

- 트랜잭션은 DBMS에서 발생하는 **하나의 논리적 작업 단위**이다. 하나의 작업에는 하나 이상의 쿼리문이 포함된다.
- 예를 들어 ‘인출’이라는 작업은 계좌에 잔액을 확인하고 일정 금액을 빼서 금액을 업데이트하는 것을 포함한다.

## 📌 트랜잭션의 연산

### Commit

- commit 연산이 수행되면 **트랜잭션 내에서 수행된 작업이 DB에 영구적으로 반영**된다. commit을 수행하면 해당 트랜잭션은 종료된다.

### Rollback

- 트랜잭션 작업이 실패하거나 중단되면 rollback 연산이 수행된다. rollback을 수행하면 **트랜잭션 내에서 수행한 모든 작업이 취소**되며 트랜잭션이 시작하기 이전의 상태로 되돌아간다.

## 📌 트랜잭션의 속성

- `ACID`: Atomic, Consistency, Isolation, Durability

### Atomic(원자성)

- 트랜잭션 내의 모든 작업이 **완전히 수행되거나 전혀 수행되지 않아야 한다**. 트랜잭션의 작업이 끝까지 성공적으로 수행되면 DB에 반영되고(commit), 그렇지 않으면 반영하지 않고 되돌려야 한다(rollback). 이 속성은 트랜잭션이 “**All or Nothing**”이라는 원칙을 만족하도록 보장한다.

### Consistency(일관성)

- 트랜잭션이 완료되면 DB는 **일관성 있는 상태를 유지**해야 한다. 즉, 트랜잭션 전후의 DB 규칙, 제약 조건 등은 변함이 없어야 한다.

### Isolation(독립성)

- 트랜잭션은 서로 영향을 줄 수 없다. **각각의 트랜잭션은 독립적으로 수행**되며 다른 트랜잭션의 중간 상태를 확인할 수 없다. 이 속성을 통해 데이터의 무결성을 유지할 수 있다.

### Durability(영속성)

- 트랜잭션이 성공적으로 완료되면 그 결과는 DB에 영구적으로 저장된다. 트랜잭션 수행 후 시스템 장애가 발생하더라도 DB는 **트랜잭션의 완료 상태 시점의 상태를 유지**해야 한다.

## 📌 트랜잭션 격리 수준

- 트랜잭션 격리 수준은 DB의 Consistency와 Isolation을 유지하면서 동시에 발생할 수 있는 데이터 충돌을 관리하는 데 도움을 준다.
- 격리 수준은 크게 4가지가 있으며, READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE이다. SERIALIZABLE에 가까울수록 가장 고립 정도가 높지만 속도가 상대적으로 느리다. READ UNCOMMITTED에 가까울수록 가장 고립 정도가 낮지만 속도가 상대적으로 빠르다.

### READ UNCOMMITTED

- 트랜잭션이 아직 commit되지 않은 변경 사항을 읽을 수 있는 격리 수준이다. 즉, **다른 트랜잭션이 수행 중인 데이터에 접근이 가능**하다. 이러한 현상을 `Dirty Read`라고 한다.
- tx A가 tx B의 데이터를 읽는 Dirty Read가 발생했다고 하자. 만약 tx B가 rollback된다면 tx A가 읽은 데이터는 무효가 되며, 일관성이 깨지게 된다.
    
    ![001.png](assets/img/transaction/001.png)
    
- 트랜잭션 간 잠금의 범위를 최소화하므로 DB의 동시성 및 성능이 높다는 장점이 있다. 그러나 데이터의 일관성이 보장되지 않는다.

### READ COMMITTED

- 트랜잭션이 **commit된 데이터만 읽을 수 있도록 허용**하는 격리 수준이다.
- tx A가 수행 중인데 tx B가 테이블의 데이터를 읽는 것을 시도한다고 하자. tx B는 어떤 데이터를 읽게 될까? tx A가 수행되기 전 **이전 테이블 상태를 Undo 영역에 잠시 저장**한다. 따라서 tx A에서 아직 commit되지 않는 데이터를 읽는 것이 아닌 Undo 영역의 데이터를 읽게 된다. 즉, Dirty Read가 발생하지 않는다.
    
    ![002.png](assets/img/transaction/002.png)
    
- 그러나 `Non Repeatable Read` 문제가 발생할 수 있다. tx A가 수행 중이고 tx B가 테이블에서 데이터를 읽는다면 Undo 영역에서 데이터를 읽을 것이다. 그러나 이후 tx A가 commit되고 tx B가 동일한 쿼리를 수행할 경우 tx A가 변경한 데이터를 읽게 된다. 즉, **읽은 데이터가 달라질 수 있다.**
    
    ![003.png](assets/img/transaction/003.png)
    

### REPEATABLE READ

- 트랜잭션이 시작된 후 동일한 데이터를 여러 번 읽을 수 있으며 **각 읽기 결과가 일관되게 유지되도록** 하는 격리 수준이다.
- 읽기 쿼리의 결과가 모두 동일하므로 Non-Repeatable Read 문제를 해결한 방법이다.
- 그러나 `Phantom Read` 문제가 발생할 수 있다. Phantom Read는 특정 **트랜잭션이 반복적으로 같은 쿼리를 실행할 때, 이전 결과와 다르게 새로운 행이 추가 또는 삭제되는 현상**이다. tx A가 특정 조건을 만족하는 데이터를 조회하는 쿼리를 연속해서 보낸다고 하자. tx B가 테이블의 행을 추가 또는 삭제하면 tx A 쿼리의 결과가 달라질 수 있다.
- Repeatable Read를 구현하는 방법은 `Read Lock`, `MVCC(Multi-Version Concurrency Control)`가 있다.

1. Read Lock
- 트랜잭션이 데이터를 읽을 때 해당 데이터에 대해 읽기 잠금을 설정한다. 읽기 잠금이 설정된 데이터는 다른 트랜잭션이 수정할 수 없다. 즉, 트랜잭션이 종료될 때까지 데이터의 상태를 보호한다.
- 예를 들어보자. tx A가 데이터를 읽을 때, 해당 데이터에 읽기 잠금을 설정한다. tx B는 해당 데이터에 수정을 시도하지만, 잠금이 걸려있어 대기한다. tx A가 **commit 또는 rollback되면 읽기 잠금은 해제**된다. 이제 tx B에 해당 데이터에 접근하여 수정할 수 있다.
- Read Lock으로 인해 동시성이 저하될 수 있으며, 성능이 떨어질 수 있다. **Phantom Read 문제는 주로 Read Lock을 사용하는 RDMS에서 자주 발생**한다. 새로운 행의 추가는 막을 수 없기 때문이다.
    
    ![B가 데이터 조회.png](assets/img/transaction/004.png)
    

1. MVCC
- DB에서 여러 버전의 데이터를 동시에 유지하는 방법이다. **각 트랜잭션이 특정 시점의 버전을 참조**하여 다른 트랜잭션의 변경에 영향을 받지 않는다.
    
    ![004-Photoroom.png](assets/img/transaction/005.png)
    
- 동시성을 높일 수 있으나 추가적인 메모리 사용이 필요하며, 복잡성이 증가할 수 있다.

### SERIALIZABLE

- Serializable은 DB의 모든 트랜잭션이 서로 독립적으로 실행하도록 보장하는 격리 수준이다. Dirty Read, Non Repeatable Read, Phantom Read를 모두 방지한다.
- 각 트랜잭션은 접근하고 있는 데이터에 읽기 또는 쓰기 잠금을 걸어 다른 트랜잭션이 접근하지 못하도록 한다.
- 무결성, 일관성, 신뢰성이 보장되나 성능이 보통 좋지 않으며 데드락이 발생할 수 있다.

### 정리

![TX.png](assets/img/transaction/006.png)

## 📌 트랜잭션 상태

![image.png](assets/img/transaction/007.png)

- 트랜잭션의 상태는 트랜잭션의 진행 상황을 나타내는 단계이다.

### Active

- 트랜잭션이 실행 중인 상태로 DB에 대한 작업을 수행 중이다. 작업이 완료되면 Partially Committed, 실패하면 Failed 상태가 된다.

### Partially Committed

- 트랜잭션의 모든 작업이 완료되었지만 commit되지 않은 상태이다. 즉, 트랜잭션의 결과가 DB에 반영되지 않은 상태이다. 성공적으로 DB에 반영되면 Committed. 실 패하면 Failed 상태가 된다.

### Failed

- 트랜잭션 실행 중 오류가 발생하여 실패한 상태이다. 이후 rollback이 수행되어 이전 상태로 되돌아간다.

### Committed

- 트랜잭션이 완료되고 DB에 commit된 상태이다. 모든 변경 사항이 DB에 영구적으로 반영된다.

### Aborted

- 실패한 트랜잭션이 rollback되어 모든 변경 사항이 취소된 상태이다.

## 📌 참고

https://mangkyu.tistory.com/299