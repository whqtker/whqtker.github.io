---
title : "JUnit, AssertJ 라이브러리를 사용한 테스트"
date : 2025-06-26 12:00:00 +0900
categories : [Java]
tags : [java, junit, assertj, test]
---

## 📌 JUnit이란?

`JUnit` 은 자바의 단위 테스트 프레임워크이다. 단위 테스트를 자동화하여 메서드 수준까지 독립적으로 테스트를 수행할 수 있다. 또한 어노테이션과 여러 라이브러리를 통해 간결하고 직관적으로 테스트 코드를 작성할 수 있다.

JUnit은 테스트 메서드 실행 순서를 보장하지 않고 독립적으로 테스트를 수행하므로 테스트 간 의존성이 존재하지 않도록 작성해야 한다.

## 📌 어노테이션

JUnit 5를 기준으로 작성하였다.

### @Test

```java
@Test
void testAddition() {
    // Given
    Calculator calculator = new Calculator();
    int a = 5;
    int b = 3;

    // When
    int result = calculator.add(a, b);

    // Then
    assertEquals(8, result, "5 + 3 = 8");
}
```

`@Test` 는 테스트 메서드임을 표시하는 어노테이션이다. 테스트를 실행하면 자동으로 해당 메서드를 실행한다.

### @DisplayName

```java
@Test
@DisplayName("덧셈 기능 테스트")
void testAddition(TestInfo testInfo) {
    System.out.println("현재 실행 중인 테스트: " + testInfo.getDisplayName());
    
    // ...
}
```

`@DisplayName` 는 테스트 클래스 또는 메서드에 사용자 정의 이름을 설정하는 어노테이션이다. 

`TestInfo.getDisplayName` 메서드를 통해 `@DisplayName` 에 설정한 이름을 리턴받을 수 있다.

### @Disabled

```java
@Test
@Disabled("테스트 비활성화")
@DisplayName("뺄셈 기능 테스트")
void testSubtraction() {
    // Given
    Calculator calculator = new Calculator();
    int a = 10;
    int b = 4;

    // When
    int result = calculator.subtract(a, b);

    // Then
    assertEquals(6, result);
}
```

`@Disabled` 는 테스트 클래스 또는 메서드 실행을 일시적으로 비활성화하는 어노테이션이다. 클래스 레벨에 설정하게 되면 해당 클래스 내 모든 테스트 메서드가 자동으로 비활성화된다.

`@Disabled` 만 단독으로 선언하면 테스트 코드로 인식하지 않으므로 `@Test` 와 함께 사용해야 한다.

`@Disabled` 가 붙은 테스트 메서드가 있어도 해당 테스트 클래스의 인스턴스는 그대로 생성되며, 해당 메서드의 `@BeforeEach` , `@AfterEach` 메서드 호출 또한 자동으로 실행하지 않는다. 이러한 콜백 메서드에서 예외가 발생하면 해당 테스트 자체가 실패한 것으로 간주한다.

![image.png](assets/img/junit-assertj/1.png)

테스트를 실행하지 않고 무시한다.

### @BeforeEach / @AfterEach

```java
class CalculatorTest {

  private Calculator calculator;

  @BeforeEach
  void setUp() {
      System.out.println("테스트 시작");
      calculator = new Calculator();
  }

  @AfterEach
  void tearDown() {
      System.out.println("테스트 종료");
      calculator = null;
  }

  @Test
  @DisplayName("덧셈 기능 테스트")
  void testAddition(TestInfo testInfo) {
      // ...
  }
}
```

`@BeforeEach` , `@AfterEach` 어노테이션을 통해 테스트 메서드 전후로 수행할 로직을 정의할 수 있다. 

`@BeforeEach` 어노테이션이 붙은 메서드는 테스트 메서드 실행 전에 자동으로 호출된다. 보통 테스트 객체를 초기화하거나, 테스트 데이터를 세팅, DB 커넥션 등을 위해 사용된다. 

`@AfterEach` 어노테이션이 붙은 메서드는 테스트 메서드 실행 후에 자동으로 호출된다. 보통 메모리 리소스를 정리하거나 외부 자원을 해제하는 데 사용된다.

### @BeforeAll / @AfterAll

`@BeforeAll` , `@AfterAll` 은 테스트 클래스 전체의 생명주기를 다루는 콜백 메서드를 지정하기 위한 어노테이션이다. 이 어노테이션이 붙은 메서드는 `static` 메서드여야 하며 리턴 타입은 `void` 여야 한다,

`@BeforeAll` 은 모든 테스트 메서드보다 먼저 한 번만 실행된다. 

`@AfterAll` 은 모든 테스트 메서드 실행이 완료된 후 한 번만 실행된다.

### @ParameterizedTest / @ValueSource

```java
@ParameterizedTest
@ValueSource(ints = {10, 20, 30, 40, 50})
void testSubtractionValueSource(int number) {
    // When
    int result = calculator.subtract(number, 5);
    
    // Then
    assertEquals(number - 5, result);
}
```

`@ParameterizedTest` 는 하나의 테스트 메서드를 여러 입력을 사용하여 실행하도록 하는 어노테이션이다. `@ValueSource`, `@CsvSource`, `@MethodSource` 등과 같은 어노테이션과 함께 사용해야 한다.

`@ValueSource` 는 `@ParameterizedTest` 가 설정된 메서드에 필요한 파라미터를 전달하는 어노테이션이다. 다양한 데이터 타입을 지원하며, 배열 형태로 값을 지정할 수 있다.

### @RepeatedTest

```java
@RepeatedTest(5)
void testAdditionRepeated(RepetitionInfo repetitionInfo) {
    System.out.println("현재 반복: " + repetitionInfo.getCurrentRepetition() +
            " / " + repetitionInfo.getTotalRepetitions());

    // Given
    int a = repetitionInfo.getCurrentRepetition();
    int b = 10;

    // When
    int result = calculator.add(a, b);

    // Then
    assertEquals(a + b, result, a + " + " + b + " = " + (a + b));
}
```

`@RepeatedTest` 는 테스트 메서드를 지정된 횟수만큼 반복하여 실행하도록 설정하는 어노테이션이다. 해당 테스트 메서드가 실패하면 반복 횟수와 상관없이 실패로 간주한다.

`RepetitionInfo.getTotalRepetitions` 메서드를 통해 현재 메서드 반복 횟수를 리턴받을 수 있다.

### @Nested

```java
@Nested
@DisplayName("나눗셈 연산")
class DivisionTests {

    @Test
    @DisplayName("정상적인 나눗셈 테스트")
    void testDivision() {
        // Given
        int a = 10;
        int b = 2;

        // When
        int result = calculator.divide(a, b);

        // Then
        assertEquals(5, result);
    }

    @Test
    @DisplayName("0으로 나눌 때 예외 발생 테스트")
    void testDivisionByZero() {
        // Given
        int a = 10;
        int b = 0;

        // When & Then
        IllegalArgumentException exception = assertThrows(IllegalArgumentException.class, () -> {
            calculator.divide(a, b);
        });

        assertEquals("0으로 나눌 수 없습니다.", exception.getMessage());
    }
}
```

`@Nested` 는 테스트 클래스 내부에 중첩 테스트 클래스를 정의할 수 있도록 하는 어노테이션이다. 테스트 클래스를 계층적 구조로 그룹화할 수 있다. 중첩 클래스는 외부 클래스와 인스턴스 상태를 공유할 수 있다.

외부 클래스에 설정된 `@Before/AfterEach` 는 중첩 클래스 테스트 전후에도 호출되며, 만약 중첩 클래스에 별도의 `@Before/AfterEach` 을 선언한 경우 콜백 메서드가 순차적으로 실행된다.

### @Timeout

```java
@Test
@Timeout(value = 100, unit = TimeUnit.MILLISECONDS)
void testAdditionTimeout() {
    int result = 0;
    for (int i = 0; i < 1000; i++) {
        result = calculator.add(result, i);
    }
    
    int expectedSum = (999 * 1000) / 2;
    assertEquals(expectedSum, result);
}
```

`@Timeout` 은 테스트 클래스 또는 메서드의 실행 시간을 제한하여 이를 초과하면 실패로 간주하도록 하는 어노테이션이다. 테스트 클래스에 선언한 경우 클래스 내부의 모든 테스트에 동일한 제약 조건이 적용된다. 단, 제약 조건은 콜백 메서드에는 적용되지 않는다.

## 📌 AssertJ

`AssertJ` 는 자바 assertion 라이브러리로 JUnit 등과 같은 테스트 프레임워크와 함께 사용되어 테스트 코드를 작성하는 데 사용된다. 메서드 체이닝을 통해 직관적으로 로직을 작성할 수 있고, 다양한 데이터 타입에 대한 API를 제공한다.

### Is(Not)EqualTo

`isEqualTo`는 실제 값이 기대 값과 동일한지 검증하는 메서드이다. 내부적으로 자바의 `equals` 메서드를 호출하므로 이 메서드가 구현된 모든 데이터 타입에 작용할 수 있다. 만약 참조 동등성을 비교하려면 `isSameAs` 메서드를 사용해야 한다.

### is(Not)Null

`isNull` 은 테스트 대상 값이 `null` 인지 확인하는 메서드이다. `isEqualTo(null)` 과 동일한 역할을 수행하지만 `isNull` 이 의도가 더 명확하기 때문에 이를 사용하는 것이 권장된다.

### isTrue / isFalse

`isTrue` 는 테스트 대상 값이 `true` 인지 확인하는 메서드이다.

### matches / doesNotMatch

`matches` 는 테스트 대상 문자열이 주어진 정규 표현식과 완전히 일치하는지 검증하는 메서드이다.

### startsWith / endsWith

`startsWith` 는 테스트 대상 문자열이 지정된 접두사로 시작하는지 검증하는 메서드이다. 문자열 전체를 탐색하지 않아 불필요한 오버헤드를 줄인다.

`assertThat(target).startsWith(prefix)` 와 `target.startWith(prefix)` 의 동작은 동일하다.

### contains(IgnoringCase)

`contains` 는 컬렉션, 배열, 문자열에 대해 지정된 값이 포함되었는지 검증하는 메서드이다. 순서나 중복 여부를 무시하고, 최소 한 번이라고 일치하면 테스트는 통과한다.

`containsExactly` 는 원소가 지정된 순서대로 정확히 일치하여 테스트가 통과하는 메서드이다. 반면 `containsExactlyInAnyOrder` 는 순서에 무관해도 된다.

`containsSequence` 는 테스트 대상에 대해 지정된 연속 시퀀스가 포함되어 있는지 검증하는 메서드이다. 즉 특정 값이 순서대로 연속해서 존재해야 하며 중간에 다른 값이 있다면 테스트는 실패한다. `containsSubsequence` 는 순서는 유지하되 중간에 다른 값이 있어도 된다.

### hasSize

`hasSize` 는 컬렉션, 배열, 문자열의 길이가 기대 값이랑 일치하는지 검증하는 메서드이다.