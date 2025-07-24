---
title : "[Spark] Kafka를 사용한 Spark Streaming"
date : 2025-07-24 20:15:00 +0900
categories : [Spark]
tags : [spark, 스파크, streaming, pyspark, kafka, 카프카]
---

## 📌 Apache Kafka란?

`Apache Kafka` 는 실시간으로 발생하는 대규모 이벤트 스트림을 효율적으로 처리하기 위해 설계된 분산 이벤트 스트리밍 플랫폼이다.

`Pub/Sub` 모델을 따른다. `Producer` 는 데이터를 생산하고 `Consumer` 는 데이터를 소비한다. 이들을 완전히 분리하여 서로 직접적으로 통신할 필요가 없도록 한다. `Topic` 은 메시지를 분류하는 채널이다. producer와 consumer는 각각 토픽에 메시지를 발행하고 구독한다.

### 특징

- 단일 서버가 아닌 여러 노드에 분산되어 동작되도록 설계되었다. kafka 클러스터를 구성하는 개별 노드를 `Broker` 라고 하며, 여러 개의 broker가 모여 하나의 카프카 클러스터를 구성한다.
- 하나의 거대한 토픽을 여러 개의 파티션으로 분할하여, 이들을 여러 브로커에 분산하여 저장한다. 만약 데이터의 양이 늘어나면 파티션의 수를 늘려 더 많은 브로커에 작업을 분산시킬 수 있다. consumer는 파티션에 대하여 병럴 처리가 가능하므로, 빠른 속도로 데이터를 처리할 수 있다. 또한 하나의 파티션 내에서는 메시지의 순서를 보장한다.
- 데이터의 유실을 방지하기 위해 각 파티션의 복제본을 다른 브로커에 저장한다.
- 수신한 메시지를 메모리가 아닌 디스크에 영구적으로 저장한다. consumer가 메시지를 읽어도 데이터를 즉시 삭제되지 않고 설정된 기간 동안 디스크에 보관된다. 이는 장애 발생 시 데이터 유실을 방지한다. 이 때문에 다른 consumer가 같은 데이터를 여러 번 읽어 사용할 수 있다.
- 데이터베이스, 검색 엔진, 데이터 웨어하우스 등과 통합이 쉽다.
- 수평적 확장이 쉽다. 클러스터의 용량이 부족해지면 클러스터에 브로커를 추가하면 된다.

---

```python
spark = SparkSession \
    .builder \
    .appName("StructuredWordCount") \
    .config("spark.streaming.stopGracefullyOnShutdown", "true") \
    .config("spark.sql.shuffle.partitions", "3") \
    .getOrCreate()

events = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "quickstart") \
    .option("startingOffsets", "earliest") \
    .load()
```

`spark.sql.shuffle.partitions` 에서 데이터 셔플링의 파티션 수를 설정한다.

`SparkSesion` 을 생성했다면 이제 카프카 데이터 스트림을 정의한다.

`format` 을 `kafka` 로 지정하여 데이터 스트림을 만든다.

`kafka.bootstrap.servers` 는 스파크가 카프카 클러스터에 접속하기 위해 필요한 브로커 목록이다. 스파크 driver와 executor는 해당 주소를 통해 브로커와 통신을 진행한다.

`subscribe` 는 어떤 카프카 토픽에서 데이터를 가져올지 지정한다.

`startingOffsets` 는 스트리밍 쿼리가 시작될 때 토픽의 어느 지점부터 데이터를 읽을지 결정한다. `earliest` 로 설정하면 토픽의 가장 오래된 데이터부터 읽는다.

```python
schema = StructType([
    StructField("city", StringType()),
    StructField("domain", StringType()),
    StructField("event", StringType())
])

value_df = events.select(
    col('key'),
    from_json(
        col("value").cast("string"), schema
    ).alias("value")
)
```

카프카를 통해 JSON 파일을 읽기 위해서, 먼저 JSON 스키마를 정의해야 한다. 이후 `from_json` 메서드를 통해 JSON 문자열을 정의한 `schema` 에 따라 파싱하여 구조체를 만든다.

```python
output_df = concat_df.selectExpr(
                'null',
"""
    to_json(named_struct("lower_city", lower_city, "domain", domain, "behavior", behavior)) as value
""".strip())
```

카프카 싱크는 기본적으로 key와 value 두 개의 컬럼을 기대한다. `named_struct` 메서드를 통해 여러 컬럼을 하나의 구조체로 묶고, `to_json` 메서드를 통해 하나의 JSON 문자열로 직렬화한다.

```python
file_writer = concat_df \
                .writeStream \
                .queryName("transformed json") \
                .format("json") \
                .outputMode("append") \
                .option("path", "transformed") \
                .option("checkpointLocation", "chk/json") \
                .start()

kafka_writer = output_df \
                .writeStream \
                .queryName("transformed kafka") \
                .format("kafka") \
                .option("kafka.bootstrap.servers", "kafka:9092") \
                .option("checkpointLocation", "chk/kafka") \
                .option("topic", "transformed") \
                .outputMode("append") \
                .start()

spark.streams.awaitAnyTermination()
```

만약 두 개 이상의 아웃풋에 데이터를 저장하는 경우 위와 같이 따로 데이터 스트림을 정의하면 된다. 다만 주의해야 할 점은, 두 스트리밍 쿼리는 논리적으로 별개의 쿼리이므로 다른 체크포인트 디렉터리를 사용해야 한다. 만약 이를 공유한다면 상호 간섭이 발생하게 되며, 예상치 못한 결과가 나오게 된다.

또한 `awaitTermination` 대신 `awaitAnyTermination` 를 사용하였는데, 스파크 세션 내에서 실행 중인 모든 스트리밍 쿼리 중 어느 하나라도 종료할 때까지 대기하는 메서드이다. 이를 통해 두 개 이상의 쿼리를 동시병렬적으로 실행할 수 있다.

## 📌 Stateless/Stateful transformation

여기서 state란 이전 데이터에 대한 정보나 중간 계산 결과를 말한다. 스트리밍 애플리케이션이 이를 기억하는지 여부에 따라 Stateless와 Stateful로 나뉜다.

`Stateless transformation` 은 각 데이터를 독립적으로 처리하는 연산이다. 현재 처리 중인 데이터 외 다른 정보는 필요하지 않다. 상태를 관리할 필요가 없기 때문에 간단하며 메모리 사용량이 적고 처리 속도가 빠르다. 각 데이터가 독립적이므로 worker 노드를 추가하여 병렬 처리 성능을 높일 수 있다. `select`, `withColumn`, `filter`, `map`, `explode` 연산 등이 있다.

전체 결과를 출력하는 `Complete` 모드는 사용할 수 없다.

`Stateful transformation` 은 과거 데이터의 컨텍스트를 유지하고 이를 기반으로 현재 데이터를 처리하는 연산이다. `groupBy().agg()` 와 같은 스트리밍 집계, `window` 메서드와 같은 윈도우 집계, 두 개의 스트림을 조인하는 연산이 대표적이다. 상태를 저장해야 하기 때문에 충분한 메모리 공간이 필수적이다. 상태 관리를 스파크가 자동으로 할 수 있지만, 개발자가 직접 상태를 관리할 수도 있다.

---

```python
events = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "pos") \
    .option("startingOffsets", "earliest") \
    .option("failOnDataLoss", "false") \
    .load()

value_df = events.select(
            col('key'),
            from_json(
                col("value").cast("string"), schema).alias("value"))

tf_df = value_df.selectExpr(
            'value.product_id',
            'value.amount')
```

위 과정은 Stateless라고 볼 수 있다. 모든 연산은 각 행을 기준으로 독립적으로 처리하기 때문이다. 다시 말해, 현재 데이터를 처리하는 데 과거의 데이터가 필요하지 않다.

```python
total_df = tf_df.select("product_id", "amount")\
    .groupBy("product_id")\
    .sum("amount").withColumnRenamed("sum(amount)", "total_amount")
```

반면 위 과정은 Stateful이다. `total_amount` 를 계산하기 위해 스파크는 상품의 판매액 합계가 얼마인지 기억하고 있어야 하기 때문이다.

`groupBy` 가 호출되면 스파크는 내부적으로 메모리 공간을 준비하고, 새로운 데이터가 들어오면 메모리에서 데이터의 키를 찾고, 상태를 업데이트한다.

## 📌 윈도우 집계 연산

크게 두 가지 윈도우 종류가 존재한다.

![image.png](assets/img/spark/5.png)

`Tumbling Window`는 데이터 스트림을 고정된 크기로 서로 겹치치 않게 구간을 나누는 방법이다. 즉, 각 윈도우 사이에는 빈틈이 없다. 각 이벤트는 하나의 윈도우에만 속한다.

```python
timestamp_format = "yyyy-MM-dd HH:mm:ss"
tf_df = value_df.select("value.*") \
                .withColumn("create_date", to_timestamp("create_date", timestamp_format))
```

`window` 메서드를 사용하기 위해 시간 기준이 되는 컬럼은 `Timestamp` 타입이어야 한다.

```python
window_duration = "5 minutes"
window_agg_df = tf_df \
    .groupBy(window(col("create_date"), window_duration)) \
    .sum("amount").withColumnRenamed("sum(amount)", "total_amount")
```

`window` 메서드를 통해 윈도우를 정의한다. `col` 메서드를 통해 어떤 시간 컬럼을 기준으로 할 지 지정하고, `window_duration` 을 통해 윈도우의 크기를 설정한다. `sliding_duration` 을 지정하지 않았으므로 자동으로 텀블링 윈도우로 동작한다.

![image.png](assets/img/spark/6.png)

`Sliding Window` 는 데이터 스트림을 고정된 크기로, 서로 겹치도록 나누는 방식이다. 즉, 하나의 이벤트가 여러 원도우에 동시에 포함될 수 있다.

두 가지 파라미터가 존재한다. ‘윈도우 크기’는 집계가 수행될 전체 구간의 길이이다. ‘슬라이드 간격’은 윈도우가 다음 위치로 이동하는 시간 간격이다. 만약 윈도우 크기와 슬라이드 간격이 같다면 윈도우는 겹치지 않게 되며, 텀블링 윈도우와 동일하게 동작한다.

```python
window_duration = "10 minutes"
sliding_duration = "5 minutes"
window_agg_df = tf_df \
    .groupBy(window(col("create_date"), window_duration, sliding_duration)) \
    .sum("amount").withColumnRenamed("sum(amount)", "total_amount")
```

`window` 메서드에 `sliding_duration` 을 지정하면 슬라이딩 윈도우로 동작한다.

## 📌 Watermark

`Watermark` 는 데이터 스트림에서 얼마나 늦게 도착하는 데이터까지 처리할 것인가를 정의하는 기준이다.

워터마크를 이해하기 위해 두 가지 시간 개념이 등장한다. `Event Time` 은 실제로 이벤트가 발생한 시간이며, `Processing Time` 은 이벤트가 스트리밍 애플리케이션에 도착하여 처리되는 시간이다. 항상 이벤트 시간 기준으로 데이터가 정렬되어 들어오는 것이 아니다. 네트워크 지연이나 시스템 부하 등 여러가지 이유가 존재한다.

워터마크는 이벤트 시간을 기준으로 동작한다. Stateful transformation에서 메모리에 중간 집계 결과를 저장할 때, 이를 정리하는 규칙이 없다면 state는 무한히 커지게 되며, OOM 오류가 발생하게 된다.

또한 워터마크는 너무 늦게 도착하는 데이터를 무시하는 기준이 되기도 한다. 지연 임계값보다 늦게 도착한 데이터는 집계에 포함시키지 않는다.

### Output Mode

출력 모드는 스트리밍 쿼리의 결과를 `sink` 에 어떻게 쓸지를 결정한다.

`Complete` 모드는 매 트리거마다 전체 집계 결과 테이블을 출력한다. 전체 결과를 보여줘야 하므로 워터마크에 의해 오래된 윈도우 상태를 정리할 수 없다. 즉, Complete 모드에서 워터마크는 늦게 온 데이터 필터링 역할만 하지, 상태 정리 역할은 수행할 수 없다. 이는 결국 OOM 오류를 유발하게 되므로, 장시간 실행되는 Stateful 스트리밍에서는 사용하면 안 된다.

`Update` 모드는 마지막 트리거 이후 결과가 변경된 행만 출력한다. 데이터가 들어와 윈도우의 집계 값이 업데이트되면 해당 윈도우의 변경된 결과가 싱크로 출력된다. 워터마크로 진행되어 특정 윈도우가 만료되면 해당 윈도우의 상태를 삭제한다. Update 모드는 기존 결과 행을 수정해야 하므로 싱크는 키를 기반으로 행를 찾아 수정해야 한다. 이를 `Upsert` 라고 한다. 따라서 키 기반 수정이 가능한 시스템에 적합하다.

`Append` 모드는 확정되어 다시는 변경되지 않을 행만 출력한다. 워터마크가 없는 경우 일반적인 집계 연산은 계산이 계속 바뀔 수 있으므로 Append 모드를 사용할 수 없다. Append 모드에서 워터마크는 특정 윈도우를 최종 확정시키는 효과를 가져온다. 원도우의 종료 시간이 워터마크를 지나면 해당 윈도우의 결과는 더 이상 변경되지 않을 것이라고 보장할 수 있다. 단, 원도우의 결과는 워터마크에 의해 만료될 때까지 기다려야 볼 수 있다.

---

```python
window_duration = "5 minutes"
window_agg_df = tf_df \
    .withWatermark("create_date", "10 minutes") \
    .groupBy(window(col("create_date"), window_duration)) \
    .sum("amount").withColumnRenamed("sum(amount)", "total_amount")
```

`withWatermark` 메서드를 통해 워터마크를 정의한다. 첫 번째 파라미터에 워터마크 계산 시 사용할 기준 시간 컬럼, 두 번째 파라미터에 지연 임계값을 설정한다.

## 📌 Streaming - Static Join w/ Cassandra

### Cassandra

`Cassandra` 는 수많은 데이터를 지연 없이 저장하고, 사용자 수가 늘어날 때마다 서버를 유연하게 추가할 수 있고, 특정 서버에 장애가 발생해도 서비스가 중단되지 않도록 하는 목표를 가지고 등장한 분산 데이터베이스이다.

카산드라 클러스터에는 특별한 역할을 하는 마스터 노드가 존재하지 않고, 모든 노드가 동등한 역할을 수행한다. 노드들은 `Gossip` 이라는 프로토콜을 통해 서로의 상태를 지속적으로 교환한다. 마스터 노드가 없으므로 어떤 노드가 다운되어도 전체 클러스터는 멈추지 않는다.

데이터가 들어오면 카산드라는 해당 데이터를 저장할 노드들을 결정하고 해당 노드들과 노드의 복제본에게 데이터를 전송한다. 고가용성을 높이기 위해 읽기 및 쓰기 요청 시 몇 개의 노드로부터 응답을 받아야 요청이 성공되었다고 간주한다.

카산드라는 수평적 확장에 최적화되어 있다. 데이터를 분산시키기 위해 `Hash Ring` 이라는 가상의 링을 사용한다. 각 노드는 링의 특정 구간을 담당하며, 데이터를 저장할 때 데이터의 PK를 해싱하여 링 위의 특정 위치 값을 얻고, 데이터는 해당 위치가 속한 구간을 담당하는 노드에 저장된다. 이 과정은 서비스 중단 없이 이루어진다.

또한 조인 연산을 지원하지 않는다. 애플리케이션이 어떻게 데이터를 조회할 것인지 설계하고, 의도적으로 비정규화하여 조회 쿼리에 최적화된 테이블을 만든다.

`Cassandra Query Language, CQL` 이라는 쿼리 언어를 사용하는데, SQL과 유사한 문법을 사용한다. 가장 큰 차이점은 `WHERE` 절의 사용이 제한적이라는 점이다. 성능을 보장하기 위해 PK에 포함된 컬럼을 조건으로만 조회할 수 있다.

---

```python
spark = SparkSession \
    .builder \
    .appName("WaterMark") \
    .config("spark.streaming.stopGracefullyOnShutdown", "true") \
    .config("spark.sql.shuffle.partitions", "3") \
    .config("spark.cassandra.connection.host", "cassandra") \
    .config("spark.cassandra.connection.port", "9042") \
    .config("spark.cassandra.auth.username", "cassandra") \
    .config("spark.cassandra.auth.password", "cassandra") \
    .config("spark.sql.extensions", "com.datastax.spark.connector.CassandraSparkExtensions") \
    .config("spark.sql.catalog.lh", "com.datastax.spark.connector.datasource.CassandraCatalog") \
    .getOrCreate()
```

`spark-cassandra-connector` 라이브러리를 사용하여 스파크와 카산드라 클러스터를 연결한다.

```python
join_df = event_df.join(
            user_df,
            event_df.login_id == user_df.login_id,
            "inner").drop(user_df.login_id)
```

스파크는 카프카로부터 마이크로 배치 단위로 이벤트를 가져온다. 각 마이크로 배치에 대해 메모리에 로드해 준 정적 데이터와 조인을 수행한다. 최종 결과는 정적 데이터가 포함된 스트리밍 데이터프레임이 된다. 이전 상태를 기억할 필요가 없으므로 Stateless transformation이다.

```python
def cassandraWriter(batch_df, batch_id):
    batch_df \
        .write \
        .format("org.apache.spark.sql.cassandra") \
        .option("keyspace", "test_db") \
        .option("table", "users") \
        .mode("append") \
        .save()
    
query = output_df \
        .writeStream \
        .foreachBatch(cassandraWriter) \
        .outputMode("update") \
        .trigger(processingTime="5 seconds") \
        .start()
```

`foreachBatch` 는 스트리밍 데이터프레임을 마이크로 배치 단위로 처리할 때 각 배치를 일반적인 정적 데이터프레임처럼 다룰 수 있게 하는 메서드이다.

## 📌 Streaming - Streaming Join

```python
join_df = impressions_df.join(
    clicks_df,
    impressions_df.uuid == clicks_df.uuid,
    "inner"
)
```

스파크는 양쪽 스트림 각각에 대해 상태 저장소를 유지한다. 조인 조건이 일치하면 두 데이터를 합친 새로운 행을 만들어 결과 스트림으로 내보낸다.

inner join을 사용하는 경우 조인 조건이 일치하는 순간 결과가 생성되지만, outer join을 사용하는 경우 언제까지 데이터를 기다리고, 해당 필드를 null로 설정할지에 대한 기준이 있어야 한다.

```python
impressions_df = impression_events.select(...) \
    .withColumnRenamed("create_date", "impr_date") \
    .withWatermark("impr_date", "5 minutes") \
    .withColumnRenamed("uuid", "impr_uuid")

clicks_df = click_events.select(...) \
    .withColumnRenamed("create_date", "click_date") \
    .withWatermark("click_date", "5 minutes") \
    .withColumnRenamed("uuid", "click_uuid")

```

이러한 문제를 방지하기 위해 양쪽 스트림에 워터마크를 설정한다.

```python
join_df = impressions_df.join(
    clicks_df,
    expr("""
        impr_uuid = click_uuid AND
        click_date >= impr_date AND
        click_date <= impr_date + interval 5 minutes
        """),
    "leftOuter"
)
```

그리고 `expr` 메서드를 통해 명시적으로 조인 조건을 세부적으로 설정한다.

정리하면, 워터마크를 통해 언제 상태를 정리해야 할지 결정할 수 있으며, `expr` 를 통한 세부 조건을 통해 언제까지의 데이터를 조인할지 결정할 수 있다.

## 📌 참고

https://www.databricks.com/blog/2017/05/08/event-time-aggregation-watermarking-apache-sparks-structured-streaming.html