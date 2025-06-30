---
title : "[Java] Lambda, Stream"
date : 2025-06-30 12:35:00 +0900
categories : [Java]
tags : [java, lambda, 람다, stream, 스트림]
---

## 📌 Lambda Function

`Lambda` 함수는 함수를 하나의 식처럼 표현하는 방법이다. 함수명이 없기 때문에 ‘익명 함수’라고 부르기도 한다. 익명 함수는 함수형 인터페이스의 인스턴스로 취급되므로 ‘일급 객체’이다. 일급 객체란 다른 객체들에게 일반적으로 적용 가능한 연산을 모두 지원하는 객체로, 익명 함수 또한 정수나 문자열과 동일한 방법으로 다룰 수 있음을 의미한다.

```java
(매개변수 목록) -> { 함수 내부 }
```

위와 같은 형태를 람다 함수라고 한다.

```java
// 기존 함수
public int add(int x, int y) {
    return x + y;
}

// 람다 함수
(int x, int y) -> x + y

```

다음과 같이 람다 함수로 변환할 수 있다.

```java
(x, y) -> x + y
```

컴파일러가 매개변수 타입을 추론할 수 있는 경우 생략할 수 있다.

```java
x -> x * 2
```

매개변수가 하나라면 괄호를 생략할 수 있다. 단, 매개변수가 없다면 빈 괄호를 사용해야 한다.

```java
(x, y) -> x + y
```

함수 내부가 한 줄이라면 중괄호 및 `return` 을 생략해야 한다.

## 📌 Functional Interface

함수형 인터페이스는 오직 하나의 추상 메서드만을 갖는 인터페이스이다. `@FunctionalInterface` 어노테이션을 사용하여 선언한다. 이 어노테이션을 사용하면 컴파일러가 함수형 인터페이스가 되기 위한 조건을 만족하는지 확인한다.

함수형 인터페이스가 되기 위한 조건은 다음과 같다.

1. 단 하나의 추상 메서드를 가져야 한다.
2. `default` , `static` 메서드는 여러 개 존재해도 무방하다.

### Consumer<T>

`Consumer` 인터페이스는 하나의 인자를 받아 로직을 수행하고 아무것도 리턴하지 않는 연산을 정의할 때 사용한다.

```java
Consumer<String> printer = s -> System.out.println(s);
printer.accept("Hello, Lambda!");
```

`accept` 는 `Consumer` 인터페이스의 추상 메서드로, `T` 타입의 인자를 받아 로직을 수행한다. 리턴 타입은 `void` 이다.

### Supplier<T>

`Supplier` 인터페이스는 인자를 받지 않고 오직 값을 리턴하는 연산을 정의할 때 사용한다.

```java
Supplier<User> userSupplier = () -> new User("기본사용자", 0);
User defaultUser = userSupplier.get();
```

`get` 은 `Supplier` 의 추상 메서드로, `T` 타입의 객체를 리턴한다.

### Function<T, R>

`Function` 인터페이스는 하나의 인자를 받아 특정 타입의 결과를 리턴하는 작업을 정의할 때 사용한다.

```java
Function<Integer, Integer> square = x -> x * x;
System.out.println(square.apply(4));
```

`apply` 는 `Function` 의 추상 메서드로, `T` 타입의 인자를 받아 `R` 타입의 결과를 리턴한다.

### Predicate<T>

`Predicate` 인터페이스는 하나의 인자를 받아 `boolean` 값을 리턴하는 인터페이스이다.

```java
Predicate<String> isEmpty = s -> s.isEmpty();
System.out.println(isEmpty.test(""));  
```

`test` 는 `Predicate` 의 인터페이스로 조건에 맞으면 `true`, 그렇지 않으면 `false` 를 리턴한다.

## 📌 Stream

`Stream` API는 Java 8부터 도입되었으며, 컬렉션, 배열 등과 같은 데이터를 함수형 프로그래밍 스타일로 다룰 수 있도록 하는 도구이다. 

스트림 파이프라인은 크게 생성, 중간 연산, 최종 연산으로 나뉘는데, 중간 연산은 최종 연산이 호출될 때까지 실제로 연산되지 않는다. 이를 `Lazy Evaluation` 이라고 한다. 

또한 스트림은 멀티 쓰레드를 통한 병렬 연산이 가능하다. 연산을 수행하여도 원본은 변경되지 않으며 새로운 스트림 또는 결과가 리턴된다. 한 번 최종 연산을 수행한 스트림은 다시 사용할 수 없다.

```java
// 람다 함수
List<String> upper1 = names.stream()
    .map(s -> s.toUpperCase())
    .collect(Collectors.toList());

// 메서드 참조
List<String> upper2 = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

스트림에 람다 함수를 사용할 수 있지만, 메서드 참조를 사용하면 더 간결해지고 가독성이 좋아지는 경우가 있다.

## 📌 Stream 생성

```java
List<String> list = List.of("A", "B", "C");

Stream<String> sequential = list.stream();
Stream<String> parallel = list.parallelStream();
```

`Collection` 을 상속한 클래스는 `stream` 과 `parallelStream` 메서드를 통해 스트림 또는 병렬 스트림을 생성할 수 있다.

```java
String[] array = {"X", "Y", "Z"};
Stream<String> stream1 = Arrays.stream(array);
```

배열 또한 `stream` 메서드를 통해 스트림을 생성할 수 있다.

```java
Stream<String> s1 = Stream.of("A", "B", "C");
Stream<Integer> s2 = Stream.of(1, 2, 3, 4);
```

`Stream.of` 메서드를 통해 원소를 직접 지정하여 스트림을 생성할 수 있다.

```java
Stream<String> builtStream = Stream.<String>builder()
    .add("A")
    .add("B")
    .add("C")
    .build();
```

`Stream.builder` 메서드를 통해 빌더 패턴을 사용하여 스트림을 사용할 수 있다.

```java
Stream.iterate(0, n -> n + 1)
      .limit(10)
      .forEach(System.out::println);
      
Stream.iterate(0,           
               n -> n < 10, // 종료 조건
               n -> n + 1)
      .forEach(System.out::println);
```

`iterate` 메서드를 통해 스트림을 생성할 수 있다. `iterate` 는 초기값과 연산 함수를 입력받아 무한 스트림을 생성한다.

Java 9부터 종료 조건을 함께 넘겨 유한 스트림을 생성할 수 있다.

## 📌 중간 연산

### filter

```java
Stream<T> filter(Predicate<? super T> predicate)
```

`filter` 메서드는 스트림 내 원소를 주어진 조건에 맞는 원소만 필터링하여 새로운 스트림으로 리턴한다. `Predicate<T>` 를 인자로 받아 `test` 메서드를 통해 각 원소가 조건을 만족하는지 확인한다.

`filter` 를 여러 번 사용하는 경우 `filter(a).filter(b)` 대신 `filter(a.and(b))` 와 같이 사용하는 것이 성능적으로 도움이 된다. 필터링의 횟수가 줄어들기 때문이다.

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "David");

List<String> filteredNames= names.stream()
    .filter(name -> name.startsWith("C"))
    .collect(Collectors.toList());
// ["Charlie"]
```

### map

```java
<R> Stream<R> map(Function<? super T, ? extends R> mapper)
```

`map` 메서드는 스트림의 각 원소를 다른 형태로 변환하여 새로운 스트림을 생성하는 연산이다. `Function<T, R>` 를 인자로 받으며, `T` 는 원본 스트림 원소의 타입, `R` 은 리턴된 스트림의 원소 타입이다.

기존 값을 직접 변경하는 것이 아니라 새로운 값을 생성하는 것에 주의해야 한다.

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

List<Integer> squaredNumbers = numbers.stream()
    .map(n -> n * n)
    .collect(Collectors.toList());
// [1, 4, 9, 16, 25]
```

### flatMap

```java
<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper)
```

`map` 메서드는 각 원소를 일대일 매핑하여 결과가 `Stream<Stream<R>>` 이 되지만, `flatMap` 메서드는 각 원소를 `Stream`으로 매핑한 후 평탄화하여 `Stream<R>` 형태로 만든다. 따라서 `flatMap` 내부에서 반드시 `Stream` 을 리턴하도록 해야 한다.

```java
List<List<String>> characters = List.of(
    List.of("A","B"),
    List.of("C","D")
);

characters.stream()
      .flatMap(list -> list.stream())
      .forEach(System.out::println);
// A
// B
// C
// D
```

### sorted

```java
Stream<T> sorted();

// 커스텀 정렬
Stream<T> sorted(Comparator<? super T> comparator);
```

`sorted` 메서드는 스트림 내 원소를 정렬하여 새로운 스트림을 리턴한다. 스트림의 원소가 `Comparable<T>` 를 구현해야 한다. 또는 `Comparable<T>` 를 인자로 넘겨 커스텀 정렬을 수행할 수 있다.

```java
class User {
    String name;
    int age;
}

List<User> users = List.of(
    new User("Alice", 30),
    new User("Bob", 25),
    new User("Charlie", 28)
);

List<User> byAge = users.stream()
    .sorted(Comparator.comparing(User::getAge))
    .collect(Collectors.toList());
// Bob(25), Charlie(28), Alice(30)
```

### distinct

```java
Stream<T> distinct();
```

`distinct` 메서드는 스트림의 원소 중 동일한 값이 여러 번 등장하면 첫 번째 원소만 남기고 나머지는 필터링하여 생성된 새로운 스트림을 리턴한다. 내부적으로 `equals` 또는 `hashCode` 메서드를 통해 중복을 판단한다. 따라서 사용자 정의 객체에서 `distinct` 메서드를 사용하고 싶다면 `equals` 나 `hashCode` 메서드를 오버라이드해야 한다.

```java
List<Integer> list = List.of(1, 2, 2, 3, 1, 4);

List<Integer> distinctList = list.stream()
    .distinct()
    .collect(Collectors.toList());
// [1, 2, 3, 4]
```

### limit

```java
Stream<T> limit(long maxSize)
```

`limit` 메서드는 스트림의 원소의 개수를 최대 `maxSize` 로 제한하여 처음부터 지정된 개수만큼 이루어진 스트림을 리턴한다.

```java
List<Integer> evens = Stream.iterate(0, n -> n + 2)
    .limit(10)
    .collect(Collectors.toList());
// [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

```

### skip

```java
Stream<T> skip(long n)
```

`skip` 메서드는 처음 n개의 원소를 건너뛰고 나머지 원소만으로 이루어진 스트림을 리턴한다. 만약 스트림 원소의 수가 n보다 작으면 빈 스트림을 리턴한다.

```java
List<String> items = List.of("A", "B", "C", "D", "E", "F");

List<String> result = items.stream()
    .skip(2)
    .collect(Collectors.toList());
// [C, D, E, F]
```

### peak

```java
Stream<T> peek(Consumer<? super T> action)
```

`peak` 메서드는 특정 시점의 원소를 살펴보고 추가적인 동작을 수행한다. 주로 파이프라인 중간의 값을 로깅할 때 사용한다.

```java
Stream.of(1, 2, 3, 4, 5)
    .peek(i -> log.debug("값: {}", i))
    .filter(i -> i % 2 == 0)
    .forEach(System.out::println);
```

## 📌 최종 연산

### forEach

```java
void forEach(Consumer<? super T> action)
```

`forEach` 메서드는 스트림의 원소에 대해 지정된 동작을 수행하고 리턴값 없이 종료하는 역할을 한다. 

병렬 스트림을 사용하는 경우 원본 데이터의 순서를 보장하지 않는다. 병렬 스트림에서 순서 또한 보장하기 위해서 `forEachOrdered` 를 사용해야 한다.

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

numbers.stream()
       .forEach(System.out::println);
```

### collect

```java
<R, A> R collect(Collector<? super T, A, R> collector)
```

`collect` 메서드는 스트림 원소들을 컨테이너로 합치는 역할을 한다. `T` 는 스트림의 원소 타입, `A` 는 스트림 내부에서 값을 잠시 저장하는 임시 컨테이너, `R` 은 최종 결과의 타입이다. 보통 `Collectors` 에서 미리 정의된 팩토리 메서드를 사용한다.

> `A` 는 `Accumulator` (누산기)라고 부른다.
> 

```java
Map<Character, List<String>> grouped =
    Stream.of("apple", "banana", "avocado")
          .collect(Collectors.groupingBy(s -> s.charAt(0)));
// {'a': ["apple","avocado"], 'b': ["banana"]}
```

### reduce

`reduce` 메서드는 스트림의 원소들을 하나의 결과로 결합할 때 사용한다. 주로 집계 연산에 사용된다.

`reduce` 는 총 세 가지의 오버로딩된 메서드를 제공한다.

```java
Optional<T> reduce(BinaryOperator<T> accumulator)
```

스트림의 원소 중 첫 번째를 `accumulator`의 초기값으로 사용하여 남은 원소를 차례로 결합한다.

```java
List<Integer> nums = List.of(1, 2, 3, 4);
Optional<Integer> sum = nums.stream()
    .reduce((a, b) -> a + b);
// 10
```

---

```java
T reduce(T identity,
         BinaryOperator<T> accumulator)
```

초기값 `identity` 를 명시하고 그 이후 원소들을 차례대로 결합한다. 이전에는 `Optional` 로 리턴값을 받았는데, 이번에는 빈 스트림 또한 `identity` 를 리턴하기 때문에 `T` 타입으로 리턴값을 받는다.

```java
List<Integer> nums = List.of(1, 2, 3, 4);
int product = nums.stream()
    .reduce(1, (a, b) -> a * b);
// 24
```

---

```java
<U> U reduce(U identity,
             BiFunction<U, ? super T, U> accumulator,
             BinaryOperator<U> combiner)
```

위 메서드는 병렬 스트림에서 자주 사용하며, `U` 타입으로 `accumulator` 를 만들고, 각 원소를 누적하여 중간 결과를 생성하고, 최종적으로 이를 `combiner` 로 합쳐 리턴한다.

```java
List<String> words = List.of("Java", "Stream", "API");
int totalLength = words.parallelStream()
    .reduce(
        0,
        (sum, w) -> sum + w.length(),
        Integer::sum
    );
// 13
```

### stream의 집계 함수

```java
long count()
```

`count` 메서드는 스트림에 포함된 요소의 총 개수를 `long` 타입으로 리턴한다. 내부적으로 모든 원소를 순회하여 개수를 세고 결과를 리턴한다. 빈 스트림의 경우 0을 리턴하며, 무한 스트림인 경우 `count` 메서드는 종료되지 않는다.

```java
List<String> fruits = List.of("apple", "banana", "cherry", "durian");
long count = fruits.stream().count();
// 4
```

---

```java
int sum() // IntStream의 경우
```

`sum` 메서드는 스트림의 모든 원소의 합을 리턴한다. 스트림의 모든 원소를 내부 버퍼에 저장할 후 0을 초기값으로 하여 원소를 차례로 더한다.

```java
int total = IntStream.of(10, 20, 30, 40, 50)
                     .sum();
// 150
```

---

```java
Optional<T> max(Comparator<? super T> comparator)
```

`max` 메서드는 스트림 원소 중 가장 큰 값을 리턴한다. 객체 스트림인 경우 `Comparator` 를 인자로 넘겨주어야 한다.

```java
List<String> names = List.of("Alice", "Bob", "Charlie");

Optional<String> opt = names.stream()
    .max(Comparator.naturalOrder());
```

### find 계열

```java
Optional<T> findFirst()
```

`findFirst` 메서드는 스트림의 첫 번째 원소를 탐색하여 리턴한다. 이는 병렬 스트림에서도 마찬가지이다. 즉, `findFirst` 메서드는 내부적으로 가장 작은 인덱스를 가진 원소를 리턴한다.

```java
List<String> list = List.of("A1", "A2", "A3", "A4");
Optional<String> firstParallel = list.parallelStream()
    .filter(s -> s.startsWith("A"))
    .findFirst();
System.out.println(firstParallel.get());
// A1
```

---

```java
Optional<T> findAny()
```

`findAny` 메서드는 스트림에서 임의의 원소를 찾아 `Optional<T>` 로 리턴한다. 순차 스트림인 경우 `findFirst` 메서드와 동일하게 동작하며, 병렬 스트림인 경우 각 쓰레드 의 스트림 중 가장 먼저 처리된 원소를 리턴한다.

### toArray

```java
Object[] toArray();
```

`toArray` 메서드는 스트림의 원소들을 배열로 변환한다.

스트림의 최종 크기를 미리 알 수 없기 때문에 스트림의 원소들을 `ArrayList` 와 같이 크기를 조정할 수 있는 컨테이너에 담고, 개수를 계산하여 배열을 생성하고 컨테이너의 원소들을 새로 생성된 배열에 복사하는 과정을 진행한다.

```java
Stream<String> stream = Stream.of("apple", "banana", "cherry");
Object[] array = stream.toArray();
// [apple, banana, cherry]
```

### match 계열

```java
boolean allMatch(Predicate<? super T> predicate);
```

`allMatch` 메서드는 스트림의 모든 원소가 주어진 조건 `predicate` 를 만족하는지 검사하는 연산이다.

조건을 만족하지 않는 원소를 발견하는 순간 즉시 `false` 를 리턴하고 더 이상 원소를 처리하지 않는다. 만약 빈 스트림을 검사하는 경우 항상 `true` 를 리턴한다.

단, 무한 스트림의 경우 조건을 만족하는 원소가 없다면 영원히 실행될 수 있다.

```java
List<Integer> numbers = Arrays.asList(2, 4, 6, 8, 10);
boolean allEven = numbers.stream()
    .allMatch(num -> num % 2 == 0) // true
```

`anyMatch` 는 하나라도 조건을 만족하면 `true` 를 리턴한다. 조건을 만족하는 원소를 발견하는 즉시 검사를 종료하고 `true` 를 리턴한다.

`noneMatch` 는 모든 원소가 조건을 만족하지 않으면 `true` 를 리턴한다. 조건을 만족하는 원소를 발견하는 즉시 검사를 종료하고 `false` 를 리턴한다.