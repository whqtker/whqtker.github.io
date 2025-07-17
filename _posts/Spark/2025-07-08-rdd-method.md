---
title : "[Spark] RDD 관련 메서드"
date : 2025-07-18 00:30:00 +0900
categories : [Spark]
tags : [spark, 스파크, rdd, pyspark]
---

## 📌 RDD 관련 메서드

```python
text_file = sc.textFile(test_file)
```

`textFile` 은 텍스트 파일을 읽어 각 줄을 하나의 문자열로 갖는 RDD를 생성하는 메서드이다.

```python
counts = text_file.flatMap(lambda line: line.split(" ")) \
             .map(lambda word: (word, 1)) \
             .reduceByKey(lambda a, b: a + b)
```

`flatMap` 은 `map` 메서드의 결과를 평평하게 펴서 하나의 리스트로 만드는 메서드이다. 일대다 매핑이며 컬렉션을 리턴해야 한다.

`map` 메서드는 주로 RDD의 키-값 쌍을 생성할 때 사용한다. 일대일 매핑이며 단일 객체나 값을 리턴해야 한다.

다만 값만 필요한 경우 `map`, `flatMap` 대신 `mapValues` 나 `flatMapValues` 를 사용하는 것이 더 효율적이다. RDD에는 특정 키가 어느 파티션에 저장될지를 결정하는 규칙인 `partitioner` 가 존재할 수 있는데, 만약 존재한다면 비싼 작업인 셔플링을 생략할 수 있다.

`map` 메서드는 함수가 키, 값 모두 접근할 수 있으므로 키가 변경될 수 있는 가능성이 열려있다. 따라서 Spark 입장에서 `map` 연산 후 키가 그대로 유지될 것이라는 보장은 없다. 따라서 원본 RDD의 파티셔너 정보를 소실시킨다. 이후 `reduceByKey` 같은 연산을 진행하면 셔플링을 수행한다.

`reduceByKey` 메서드는 키-값 쌍의 값을 집계하는 데 사용한다. 내부적으로 셔플링을 최소화하기 위해 `Map-side Combine` 방법을 사용한다. 각 워커 노드는 자신의 파티션에서 동일한 키의 값들을 집계한 후 셔플링한다.

이는 `groupByKey` 와 대조되는데, `groupByKey` 는 `Map-side Combine` 없이 바로 셔플링을 진행한다.

```python
print(counts.collect())
```

`collect` 는 여러 워커 노드에 흩어진 모든 RDD를 수집하여 파이썬 리스트로 리턴하는 메서드이다.

```python
grade_count = grade.countByValue()
```

`countByValue` 메서드는 RDD의 값이 몇 번 나타나는지 계산하는 메서드이다.

```python
rdd = sc.parallelize([("a", 1), ("b", 1), ("a", 1)])
sorted(rdd.groupByKey().mapValues(len).collect())
# [('a', 2), ('b', 1)]

sorted(rdd.groupByKey().mapValues(list).collect())
# [('a', [1, 1]), ('b', [1])]
```

`parallelize` 는 키-값 형태의 데이터를 RDD로 생성하는 메서드이다.

`groupByKey` 는 RDD의 모든 데이터를 셔플링하는 메서드이다. 이 결과는 `ResultIterable` 이라는 객체인데, 여기에 `mapValues(list)` 를 적용하면 파이썬 리스트로 변환된다.

```python
tmp = [('a', 1), ('b', 2), ('1', 3), ('d', 4), ('2', 5)]
sc.parallelize(tmp).sortByKey().first()
# ('1', 3)
```

`sortByKey` 는 키를 기준으로 RDD를 정렬하는 메서드이다. 먼저 RDD의 키들을 샘플링하여 분포를 파악한 후 이를 바탕으로 각 파티션이 담당할 키의 범위를 결정한다. 이후 셔플링을 통해 재분배를 한 후, 각 파티션 내부에서 데이터를 정렬한다.

`first` 는 RDD의 첫 번째 원소를 가져오는 action 연산이다. 단, RDD의 파티션 및 파티션 내 원소의 순서는 기본적으로 보장되지 않으므로 매번 `first` 메서드를 실행할 때마다 다른 결과가 나올 수 있다.

```python
rdd = sc.parallelize([("a", 1), ("b", 1), ("a", 1)])
rdd.keys()
# ['a', 'b', 'a']
```

`keys` 는 RDD의 키를 추출하여 새로운 RDD를 생성하는 메서드이고, `values` 는 RDD의 값을 추출하여 새로운 RDD를 생성하는 메서드이다.

```python
x = sc.parallelize([("a", 1), ("b", 4)])
y = sc.parallelize([("a", 2), ("a", 3)])
sorted(x.join(y).collect())
# [('a', (1, 2)), ('a', (1, 3))]
```

`join` 은 RDD를 키를 기준으로 결합하는 메서드이다.