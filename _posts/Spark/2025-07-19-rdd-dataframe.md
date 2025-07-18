---
title : "[Spark] Spark DataFrame과 SparkSQL"
date : 2025-07-19 01:15:00 +0900
categories : [Spark]
tags : [spark, 스파크, rdd, dataframe, sparksql, pyspark]
---

## 📌 Spark DataFrame

`Spark DataFrame` 은 RDD의 개념을 확장하여 데이터를 이름과 타입을 가진 열로 구성된 2차원 테이블로 다룰 수 있도록 한다. 데이터의 스키마를 내부적으로 가지고 있다.

### 특징

- 데이터는 이름과 타입이 정의된 열로 구성되었기 떄문에, 열의 이름을 사용하여 데이터에 접근할 수 있다.
- 스키마 정보를 가지고 있으므로 존재하지 않는 열의 이름을 사용하거나 적절하지 않은 연산을 시도하면 컴파일 단계에서 오류를 검출할 수 있다.
- `lazy evaluation` 을 지원한다. action 연산이 수행되기 전 transformation 연산은 수행되지 않는다. action 연산이 호출되는 순간 실행 plan을 살펴보고 `catalyst optimizer` 를 통해 효율적인 실행 순서 및 방법을 결정한 후 클러스터에 작업을 분배하여 실행한다.
- 여러 개의 파티션으로 나뉘어 여러 노드에 분산 저장된다. 따라서 병렬 연산이 가능하다.
- 일부 파티션이 유실되어도 기록된 실행 계획(lineage)를 통해 자동으로 해당 파티션을 복구할 수 있다.

### RDD vs. DataFrame

| 구분 | RDD | DataFrame |
| --- | --- | --- |
| 데이터 구조 | 비구조화 데이터 | (반)구조화 데이터 |
| 추상화 수준 | 저수준 | 고수준 |
| 사용 난이도 | 조금 여러움 | 쉬움 |
| 성능 | 개발자가 직접 최적화 로직 작성 | catalyst optimizer를 통해 최적화 가능 |
| 스키마 | 없음 | 있음 |

### Spark DataFrame vs. Pandas DataFrame

| 구분 | Spark DataFrame | Pandas DataFrame |
| --- | --- | --- |
| 아키텍처 | 분산 컴퓨팅 | 단일 노드 인메모리 |
| 확장성 | 수평적 확장 | 수직적 확장 |
| 실행 방식 | 지연 연산 | 즉시 연산 |
| 데이터 변경 가능 여부 | 불변성 | 가변성 |

## 📌 SparkSQL

`SparkSQL` 은 DataFrame API와 통합된 하나의 패키지로, SQL 쿼리와 데이터프레임 연산을 자유롭게 오가며 데이터를 처리할 수 있다. SQL 쿼리를 직접 데이터프레임에 실행할 수 있고, 결과 또한 데이터프레임으로 받을 수 있다.

SQL 쿼리로 무엇을 할지 정의하면 `catalyst optimizer` 가 쿼리를 분석하여 효율적인 plan을 자동으로 수립한다. JSON, Parquet 등과 같은 데이터를 다룰 수 있으며, BI 도구의 데이터 소스로도 활용될 수 있다. 또한 `Hive` 에서 사용하던 쿼리(`HiveQL`)도 지원한다.

> BI 도구란 데이터에서 의미 있는 정보나 인사이트를 찾도록 돕는 소프트웨어이다. 대표적으로 Tableau와 Power BI가 있다.
> 

## 📌 메서드

```python
from pyspark.sql import (
    Row,
    SparkSession)
from pyspark.sql.functions import col, asc, desc

def parse_line(line: str):
    fields = line.split('|') # |
    return Row(
        name=str(fields[0]),
        country=str(fields[1]),
        email=str(fields[2]),
        compensation=int(fields[3]))

spark = SparkSession.builder.appName("SparkSQL").getOrCreate()
lines = spark.sparkContext.textFile("file:///home/jovyan/work/sample/income.txt")
income_data = lines.map(parse_line)
```

`Row` 객체는 SparkSQL에서 데이터프레임의 한 행을 표현하는 객체이다. 필드 이름과 인덱스를 통해 데이터에 접근할 수 있다. `Row` 객체로 데이터프레임을 생성하면 필드 이름과 타입을 자동으로 분석하여 데이터프레임의 스키마를 자동으로 추론한다.

```python
schema_income = spark.createDataFrame(data=income_data).cache()
```

`createDataFrame` 은 데이터로부터 스파크 데이터프레임을 생성하는 메서드이다. `data` 파라미터로 RDD, 리스트와 같은 컬렉션을 받는다. 자동으로 데이터의 구조를 분석하여 스키마를 자동으로 추론한다.

`cache` 는 데이터프레임을 처음 계산한 뒤 결과를 클러스터의 메모리에 저장하는 메서드이다. 데이터프레임의 연산 속도를 높이기 위해 사용한다.

```python
schema_income.createOrReplaceTempView("income")
```

`createOrReplaceTempView` 는 데이터프레임을 SQL로 쿼리할 수 있는 view로 등록하는 메서드이다. 즉, 데이터프레임 객체를 DB 테이블처럼 취급하여 SQL 쿼리를 실행하게 할 수 있다. 동일한 이름의 뷰가 존재하면 기존 뷰를 삭제하고 덮어쓴다. 생성된 뷰는 세션 범위에서만 유효하며 스파크 세션이 종료되면 자동으로 사라진다.

```python
medium_income_df = spark.sql(
    "SELECT * FROM income WHERE compensation >= 70000 AND compensation <= 100000")
medium_income_df.show()
```

`sql` 은 스파크 세션 내에서 SQL 쿼리를 실행하고 결과를 새로운 데이터프레임으로 변환하는 메서드이다. 문자열 형태의 SQL 쿼리를 인자로 받는다. `sql` 메서드 자체는 transformation 연산이라, 실제 action 연산이 호출되기 전까지 실행되지 않는다.

`show` 는 데이터프레임의 내용을 출력하는 action 메서드이다.

```python
for income_data in medium_income_df.collect():
    print(income_data)
```

`collect` 메서드를 통해 데이터프레임의 `Row` 객체에 접근할 수 있다.

```python
data.printSchema()
```

`printSchema` 는 데이터프레임의 스키마를 트리 형태로 보여주는 메서드이다. 열의 이름과 데이터 타입, 널 허용 여부를 확인할 수 있다.

```python
data.select("name", "age").show()
```

`select` 는 데이터프레임의 특정 열을 선택하여 새로운 데이터프레임을 생성하는 transformation 메서드이다.

```python
data.filter(data.age > 20).show()
```

`filter` 는 데이터프레임에서 특정 조건을 만족하는 `Row` 만 선택하여 새로운 데이터프레임을 생성하는 transformation 연산이다. SQL의 `WHERE` 절과 동일한 역할을 수행하며, `where` 메서드와 `filter` 메서드는 이름만 다를 뿐 기능적으로 차이가 없다.

```python
df = spark.createDataFrame([
        Row(a=1,
            intlist=[1,2,3],
            mapfield={"a": "b"}
           )])

df.select(functions.explode(df.intlist).alias("anInt")).collect()
```

`explode` 는 배열이나 맵 형태의 열을 여러 개의 행으로 펼치는 메서드이다.

위 예제에서 `explode` 메서드는 리스트 형태인 `intlist` 의 각 요소 1, 2, 3에 대해 각각 새로운 행을 생성하여 데이터프레임을 구성한다.

```python
df = spark.createDataFrame([
        Row(word="hello world and pyspark")])
df.select(functions.split(df.word, ' ').alias("word")).collect()
```

`split` 메서드는 데이터프레임의 문자열 타입 열을 구분자로 나누어 배열 타입의 새로운 열로 변환하는 메서드이다. 파이썬 내장 함수인 `split` 과 유사하다.

```python
table_schema = t.StructType([
    t.StructField("country", t.StringType(), True),
    t.StructField("temperature", t.FloatType(), True),
    t.StructField("observed_date", t.StringType(), True)])
```

`StructType` 은 데이터프레임의 스키마를 프로그래밍 방식으로 정의하기 위한 객체이다. `StructType` 은 여러 개의 `StructField` 객체로 구성된 리스트이다.

`StructField` 는 데이터프레임의 단일 열에 대한 모든 정보를 담고 있으며, `name`, `dataType`, `nullable` 파라미터를 가지고 있다.

```python
f_temperature = data.withColumn(
                    "temperature",
                    (f.col("temperature") * 9 / 5) + 32)\
                .select("country", "temperature")
f_temperature.show()
```

`withColumn` 은 데이터프레임에 새로운 열을 추가하거나 수정할 때 사용하는 메서드이다. 첫 번째 인자는 새로 만들거나 수정할 열의 이름, 두 번째 인자는 해당 열에 들어갈 값을 지정한다.

```python
meta = {
    "1100": "engineer",
    "2030": "developer",
    "3801": "painter",
    "3021": "chemistry teacher",
    "9382": "priest"
}
occupation_dict = spark.sparkContext.broadcast(meta)
```

`broadcast` 메서드는 딕셔너리와 리스트같은 파이썬 변수를 브로드캐스트 변수라는 형태로 감싼다.

브로드캐스트 변수는 읽기 전용이며, 드라이버에서 클러스터의 모든 executor 노드로 데아터를 단 한 번으로 데이터를 전송할 수 있다.

만약 브로드캐스팅을 사용하지 않는다면 전송할 데이터를 네트워크를 통해 여러 노드에 반복적으로 전송해야 하며, 이는 큰 오버헤드의 원인이 된다.

그러나 브로드캐스팅을 사용하면 데이터를 한 번에 모든 executor에게 전송할 수 있으며, 각각의 executor는 데이터를 자신의 메모리에 캐시한다.

즉, `broadcast` 메서드를 통해 비싼 셔플링 연산을 방지할 수 있다.

```python
occupation_lookup_udf = f.udf(get_occupation_name)
```

`udf` 는 일반 파이썬 함수를 스파크 데이터프레임에서 사용할 수 있도록 wrapping하는 메서드이다.

다만 이렇게 변환된 UDF는 스파크 내장 함수에 비해 느리게 동작할 수 있으며, 스파크의 내장 함수를 우선적으로 사용하는 것이 좋다.

> 스파크의 JVM 환경과 파이썬 프로세스 간 데이터를 주고받는 (역)직렬화 오버헤드가 각 행마다 발생하기 때문이다.
> 

```python
data = df.groupBy("hero1")\
            .agg(
                f.collect_set("hero2").alias("connection"))\
            .withColumnRenamed("hero1", "hero")
data.show()
```

`collect_set` 은 그룹핑 후 각 그룹에 속한 특정 열의 값들을 모아 중복을 제거한 하나의 배열로 만드는 메서드이다.

`collect_list` 는 중복을 허용하여 배열을 만든다.

```python
data = data.withColumn("connection", f.concat_ws(",", f.col("connection")))
data.show()
```

`concat_ws` 는 특정 구분자를 사용하여 여러 문자열이나 배열의 원소들을 하나로 합쳐 단일 문자열로 만드는 함수이다. 첫 번째 인자는 구분자, 두 번째 인자는 합칠 대상이 되는 열이다.

```python
data.coalesce(1).write.option("header", True).csv("output")
```

`coalesce` 는 데이터프레임의 파티션 수를 줄이는 메서드이다.

`repartition` 은 파티션 수를 늘리거나 줄일 때 사용하는 메서드이다. `coalesce` 메서드는 전체 셔플링을 수행하지 않기 때문에 단순히 파티션 수를 줄이는 것이 목적이라면 `coalesce` 를 사용하는 것이 효율적이다.

```python
df.na.drop(how="any").show()
df.na.drop(thresh=2).show()
df.na.drop(subset=["salary"]).show()

df.printSchema()
```

`df.na` 는 `null`을 처리하기 위한 기능을 모아놓은 객체이다.

`na.drop` 은 `null` 이 포함된 행을 제거하여 데이터프레임을 정제하는 메서드이다. `how` 파라미터는 행을 제거할 때 사용할 기준이다. `any` 로 설정되면 행에 하나라도 `null` 이 존재하는 경우, `all` 로 설정되면 행의 모든 열이 `null` 인 경우 행을 제거한다.

`thresh` 파라미터에는 유효한 값의 최소 개수를 지정한다. `thresh=2` 는 `null` 이 아닌 값이 최소 2개 이상 있다면 행을 유지한다.

`subset` 파라미터는 `null` 여부를 `subset` 으로 지정된 열만 확인하도록 한다.

```python
df.na.fill(0).show()
```

`na.fill` 은 데이터프레임의 결측값을 사용자가 지정한 값으로 대체하는 메서드이다.

```python
df_user.join(df_salary, 
               df_user.id == df_salary.id, 
               "fullouter").show()
```

데이터프레임 역시 조인 연산이 존재한다. `join` 메서드는 두 개의 데이터프레임을 특정 키를 기준으로 결합하여 새로운 데이터프레임을 생성한다.

`how` 파라미터에 조인 유형을 명시할 수 있다. 조인 유형은 다음과 같다.

| `how` 파라미터 | 조인 결과의 행 |
| --- | --- |
| `inner` (default) | 양쪽 데이터프레임에 모두 일치하는 키의 행 |
| `leftouter`  | 왼쪽 데이터프레임의 모든 행 |
| `rightouter`  | 오른쪽 데이터프레임의 모든 행 |
| `fullouter`  | 양쪽 데이터프레임의 모든 행 |
| `leftsemi`  | 오른쪽 데이터프레임에 일치하는 키가 있는 왼쪽 데이터프레임의 행 |
| `leftanti`  | 오른쪽 데이터프레임에 일치하는 키가 없는 왼쪽 데이터프레임의 행 |
| `cross`  | 양쪽 데이터프레임의 카테시안 곱 |