---
title : "[Spark] Apache Spark 엔진에 대해 자세히 알아보자"
date : 2025-07-20 00:10:00 +0900
categories : [Spark]
tags : [spark, 스파크, codegen, sql hint, accumulator, pyspark, transformation, action, aqe]
---

## 📌 Spark Submit

Spark Submit은 스칼라나 자바, 파이썬 등으로 작성된 스파크 애플리케이션을 클러스터에 제출하여 실행하기 위한 명령이다.

---

애플리케이션을 스칼라로 작성한 경우 아래와 같이 작성한다.

```bash
/bin/spark-submit \
    --master yarn \
    --deploy-mode cluster \
    --driver-memory 8g \
    --executor-memory 16g \
    --executor-cores 2 \
    --class org.apache.spark.examples.SparkPi \
    /path/to/your-spark-app.jar 80

```

`/bin/spark-submit` 은 스파크 애플리케이션 제출을 위한 기본 명령이다.

`--driver-memory` 는 드라이버 프로세스에 할당한 메모리 크기를 지정한다.

`--executor-memory` 는 executor 프로세스에 할당할 메모리 크기를 지정한다.

`--class` 는 실행할 애플리케이션의 entrypoint가 되는 클래스의 이름을 지정한다. 마지막으로 실행할 애플리케이션 코드와 전달할 인자를 지정한다.

더 자세한 옵션 설명은 후술한다.

---

애플리케이션을 파이썬으로 작성한 경우 아래와 같이 작성한다.

```bash
/bin/spark-submit \
    --master yarn \
    --deploy-mode cluster \
    --py-files dependencies.zip,utils.py \
    main_app.py
```

### master

`spark-submit` 의 `master` 옵션에 전달할 수 있는 값을 알아보자.

| 옵션 | 값 | 설명 |
| --- | --- | --- |
| Yarn | `yarn` | 클러스터 자원이 Hadoop의 YARN에 의해 관리될 경우 |
| Standalone | `spark://HOST:PORT` | 스파크가 자체적으로 제공하는 독립형 클러스터에서 실행할 경우, `HOST`와 `PORT`는 실제 마스터 노드의 주소와 포트 |
| Kubernetes | `k8s://HOST:PORT` 또는`k8s://https://HOST:PORT` | 쿠버네티스 클러스터에서 실행하는 경우, `HOST`와 `PORT`는 쿠버네티스 API 서버의 주소와 포트, 기본적으로 HTTPS 연결을 시도 |
| local | `local[k]` 또는 `local[K,F]` | 클러스터 없이 로컬 머신에서 실행하는 경우,  `local` 은 단일 워커 스레드로 실행, `local[k]`는 로컬 머신의 `k`개 코어를 사용하여 `k`개의 워커 스레드로 실행,  `local[K,F]` 는`K`개의 워커 스레드로 실행, 실패 시 `F`번 재시도하도록 지정, `F` 는 최소 2 이상 |

### deploy-mode

`deploy-mode` 옵션에 전달할 수 있는 값을 알아보자.

| 값 | 설명 |
| --- | --- |
| `cluster` | 드라이버가 워커 노드 중 하나에서 실행되며 해당 노드는 애플리케이션의 스파크 웹 UI에 드라이버로 표시, 프로덕션 작업을 실행하는 데 사용, `spark-submit` 명령을 실행한 클라이언트가 연결이 끊어져도 애플리케이션은 계속 실행 |
| `client` | `spark-submit` 명령을 실행한 클라이언트 머신에서 직접 실행, 주로 대화형 작업 및 디버깅 목적으로 사용, 드라이버만 로컬에서 실행되고 다른 모든 익스큐터는 클러스터의 다른 노드에서 실행 |

## 📌 Spark API 카테고리

### Transformation, Action 연산

스파크에는 크게 두 가지의 연산 방법이 존재하는데, `transformation` 과 `action` 이다.

transformation 연산은 기존 RDD나 데이터프레임을 통해 새로운 RDD나 데이터프레임을 생성하는 모든 연산을 지칭한다. 가장 큰 특징은 해당 연산이 호출될 때 즉시 실행되지 않고 연산에 대한 실행 계획만 정의한다. 이를 `lazy evaluation` 이라고 한다.

action 연산은 정의된 모든 transformation 연산을 실제로 실행하도록 하고, 최종 결과를 만드는 연산이다. 호출되는 즉시 클러스터에서 작업이 실행된다.

### Dependency

‘의존성’은 transformation 연산이 실행될 때 부모 RDD/데이터프레임의 파티션과 자식 RDD/데이터프레임의 파티션이 어떻게 연결되는지에 대한 개념이다.

`Narrow Dependency` 는 부모 파티션이 최대 하나의 자식 파티션에만 영향을 미치는 관계이다. 가장 큰 특징은 셔플링이 발생하지 않는다는 점이다. 즉, 각 파티션은 독립적으로 계산될 수 있다. 당연히 네트워크 비용이 없기 때문에 속도가 빠르며, 파티션에 장애가 발생해도 해당 파티션의 부모 파티션만 다시 계산하면 되므로 복구가 빠르다. 다시 말해 `Fault Tolerance` 가 좋다.

`Wide Dependency` 는 하나의 자식 파티션이 여러 개의 부모 파티션에 의존하는 관계이다. 동일한 키를 가진 데이터를 한곳으로 모아야 하므로 셔플링이 필수적이다. 네트워크 및 디스크 I/O로 인해 성능 저하 이슈가 있으며, 하나의 자식 파티션에 장애가 발생하면 연관된 모든 부모 파티션을 다시 계산해야 하므로 복구 비용이 크다.

### API별 연산

| API | 카테고리 | 연산 |
| --- | --- | --- |
| RDD API | Narrow Transformation | `map`, `mapPartitions`, `flatMap`, `filter`, `union` |
|  | Wide Transformation | `groupByKey`, `reduceByKey`, `aggregate`, `join`, `repartition` |
|  | Action | `collect`, `count`, `first`, `take`, `reduce` |
| DataFrame API | Narrow Transformation | `select`, `filter`, `withColumn`, `drop`, `where` |
|  | Wide Transformation | `groupBy`, `join`, `agg`, `cube`, `rollup` |
|  | Action | `show`, `head`, `first`, `count`, `collect` |

## 📌 Plan

### Logical Plan

`Logical Plan` 은 `catalyst optimizer` 가 사용자의 쿼리를 분석하고 최적화하는 단계이다. “무엇을 할 것인가”에 대한 계획이다.

다음 3단계를 거쳐 최적화된다.

1. `Unresolved Logical Plan` : SQL 쿼리나 데이터프레임 코드를 파싱하여 만든 트리 형태의 계획이다. 오직 코드의 문법적 유효성을 검증하며, 데이터프레임의 열이 실제로 존재하는지, 타입이 올바른지 등은 알지 못한다.
2. `Analyzed Logical Plan` : `catalog` 를 참조하여 이전 단계 계획의 유효성을 검증한 후의 계획이다. 즉, 이 계획인 문법적, 의미적으로 유효함이 보장된다.

> `catalog` 는 스파크가 관리하는 모든 데이터의 메타데이터 저장소이다.
> 
1. `Optimized Logical Plan` : 이전 단계 계획에 다양한 최적화 규칙을 적용하여 논리적으로 동등하면서 성능을 개선한 최종 계획이다. 주요 최적화 규칙은 다음과 같다. `Predicate Pushdown` 은 `WHERE` 절의 필터 조건을 가능한 데이터 소스에 가깝게 이동시킨다. 애초에 필요한 데이터로만 연산을 진행하여 네트워크 비용을 줄인다. `Projection Pruning` 은 최종 결과에 필요하지 않은 열은 데이터 소스를 읽는 단계부터 제거하는 방법이다. `Constant Folding` 은 쿼리 실행 전 미리 계산할 수 있는 상수 표현식은 컴파일 시간에 계산하는 방법이다.

### Physical Plan

Logical Plan이 “무엇을 할 것인가”를 정의했다면, `Physical Plan` 은 “어떻게 실행할 것인가”에 대한 방법을 정의한다.

논리적 계획 단계에 이어서 물리적 계획이 생성된다.

1. Optimized Logical Plan을 통해 여러 물리적 계획 전략을 생성한다. 각각의 전략을 `Candidate Physical Plan` 이라고 한다. 논리적으로 동일한 결과를 생성하는 다양한 실행 방법을 고려한다.
2. 이전 계획을 cost model에 기반하여 평가하고 그 중 가장 비용이 낮은 계획을 선택한다. 이 단계를 `Cost-Based Optimization, CBO` 라고 한다.
3. CBO를 통해 선택된 단 하나의 실행 계획을 `Final Physical Plan` 이라고 한다. 이 계획을 바탕으로 최적화된 자바 바이트코드를 동적으로 생성한다.

---

```python
final_df.explain(extended=True)
```

`explain` 은 쿼리 실행 계획을 사람이 읽을 수 있는 형태로 출력해주는 메서드이다. `extended` 파라미터는 모든 실행 계획을 보여줄지 여부를 결정한다.

```python
words = df.select(
    f.explode(
        f.split(df.value, ' ')).alias("word"))

word_counts = words.groupBy("word").count()

word_counts.explain(extended=True)
```

위 코드의 작업 순서를 살펴보자.

1. `split` 메서드를 통해 `df`의 `value`열을 공백을 기준으로 잘라 배열을 만든다.
2. `explode` 메서드를 통해 생성된 배열의 각 요소를 개별적인 `Row` 로 펼친다.
3. `alias` 메서드를 통해 새로 생성된 열의 이름을 `word` 로 지정한다.
4. `groupBy` 메서드를 통해 `word` 열의 값이 같은 것끼리 그릅화하고, 각 그룹에 속한 행의 개수를 센다.

```
== Parsed Logical Plan ==
'Aggregate ['word], ['word, count(1) AS count#16L]
+- Project [word#12]
   +- Generate explode(split(value#0,  , -1)), false, [word#12]
      +- Relation [value#0] text

== Analyzed Logical Plan ==
word: string, count: bigint
Aggregate [word#12], [word#12, count(1) AS count#16L]
+- Project [word#12]
   +- Generate explode(split(value#0,  , -1)), false, [word#12]
      +- Relation [value#0] text

== Optimized Logical Plan ==
Aggregate [word#12], [word#12, count(1) AS count#16L]
+- Generate explode(split(value#0,  , -1)), [0], false, [word#12]
   +- Relation [value#0] text

== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- HashAggregate(keys=[word#12], functions=[count(1)], output=[word#12, count#16L])
   +- Exchange hashpartitioning(word#12, 200), ENSURE_REQUIREMENTS, [plan_id=18]
      +- HashAggregate(keys=[word#12], functions=[partial_count(1)], output=[word#12, count#20L])
         +- Generate explode(split(value#0,  , -1)), false, [word#12]
            +- FileScan text [value#0] Batched: false, DataFilters: [], Format: Text, Location: InMemoryFileIndex(1 paths)[file:/home/jovyan/work/sample/lorem_ipsum.txt], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<value:string>
```

`explain` 메서드를 통해 얻은 스파크의 실행 계획이다. 하나씩 단계별로 살펴보자.

```
== Parsed Logical Plan ==
'Aggregate ['word], ['word, count(1) AS count#16L]
+- Project [word#12]
   +- Generate explode(split(value#0,  , -1)), false, [word#12]
      +- Relation [value#0] text
```

아래에서 위로 읽으면 된다.

1. 텍스트 데이터 소스를 읽는다.
2. `split`, `explode` 메서드를 수행한다.
3. `word` 열을 선택한다.
4. `word` 로 그룹화하고 개수를 센다.

Parsed Logical Plan이므로 `value` 열이 실제로 존재하는지 등은 아직 확인하지 않은 상태이다.

```
== Analyzed Logical Plan ==
word: string, count: bigint
Aggregate [word#12], [word#12, count(1) AS count#16L]
+- Project [word#12]
   +- Generate explode(split(value#0,  , -1)), false, [word#12]
      +- Relation [value#0] text
```

Analyzed Logical Plan은 Parsed Logical Plan 거의 유사한 형태이나, 각 열의 타입이 명시된 것을 확인할 수 있다.

```
== Optimized Logical Plan ==
Aggregate [word#12], [word#12, count(1) AS count#16L]
+- Generate explode(split(value#0,  , -1)), [0], false, [word#12]
   +- Relation [value#0] text
```

Optimized Logical Plan에서 이전 계획과 다른 점은 `word` 열을 선택하는 부분이 사라졌다. `explode` 연산을 통해 `word` 열을 생성하므로 이를 다시 선택하는 연산은 불필요하다고 판단한 것이다. `Projection Pruning` 이 사용된 예이다.

```
== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- HashAggregate(keys=[word#12], functions=[count(1)], output=[word#12, count#16L])
   +- Exchange hashpartitioning(word#12, 200), ENSURE_REQUIREMENTS, [plan_id=18]
      +- HashAggregate(keys=[word#12], functions=[partial_count(1)], output=[word#12, count#20L])
         +- Generate explode(split(value#0,  , -1)), false, [word#12]
            +- FileScan text [value#0] Batched: false, DataFilters: [], Format: Text, Location: InMemoryFileIndex(1 paths)[file:/home/jovyan/work/sample/lorem_ipsum.txt], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<value:string>
```

최종 물리적 계획이다. 역시 아래서부터 단계별로 살펴보자.

1. 디스크에서 텍스트 파일을 읽는다.
2. 각 노드에서 `split` 과 `explode` 연산을 수행한다.
3. 첫 번째 `HashAggregate` 은 Map 단계로, 셔플링 전 executor 노드가 자신의 데이터에 대해 부분 집계를 먼저 수행하는 단계이다. 전체 데이터를 네트워크에 태우기 전 미리 크기를 줄여 셔플링 효율을 높이기 위함이다.
4. `Exchange hashpartitioning` 단계에서 실제 셔플링을 진행한다. `word` 를 기준으로 해시값을 계산하여 동일한 단어는 항상 동일한 노드로 가도록 보장한다.
5. 두 번째 `HashAggregate` 은 Reduce 단계로, 각 노드는 자신에게 모은 동일한 단어들의 부분 집계 결과들을 최종적으로 합산한다.

## 📌 Codegen

스파크는 특정 쿼리에 고도로 최적화된 코드를 런타임에 동적으로 컴파일할 수 있다. 이러한 기술을 `Codegen` 이라고 한다.

Codegen 이전 방식은 데이터의 각 행에 대해 수행해야 하는 연산을 순차적으로 해석하고 실행하였다. 이 방법은 각 단계마다 virtual function call과 같은 오버헤드가 발생하여 성능 저하가 발생했다.

Codegen의 기본 개념은 쿼리 실행 계획의 여러 단계를 하나의 함수로 통합하고 이를 효율적인 자바 바이트코드로 즉석에서 만드는 것이다. 이 방법으로 데이터가 CPU 캐시를 벗어나지 않고 파이프라인처럼 연속적으로 처리될 수 있다.

Codegen은 `Tungsten Execution Engine` 의 주요 기능이다. Tungsten은 스파크의 성능을 소프트웨어 수준을 넘어 하드웨어 수준까지 최적화하는 것을 목표로 한다.

Tungsten이 생성하는 코드는 다양한 저수준 최적화 기법들이 적용된다. 그 종류는 다음과 같다.

- `Loop Unrolling`: 루프의 본문을 여러 번 펼처 한 번의 반복에 더 많은 작업을 수행하도록 코드를 재구성한다. 루프를 반복할 때마다 발생하는 오버헤드를 줄일 수 있다.
- `Vectorized Processing`: 한 행씩 처리하는 게 아닌 여러 행의 데이터를 묶은 열 단위 벡터를 처리하도록 한다.

## 📌 메모리 할당과 관리

스파크 클러스터의 메모리는 먼저 driver와 executor를 위해 할당된다.

각각의 executor에 할당된 메모리는 다리 여러 영역으로 나뉘어 사용된다. 스파크 1.6 이후 도입된 ‘통합 메모리 모델’이 이를 정의한다.

`Reserved Memory` 는 시스템 프로세스가 안정적으로 동작하기 위해 예약된 고정된 크기의 메모리 공간이다.

`Unified Memory` 는 Reserved Memory를 제외한 대부분의 영역이다. 이 공간은 `Execution Memory` 와 `Storage Memory` 가 동적으로 공유한다.

`Execution Memory` 는 데이터 처리 연산 중 일시적으로 필요한 데이터를 저장하는 공간이다. 연산이 끝나면 이 메모리는 비워진다.

`Storage Memory` 는 RDD나 데이터프레임 파티션을 캐시하여 재사용할 수 있도록 저장하는 공간이다.

`User Memory` 는 Unified Memory에 속하지 않는 나머지 공간으로 스파크가 관리하지 않는 사용자 정의 객체들을 위해 사용된다.

위의 모든 메모리들은 JVM Heap 내에서 관리된다. JVM 외부 메모리를 활용하는 경우를 살펴보자.

`Off-Heap Memory` 는 JVM의 GC 관리 대상에서 벗어난 Native Memory 영역이다. 따라서 대용량 메모리 사용 시 발생하는 GC 중단 현상을 회피할 수 있다. Tungsten Engine에서 정렬, 셔플 등과 같은 작업을 위해 이 메모리를 사용하기도 하며 직렬화된 RDD나 데이터프레임 파티션의 캐시를 저장하는 데 사용된다.

`External Process Memory` 는 JVM이 실행하는 별도의 외부 프로세스가 사용하는 메모리이다. 즉, 스파크의 메모리 관리자와 독립적이며 OS가 직접 관리한다. 예를 들어 PySpark나 SparkR은 각자의 코드를 실행하기 위해 별도의 프로세스를 생성한다.

스파크의 메모리와 관련된 간단한 문제를 통해 깊게 이해해보자.

> Driver 힙 메모리: 8GB
Executor 힙 메모리: 12GB
메모리 오버헤드 비율: 10%
Executor Off-Heap 메모리: 0.5GB
> 

1. 클러스터 매니저가 드라이버를 위해 할당해야 하는 총 메모리를 계산해보자. 총 메모리는 JVM 힙 메모리와 오버헤드를 더한 값이므로, 총 필요한 메모리는 8GB + 8GB * 0.1 = 8.8GB이다.
2. 하나의 executor를 위해 워커 노드에 할당해야 하는 총 메모리를 계산해보자. 총 메모리는 JVM 힙 메모리, 오버헤드, 오프힙 메모리를 모두 더한 값이다. 따라서 총 필요한 메모리는 12GB + 12GB * 0.1 + 0.5GB = 13.7GB이다.

메모리 부족에 따른 스파크의 동작을 예시 상황을 통해 살펴보자.

현재 executor가 6개의 코어를 할당받았고, 2개의 태스크를 동시에 실행하고 있다고 하자. 추가적인 메모리가 필요한 경우 다음 단계를 거쳐 할당받는다.

1. Storage Memory에 캐시된 데이터 블록이 있다면 덜 중요한 데이터 블록을 메모리에서 내리고 확부된 공간을 사용한다.
2. 그래도 공간이 부족하면 현재 태스크가 사용하던 데이터를 디스크의 임시 공간에 내린다(`Disk Spill`).
3. 메모리가 더 필요한 경우 사용하지 않은 4개의 쓰레드 풀을 사용한다. 최대 6개의 태스크를 실행할 수 있게 된다.
4. 그래도 공간이 부족하면 `OutOfMemoryError` 가 발생한다. 스파크 드라이버는 이를 감지하고 설정된 재시도 횟수만큼 해당 태스크를 다른 executor에서 재실행하려고 시도한다.

## 📌 Adaptive Query Execution

스파크 3.0부터 쿼리를 실행하면서 계획을 수정하는 방법인 `Adaptive Query Execution, AQE` 가 도입되었다. 쿼리를 실행하는 도중에 실제 데이터의 통계를 확인하고 이를 바탕으로 계획을 동적으로 최적화한다.

우선 기존 방식대로 초기 물리적 실행 계획을 수립하고, 각 단계를 실행하면서 실제 런타임 통계를 수집한다. 수집된 통계를 바탕으로 남은 쿼리의 실행 계획을 동적으로 수정한다.

AQE만의 최적화 기능은 다음과 같다.

1. `Post-Shuffle Coalescing` : `groupBy`, `join` 등으로 데이터가 줄어들어도 초기에 설정된 많은 셔플 파티션을 그대로 유지하는 경향이 있었다. 수많은 작은 파일들이 생성되기에 성능 저하를 유발한다. AQE는 셔플 후 각 파티션의 실제 데이터 크기를 확인하고 데이터가 예상보다 작아졌다면 파티션 수를 동적으로 줄인다.
2. `Dynamic Join Strategy Switching` : 가장 일반적인 조인 방법인 `Sort-Merge Join` 사용 시 한쪽 테이블의 크기가 작다면 불필요한 셔플링이 발생한다. 이를 방지하기 위해 AQE는 조인 직전 테이블의 실제 크기를 확인한 후 한쪽 테이블이 브로드캐스트할 정도로 작아졌다면 `Broadcast Hash Join` 으로 동적으로 방법을 전환한다.
3. `Skewed Join Handling` : 조인 시 특정 키에 데이터가 몰려있는 경우 해당 태스크를 처리하는 시간이 늘어나 전체 작업이 지연되는 병목 현상이 발생한다. AQE는 다른 파티션보다 비정상적으로 큰 파티션을 감지해 데아터가 편향된 파티션을 발견하면 해당 파티션을 여러 개의 더 작은 하위 파티션으로 분할한다. 그리고 반대 쪽 테이블의 해당 키를 복제하여 분할된 파티션과 조인한다.

### Post-Shuffle Coalescing

스파크는 셔플링 후 파티션의 수가 내부 설정에 따라 고정되는 경우가 많다. 그러나 실제 데이터 크기는 설정값보다 작아질 수 있는데, 이렇게 되면 스파크 드라이버는 작은 태스크를 각각 스케쥴링하고 관리해야 하므로 오버헤드가 커진다.

이러한 문제를 막기 위해 스파크는 셔플링 후 파티션의 통계를 수집한다. 통계를 바탕으로 설정된 목표 크기보다 현저히 작은 파티션들을 식별한 후, 이를 동적으로 병합한다.

`spark.sql.adaptive.coalescePartitions.enabled` 를 활성화해야 Post-Shuffle Coalescing 를 사용할 수 있다. `spark.sql.adaptive.advisoryPartitionSizeInBytes` 는 병합 후 각 파티션이 가지게 될 목표 크기를 설정한다. `spark.sql.adaptive.coalescePartitions.minPartitionNum` 는 병합 후 남게 될 최소 파티션의 수를 설정한다.

### Dynamic Join Strategy Switching

`Sort-Merge Join, SMJ` 는 대용량 데이터 간 조인을 진행하는 가장 안정적인 전략이다. 먼저 각 데이터를 조인 키를 기준으로 셔플링을 한 후, 각 파티션 내에서 조인 키를 기준으로 정렬한다. 이후 정렬된 데이터를 순차적으로 병합하면서 조인을 수행한다. 다만 이 방법은 데이터의 크기가 증가할수록 비효율성이 증가한다.

> 셔플링과 졍렬은 비싼 연산이다!
> 

AQE는 비싼 방법은 SMJ를 피하기 위해 런타임에 조인 전략을 `Broadcast Hash Join, BHJ` 으로 동적으로 변환할 수 있다.

BHJ는 데이터 중 하나의 실제 크기가 브로드캐스트 임계값보다 작을 때 수행된다. 먼저 조인 직전 데이터의 실제 크기를 확인한 후, 크기가 임계값보다 작다면 기존의 SMJ 계획을 폐기한다. 이후 크기가 작은 데이터를 클러스터의 모든 executor 노드에 broadcast하고 나머지 데이터는 셔플 없이 조인을 수행한다.

데이터의 크기가 브로드캐스트하기에 너무 크지만 특정한 조건을 만족하는 경우 SMJ를 `Shuffled Hash Join, SHJ` 으로 변환할 수 있다.

SHJ는 데이터가 이미 특정 키를 기준으로 올바르게 파티셔닝되었을 때, 스파크가 정렬 단계를 생략하는 것이 더 효율적이라고 판단할 때 사용한다. SHJ는 SMJ와 비슷하게 동작하나, 정렬 단계를 완전히 생략하고, 한쪽 데이터를 기반으로 해시 테이블을 만들고 나머지 데이터를 스트리밍하여 해시 조인을 수행한다.

### Skewed Join Handling

조인 연산을 수행할 때 데이터가 모든 파티션에 균등하게 분배되지 않고 특정 키에 데이터가 집중되어 일부 파티션이 나머지 파티션보다 현저히 크기가 큰 현상을 ‘데이터 편향’이라고 한다. 이러한 편향된 파티션을 처리하는 태스크가 전체 태스크의 병목 지점이 된다.

AQE는 데이터 편향 문제를 분할 정복 기법을 사용하여 해결한다.

먼저 셔플 작업이 완료된 후, 파티션 통계를 수집한다. 파티션의 크기가 절대적인 임계값보다 큰 지, 파티션의 크기가 모든 파티션 크기의 중앙값보다 n배 이상으로 큰 지를 확인한다. 이 조건들을 모두 만족하면 스파크는 해당 파티션을 편향된 파티션으로 간주한다.

편향된 파티션을 감지하면 AQE는 해당 파티션으로 여러 개의 작은 하위 파티션으로 분할한다. 분할된 각각의 파티션과 조인하기 위해 반대편 파티션의 데이터를 복제한 후 진행한다. 하나의 거대하고 느린 작업이 여러 개의 작고 빠른 병렬 작업으로 바뀐 것이다.

`spark.sql.adaptive.skewJoin.enabled` 는 데이터 편향 조인 기능을 활성화하기 위해 사용된다.

`spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` 는 파티션이 편향되었다고 간주하기 위한 최소 임계값이다.

`spark.sql.adaptive.skewJoin.skewedPartitionFactor` 는 파티션이 편향되었다고 판단하기 위한 상대적 크기 배수이다.

## 📌 Partition Pruning

`Partition Pruning` 은 쿼리 실행 시 읽을 필요가 없는 데이터 파티션을 건너뛰는 기법이다.

### Static Partition Pruning

`Static Partition Pruning` 은 읽어야 할 파티션이 쿼리 컴파일 시점에 명시적으로 결정되는 경우이다. `WHERE` 절에 파티션 키에 대한 필터 조건이 명시되어 있으면 해당 조건에 맞는 파티션만 읽도록 계획을 수립한다.

### Dynamic Partition Pruning

`Dynamic Partition Pruning, DPP` 는 쿼리 실행 중에 동적으로 필터 조건을 알아내어 불필요한 파티션을 제거하는 방법이다. 주로 거대한 `Fact Table` 과 상대적으로 작은 `Dimension Table` 이 있는 `Star Schema` 구조에서 효과적이다.

`catalyst optimizer` 는 쿼리를 분석하여 dimension table에 필터 조건이 적용되고, fact table과 조인하는 데 사용되는 키가 fact table을 물리적으로 나누는 파티션 키와 동일하다면 옵티마이저는 dimension table의 필터링 결과를 fact table의 파티션을 건너뛰는 데 사용한다. 이후 동적 필터가 포함된 실행 계획을 생성한다.

이후 전체 조인을 수행하기 전 dimension table에 대한 필터링 작업을 먼저 독립적으로 실행한다. 이 서브쿼리의 결과는 partition pruning에 사용될 실제 값들과 동일하다.

계산된 필더 값 목록을 모든 executor 노드에 브로드캐스트한다. 각각의 executor는 전달받은 필터 값 목록을 로컬 메모리에 캐시한다. 이렇게 되면 모든 executor는 어떤 파티션을 읽어야 하는지에 대한 정보를 로컬에서 즉시 참조할 수 있게 된다.

이제 각 executor는 fact table을 읽는다. 특정 파티션 데이터를 디스크에서 실제로 읽기 전, 읽으려는 파티션의 키 값이 브로드캐스팅으로 받은 필터 값 목록에 포함되어 있는지 확인한다. 만약 없다면 해당 파티션에 대한 디스크 I/O 작업을 건너뛴다. 이로 인해 불필요한 디스크 I/O 작업을 막을 수 있다.

DPP가 유용한 상황은 다음과 같다.

1. 한 테이블의 필터가 다른 테이블의 결과에 의존할 때
2. 파티션이 많은 fact table에 대해 일부만 읽어야 하는 쿼리
3. 서브쿼리의 집합 연산에 따라 판별하는 경우

`spark.sql.optimizer.dynamicPartitionPruning.enabled` 를 `true` 로 설정하면 DPP를 활성화할 수 있다. `spark.sql.optimizer.dynamicPartitionPruning.timeOut` 는 partition pruner를 기다리는 시간의 최대값이다. 제한 시간 내 결과가 없다면 DPP 없이 실행된다.

DPP는 필요한 데이터만 읽기 때문에 디스크 I/O와 같은 비싼 연산을 줄일 수 있다. 또한 조인이나 집계와 같은 연산의 대상이 되는 데이터 양 자체를 줄일 수 있으며 쓸데없는 파티션을 처리하지 않아 실행 시간을 줄일 수 있다.

그러나 필터 값을 동적으로 추출하고 이를 브로드캐스트하는 부가적인 연산이 필요하며, 따라서 필터 목록이 지나치게 크다면 역효과가 발생할 수 있다.

## 📌 Spark Cache

스파크는 여러 단계의 transformation 연산 과정에서 중간 결과물을 executor의 Storage Memory에 저장했다가 필요할 때마다 재사용할 수 있다. 캐싱은 각 executor에 분산된 파티션 수준에서 이루어진다.

`cache` 메서드 역시 Lazy Evaluation 방식으로 동작하는데, 해당 메서드를 호출할 때 즉시 데이터가 메모리에 저장되는 것이 아니라 action 연산이 호출될 때 실행된다.

`persist` 메서드는 `cache` 와 유사하나, 저장 수준을 직접 저장할 수 있다. `cache` 메서드는 `persist(StorageLevel.MEMORY_ONLY)` 와 동일하다. `MEMORY_AND_DISK` 는 메모리에 일단 저장하고, 공간이 부족하면 디스크에 저장하는 방법이다.

`unpersist` 는 호출하면 캐시된 데이터를 메모리에서 해제할 수 있다.

사용 가능한 메모리보다 큰 데이터를 캐시하려고 하면 일부 파티션은 캐시되지 못하거나 기존에 캐시된 데이터가 메모리에서 밀려나는 현상이 발생한다. 또한 캐싱의 목적에 맞게 잘 사용해야 한다. 사용이 끝난 데이터는 메모리에서 해제하고, 반복적으로 사용되는 데이터만 캐시하자.

## 📌 Repartition, Coalesce

`repartition` 은 데이터프레임의 파티션 수를 줄이거나 늘려 지정된 수의 파티션에 균등하게 재분배하는 메서드이다. 셔플링을 통해 클러스터의 모든 노드에 흩어진 데이터를 다시 섞은 후, 지정된 수의 파티션에 균등하게 담는다.

계산량이 많은 작업을 수행하기 전 파티션의 수를 늘려 더 많은 클러스터의 코어를 활용하거나 데이터 편향을 해소할 때 사용한다.

`repartitionByRange` 는 하나 이상의 열의 값 범위를 기준으로 데이터를 재분배하는 메서드이다. 데이터프레임에만 사용할 수 있다. 셔플링을 진행한다는 점은 `repartition` 과 동일하지만, 지정된 열의 값에 따라 데이터를 정렬하여 파티션을 나눈다. 데이터가 자연적인 순서나 범위를 가질 때 유용하다.

`coalesce` 는 파티션의 수를 효율적으로 줄이고자 할 때 사용하는 메서드이다. 가장 큰 차이점은 셔플링을 수행하지 않는다는 것이다. 기존 파티션을 인접한 것까지 합치는 방식으로 동작한다.

| 구분 | `repartition` | `repartitionByRange` | `coalesce` |
| --- | --- | --- | --- |
| 파티션 추가 | O | O | X |
| 파티션 감소 | O | O | O |
| 데이터 셔플 | 전체 셔플 수행 | 범위 기반 셔플 수행 | 셔플 최소화 |
| 성능 | 낮음 | 최적화되었으나 낮음 | 높음 |
| 사용 가능 API | RDD, DataFrame | DataFrame만 | RDD, DataFrame |

## 📌 Spark SQL Hint

`Spark SQL Hint` 는 개발자가 스파크의 optimizer에게 쿼리를 더 효율적으로 실행할 수 있는 방법을 제안하는 지시어이다. 최종 결과에 영향을 주진 않지만 물리적 실행 계획에 영향을 줄 수 있다.

데이터의 통계 정보가 부족하거나 쿼리가 매우 복잡할 경우 `Cost-Based Optimizer, CBO` 는 최적의 실행 계획을 찾지 못할 수 있다. 개발자가 데이터에 대해 더 잘 알고 있는 경우 힌트를 통해 옵티마이저의 동작을 재정의하고, 특정 실행 전략을 강제한다.

힌트는 SQL 주석 안에 `+` 기호를 붙여서 사용한다. 하나 이상의 힌트는 `,` 로 구분한다.

```sql
SELECT /*+ BROADCAST(small_table) */ *
FROM large_table
JOIN small_table ON large_table.id = small_table.id;
```

`Broadcast Hint` 는 조인 연산 시 지정된 테이블을 클러스터의 모든 execitor 노드에 broadcast하도록 제안한다. 큰 데이터블과 작은 테이블을 조인할 때 작은 테이블을 브로드캐스트하는 것이 셔플링을 막을 수 있어 성능 상 유리하다.

```sql
SELECT /*+ COALESCE(10) */ * FROM my_table;
```

`Coalesce Hint` 는 데이터프레임의 최종 파티션의 수를 지정된 개수로 줄이도록 제안한다.

```sql
SELECT /*+ REPARTITION(100) */ * FROM my_table;
```

`Repartition Hint` 는 데이터프레임의 최종 파티션 수를 지정된 개수로 변경하도록 제안한다.

```sql
SELECT /*+ MERGE(large_table, small_table) */ *
FROM large_table
JOIN small_table ON large_table.id = small_table.id;
```

`Merge Hint` 는 조인 전략으로 SMJ을 사용하도록 제안한다.

```sql
SELECT /*+ SHUFFLE_HASH(large_table) */ *
FROM large_table
JOIN medium_table ON large_table.id = medium_table.id;
```

`Shuffle Hash Hint` 는 조인 전략으로 SHJ을 사용하도록 제안한다.

### 장점

- 셔플링과 불필요한 데이터 읽기를 최소화하여 성능 병목 현상을 막을 수 있다.
- 사용자가 원하는 방법으로 쿼리를 실행하도록 할 수 있다.
- 클러스터의 부하를 균형 있게 분산시킬 수 있다.
- 데이터나 스카마에 접근하지 않고 쿼리를 조정할 수 있다.

## 📌 Accumulator

`Spark Accumulator` 는 여러 태스크에 걸쳐 안전하게 값을 집계할 수 있는 특별한 공유 변수이다. 클러스터의 모든 노드에서 일관성을 유지하며 집계 연산을 수행하는 데 최적화되어 있다.

스파크에서 driver의 변수를 executor에서 직접 수정할 수 없다. 이러한 문제를 해결하도자 accumulator를 사용한다.

accumulator의 동작은 다음과 같다.

1. driver에서 `sc.accumulator` 메서드를 통해 accumulator를 초기화한다.
2. accumulator를 각 워커 노드에서 실행되는 태스크에 배포한다.
3. 각 워커의 태스크는 accumulator에 `add` 메서드를 통해 값을 더할 수만 있고, 값을 읽을 수 없다.
4. 작업이 완료되면 누적된 값들이 driver로 다시 전송된다.
5. driver는 모든 워커로부터 받은 누적 값을 최종적으로 합산하여 전체 결과를 계산한다. `.value` 속성을 통해 읽을 수 있다.

### 특징

- 태스크가 실패하여 재시도하는 상황에서 값의 중복 계산을 방지하여 정확성을 보장한다.
- accumulator의 값 계산은 action 연산 내부에서 진행해야 한다. transformation 연산에 정의하면 해당 연산은 action 연산이 호출될 때마다 반복적으로 재실행되고, 부정확한 결과가 나오게 된다.

> 위 상황은 데이터프레임 또는 RDD가 캐시되지 않았을 때를 가정한다.
> 
- `AccumulatorV2` 클래스를 상속받아 커스텀 accumulator를 정의할 수 있다.

## 📌 Speculative Execution

`Speculative Execution` 은 스파크의 분산 처리 환경에서 발생하는 느리게 실행되는 작업(`Stragglers`) 문제를 해결하기 위한 메커니즘이다. 전체 job의 완료 시간은 straggler에 의해 결정되므로, S=straggler 문제를 효과적으로 해결해야 한다.

Speculative Execution이 활성화되면 straggler를 감지하고 해당 job의 중복 복사본을 다른 가능한 노드에서 동시에 실행한다.

구체적인 Speculative Execution의 동작은 다음과 같다.

1. 스파크가 stage내 job의 실행 시간을 지속적으로 모니터링하며 straggler를 감지한다.
2. straggler를 식별하는 해당 job과 동일한 복사본을 다른 가능한 워커 노드에서 새로 시작한다.
3. 기존 작업과 복사된 작업 중 먼저 완료되는 작업의 결과를 최종 결과로 채택한다. 하나의 작업이 완료되면 다른 쪽의 작업은 즉시 중단된다.

Speculative Execution을 통해 병목 현상을 효과적으로 해결할 수 있으며, 전체 job의 실행 시간을 예측 가능하고 일관되게 유지하도록 할 수 있다. 또한 다른 노드들이 대기하는 시간을 줄여 전체적인 자원 활용도를 높일 수 있다.

`spark.conf.set("spark.speculation", "true")`를 통해 Speculative Execution을 활성화한다.

`spark.speculation.quantile` s는 straggler 검사를 시작하기 전 stage 내에서 완료되어야 하는 작업의 비율이다. 예를 들어 `0.75` 로 설정하면, 전체 작업의 75%가 완료된 시점부터 straggler를 감지한다.

`spark.speculation.multiplier` 는 job이 straggler로 간주되기 위해 얼마나 느려야 하는지 결정하는 배수이다. 예를 들어 `1.5` 로 설정된다면, 모든 작업의 실행 시간의 중앙값보다 1.5배 이상 길 경우 straggler로 간주된다.

Speculative Execution이 유용한 상황은 다음과 같다.

- 클러스터를 구성하는 노드를의 하드웨어 사양이 서로 다를 때
- 네트워크 또는 시스템 부하가 수시로 변하는 클라우드 환경
- 파티션에 약간의 편향이 발생한 경우

다만 Speculative Execution은 단점도 존재하는데, 동일한 job을 중복으로 실행하므로 그만큼의 추가적인 리소스가 발생한다. 또한 straggler를 감지하고, 스케쥴링 하는 데 추가적인 오버헤드가 발생한다.

## 📌 Job Scheduling

스파크가 클러스터로부터 자원을 할당받는 방식은 크게 두 가지 방식이 존재한다.

정적 자원 할당은 스파크가 애플리케이션 시작 시점에 고정된 수의 executor를 할당받아 애플리케이션이 끝날 때까지 그 수를 유지하는 방식이다. `--num-executors` 옵션으로 해당 수만큼의 executor를 클러스터 매니저에 요청하고, 할당이 완료되면 작업을 시작한다.

할당된 자원의 수가 변하지 않으므로 작업 실행 시간을 안정적으로 예측할 수 있으며, 자원 관리를 위한 추가적인 설정이 필요없기 때문이 관리가 단순하다. 그러나 작업 부하가 적은 경우에도 자원이 대기 상태로 남아있어 리소스 낭비가 발생하며, 자원을 과도하게 할당하면 클러스터의 전체 용량을 낭비하게 된다.

```python
# 동적 할당 활성화
spark.dynamicAllocation.enabled = true

# 애플리케이션 시작 시 초기 익스큐터 수
spark.dynamicAllocation.initialExecutors = 2

# 최소 익스큐터 수
spark.dynamicAllocation.minExecutors = 2

# 가능한 최대 익스큐터 수
spark.dynamicAllocation.maxExecutors = 50

# executor를 반납하기 전까지의 대기 시간
spark.dynamicAllocation.executorIdleTimeout = 60

```

동적 자원 할당은 애플리케이션이 실행 중인 작업량에 따라 필요한 만큼 executor를 동적으로 확보하고 필요 없어지면 해제하는 방법이다. 초기에 `initalExecutors` 만큼 executor를 시작하고, 부족한 경우 최대 `maxExecutors` 까지 사용할 수 있다. 특정 executor가 `executorIdleTimeout` 동안 아무 작업도 수행하지 않으면 해당 executor를 클러스터에 반납한다. 단, 최소 executor의 수가 `minExecutors` 이하가 되지 않도록 한다.

클러스터의 전체 자원을 효율적으로 사용할 수 있으나. 새로운 executor를 할당받는 과정에서 약간의 지연이 발생할 수 있으며 YARN이나 k8s 같이 동적 자원 할당을 지원하는 클러스터 매니저 환경에서만 사용할 수 있다.

---

```python
spark.conf.set("spark.scheduler.mode", "FAIR")
```

스파크 환경에서 job에서 자원을 배분하는 스케쥴링 기법은 크게 두 가지가 있다.

FIFO 스케쥴링은 기본적인 방법으로, job이 제출된 순서대로 자원을 할당하고 실행하는 방법이다. 해당 job이 완료될 때까지 나중에 제출된 job은 대기해야 한다.

구조가 단순하고 이해하기 쉬우나 특정 job이 오랫동안 실행되면 그 뒤의 간단한 job은 자원 할당을 계속 대기해야 하는 상황(`starvation`)이 발생할 수 있다.

Fair 스케쥴링은 사용 가능한 자원을 각 job에 공정하게 분배하여 job을 병렬로 실행시키는 방법이다. `pool` 이라는 개념을 통해 job을 그룹화하고, 자원을 각 pool에 균등하게 분배한다.

짧은 job이 긴 job으로 인해 지연되는 현상을 막을 수 있다.