---
title : "[Spring Boot] JPA와 영속성 컨텍스트"
date : 2025-03-27 13:00:00 +0900
categories : [Spring Boot]
tags : [java, spring boot, jpa, persistence context, 영속성 컨텍스트, 스프링 부트]
---

## 📌 JPA란?

`JPA(Java Persistence API)` 는 자바 애플리케이션에서 RDBMS를 사용하기 위한 **표준 ORM 기술**이다. JPA는 **인터페이스**로, 이를 구현한 구현체를 사용해야 한다.

JPA의 구현체는 **Hibernate, EclipseLink, OpenJPA, DataNucleus** 등이 있다.

![image.png](attachment:07cff93b-372b-46bf-b47a-1bd730dc7204:image.png)

전체적인 데이터 흐름은 아래와 같다.

1. 자바 애플리케이션은 데이터 관련 작업을 할 때 JPA를 호출한다. 
2. JPA는 객체 지향적 코드를 SQL 쿼리로 변환한 후 JDBC API를 통해 전달한다. 
3. JDBC API는 적절한 JDBC 드라이버를 통해 쿼리를 DB에 전송한다. 
4. DB는 쿼리를 실행하고 결과를 JDBC 드라이버에 전달한다. 
5. 드라이버는 다시 결과를 JDBC API로 전달하며, JPA는 받은 결과를 자바 객체로 변환하여 애플리케이션에 전달한다.

## 📌 ORM

![image.png](attachment:80aa7676-9d54-4542-9024-a2ef986a3919:image.png)

- **JPA는 ORM의 일종**이다. `ORM(Object-Relational Mapping)` 은 객체 지향 프로그래밍과 RDBMS 간 데이터 변환 및 매핑하는 도구이다. ORM을 통해 SQL 쿼리를 작성하지 않고 **객체 지향적 코드를 통해 DB 작업을 수행**할 수 있다.

### 장점

- **메서드를 호출**하여 데이터를 조작할 수 있으므로 개발 속도를 향상시킬 수 있다. 더 나아가 애플리케이션의 비즈니스 로직에 집중할 수 있다.
- **객체 지향적**인 코드 작성으로 생산성이 증가하며, 코드 재활용이 가능하다.
    - 데이터베이스가 교체되어도 코드를 수정할 필요가 없다. 즉, DBMS에 대해 종속적이지 않다.

### 단점

- 복잡한 쿼리를 표현하는 데 적합하지 않을 수 있다.
- OOP와 RDBMS의 이해가 필요하기 때문에 다소 높은 러닝 커브를 가진다.

## 📌 JPA vs. MyBatis

JPA와 MyBatis 모두 자바 애플리케이션과 DB 간 상호작용에서 사용되는 프레임워크이나, 가장 큰 차이점은 **JPA는 ORM, MyBatis는 SQL Mapper**라는 것이다.

`SQL Mapper`는 DB의 SQL 쿼리와 그 결과를 객체로 매핑하는 역할을 한다. 비교적 사용이 편리하나, DBMS에 따라 SQL 문법이 달라질 수 있으며, SQL을 직접 작성해야 한다는 단점이 존재한다.

## 📌 영속성 컨텍스트

`Persistence Context`(영속성 컨텍스트)는 엔티티를 영구적으로 저장하는 환경을 말한다. 자바 애플리케이션과 DB 사이에 `Entity`를 저장하는 가상의 DB라고 생각하면 된다. 영속성 컨텍스트에 접근 및 관리를 하는 주체는 `EntityManager`이다.

### Entity

DB의 테이블을 클래스로 구현한 것이다.

### EntityManager

Entity를 관리하는 주체이다.

EntityManager를 생성하는 방법은 두 가지가 존재한다.

1. Container-Managed

스프링 컨테이너의 `EntityManagerFactory` 에서 EntityManager를 생성하여 주입하는 방식이다. Thread-safe하다.

```java
@PersistenceContext
EntityManager entityManager;
```

1. Application-Managed

애플리케이션에서 직접 EntityManager를 생성하여 사용하는 방식이다.

```java
@Autowired
private EntityManagerFactory emf;

public void someMethod() {
    EntityManager em = emf.createEntityManager();
    // ...
    em.close();
}
```

### 메서드

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("persistence-name");
EntityManager em = emf.createEntityManager();
em.persist(member);
```

`persist()` 를 통해 엔티티를 영속성 컨텍스트에 저장할 수 있다.

```java
Member findMember = em.find(Member.class, "member1");
```

`find()` 를 통해 영속성 컨텍스트에서 PK를 기준으로 엔티티를 조회할 수 있다.

```java
em.flush();
```

`flush()` 를 통해 영속성 컨텍스트의 변경 사항을 DB에 반영할 수 있다.

### flush

flush하는 방법은 세 가지가 존재한다.

1. `flush()`를 통해 직접 flush
2. 트랜잭션 커밋 시 자동으로 flush가 호출된다.
3. JPQL과 같은 객체 지향 쿼리 실행 시 자동으로 flush가 호출된다.

flush가 호출되면 아래 작업이 수행된다.

1. Dirty Chcking이 동작하여 영속성 컨텍스트에 존재하는 모든 엔티티를 스냅샷과 비교한다.
2. 수정된 엔티티에 대한 SQL을 생성하여 쓰기 지연 SQL 저장소에 삽입한다.
3. 쓰기 지연 SQL 저장소의 쿼리를 DB에 전송한다.

### Entity Lifecycle

![image.png](attachment:b6ed3bef-29fc-4409-b382-b80b3c3389f3:image.png)

엔티티는 영속성 컨텍스트와의 관계에 따라 크게 네 가지 상태를 가진다.

1. 비영속(New/Transient)

엔티티가 영속성 컨텍스트와 전혀 관계가 없는 상태이다.

```java
Member member = new Member();
member.setId("member1");
```

1. 영속(Managed)

엔티티가 영속성 컨텍스트에 저장되어 관리되는 상태이다.

```java
em.persist(member);
```

1. 준영속(Detached)

엔티티가 영속 상태였다가 영속성 컨텍스트에서 분리된 상태이다.

```java
em.detach(member);
em.clear();
em.close();
```

`detach()` 를 통해 엔티티를 영속성 컨텍스트에서 분리한다.

`clear()` 를 통해 영속성 컨텍스트를 초기화한다.

`close()` 를 통해 애플리케이션에서 관리하는 EntityManager를 닫는다.

`merge()` 를 통해 분리된 엔티티를 다시 영속성 컨텍스트에 병합할 수 있다.

1. 삭제(Removed)

영속성 컨텍스트에 저장된 엔티티가 삭제된 상태이다.

```java
em.remove(member);
```

`remove()` 를 통해 엔티티를 영속성 컨텍스트와 DB 모두 제거한다.

### 장점

- 영속성 컨텍스트는 내부에 **1차 캐시**를 가지고 있다. 1차 캐시를 통해 동일한 엔티티 접근 시 DB가 아닌 캐시에서 조회할 수 있다. 단, **1차 캐시는 트랜잭션 단위**이므로 트랜잭션이 끝나면 캐시의 해당 정보는 사라지게 된다.
    - 애플리케이션 전체가 공유하는 캐시는 2차 캐시라고 부른다.
- 같은 트랜잭션 안에서 같은 엔티티를 조회하면 동일한 객체 참조를 리턴한다. 이를 통해 **동일성**을 보장한다.

```java
Member a = em.find(Member.class, "member1"); // DB 조회
Member b = em.find(Member.class, "member1"); // 1차 캐시 조회
System.out.println(a == b); // true (동일성 보장)
```

- SQL 쿼리를 바로 실행하는 것이 아니라, **트랜잭션이 커밋되는 시점에 이들을 모아 한 번에 실행**한다. 이를 통해 DB 접근 횟수를 줄여 성능을 최적화할 수 있다.
- 엔티티의 변경 사항을 자동으로 감지하여 UPDATE 쿼리문을 생성한다. 이를 `Dirty Checking`이라고 한다.
- `Lazy Loading`(지연 로딩) 기능을 통해 필요한 시점에 관련 데이터를 불러올 수 있다.

## 📌 JPA 메서드

JPA는 메서드 이름에서 자동으로 쿼리를 생성하는 기능을 제공한다.

`findBy` 는 특정 속성을 기준으로 엔티티를 조회한다.

`existBy` 는 특정 속성을 기준으로 엔티티 존재 여부를 확인한다.

`deleteBy` 는 특정 속성을 기준으로 엔티티를 삭제한다.

`countBy` 는 특정 속성을 기준으로 엔티티 수를 계산한다.

```java
List<Book> findByTitle(String title);
List<Book> findByTitleOrAuthor(String title, String author);
```

`@Query` 어노테이션을 통해 JPQL 쿼리를 직접 작성할 수 있다.

```java
@Query("select u from User u where u.firstname = ?1 and u.emailAddress = ?#{principal.emailAddress}")
List<User> findByFirstnameAndCurrentUserWithCustomQuery(String firstname);
```

## 📌 참고

[https://velog.io/@modsiw/JPAJava-Persistence-API의-개념](https://velog.io/@modsiw/JPAJava-Persistence-API%EC%9D%98-%EA%B0%9C%EB%85%90)

https://ittrue.tistory.com/254

[https://velog.io/@neptunes032/JPA-영속성-컨텍스트란](https://velog.io/@neptunes032/JPA-%EC%98%81%EC%86%8D%EC%84%B1-%EC%BB%A8%ED%85%8D%EC%8A%A4%ED%8A%B8%EB%9E%80)