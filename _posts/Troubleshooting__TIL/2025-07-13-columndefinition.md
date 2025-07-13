---
title : "columnDefinition은 조심해서 사용해야 한다"
date : 2025-07-13 19:20:00 +0900
categories : [Troubleshooting/TIL]
tags : [spring boot, columnDefinition, ddl, 스프링 부트]
---

![image.png](assets/img/til/3.png)

## 📌 @Column 어노테이션의 속성

엔티티의 속성을 정의할 때, `@Column` 어노테이션을 사용한다. `name` 속성을 통해 컬럼의 이름을 지정할 수 있고, `nullable` 속성을 통해 `NOT NULL` 제약 조건을 추가할 수 있다. `unique` 속성을 통해 `UNIQUE` 제약 조건을 추가할 수 있고, `length` 속성을 통해 컬럼의 길이를 지정할 수 있다.

`columnDefinition` 속성을 통해 컬럼에 대한 속성을 직접 DDL로 작성할 수 있다. 그렇다면, 다른 속성과 `columnDefinition` 속성이 매치가 되지 않을 경우 어떻게 될까?

```java
@Column(name = "username", nullable = false, length = 50, columnDefinition = "VARCHAR(100) NULL")
private String name;
```

예를 들어, 위 코드에서 과연 어떤 DDL이 생성될까? 각 속성이 다른 이야기를 하고 있다.

결론은 `columnDefinition` 에 명시된 내용으로 작성된다. **`columnDefinition` 속성이 다른 속성보다 우선순위가 더 높다.**

Hibernate와 같은 JPA 구현체가 DDL을 생성할 때의 동작에 그 이유가 있다. `nullable`, `unique` 와 같은 일반적인 속성은 DDL 구문을 만들기 위해 사용되는 반면, `columnDefinition` 은 어떠한 조합 없이 그대로 최종 DDL로 변환한다. 어떻게 보면 `columnDefinition` 속성이 더 구체적이고 직접적인 내용이 담겨 있으니 어쩌면 당연하다고 볼 수도 있겠다.

기대하지 않은 동작이 발생할 수 있기 때문에, 협업 전 충분한 합의를 통해 컨벤션을 맞추는 것이 좋을 것 같다. 개인적인 생각으로는, 특수한 경우에만 `columnDefinition` 을 사용하고, `nullable` 과 같이 `@Column` 의 속성으로 지원하는 경우에는 이를 사용하는 것이 좋을 것 같다.