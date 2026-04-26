---
layout: post
title: Kafka Producer
subtitle: ''
excerpt_image: https://imucoding.github.io/assets/images/producer.png
author: Hyunjune
categories: kafka
tags: [producer, idempotence, partitioner]
---
{% raw %}
### Producer 개요
- Producer는 Topic에 메시지를 보냄(메시지 write)
- Producer는 성능/로드밸런싱/가용성/업무 정합성등을 고려하여 어떤 브로커의 파티션으로 메시지를 보내야 할지 전략적으로 결정됨

#### Producer Record
![img.png](https://imucoding.github.io/assets/images/producerrecord.png)



#### simple producer
```java
String topicName = "simgple-producer";
Properties props = new Properties();
props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.56.101:9092");
props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

KafkaProducer<String,String> kafkaProducer = new KafkaProducer<String,String>(props);
ProducerRecord<String,String> producerRecord = new Producer<>(topicName, "hello");

kafkaProducer.send(producerRecord);
kafkaProducer.flush();
kafkaProducer.close();
```


<br>
<hr>

### Serialized Message 전송

#### kafka에서 제공하는 serializer
  - StringSerializer
  - ShortSerializer
  - IntegerSerializer
  - DoubleSerializer
  - LongSerializer
  - BytesSerializer

#### 자바 Object Serialization
- 객체의 유형, 데이터의 포맷, 적용 시스템에 상관없이 이동/저장/복원을 자유롭게 하기 위해서 바이트 스트림으로 저장하는 것

#### Custom 데이터 전송
- kafka 기본 지원 serializer는 String,Long 등의 primitive 기반 object에 대해서만 지원

Serializer 구현
```java
public class OrderSerialzer implements Serializer<Order> {
    ObjectMapper objectMapper = new ObjectMapper().registerModule(new JavaTimeModule());

    @override
    public byte[] serializer(String topic, Order order) {
        byte[] serializedOrder = null;
        try {
            serializedOrder = objectMapper.writeValueAsBytes(order);
        } catch (JsonProcessingException e) {
          ...
        }
        return serializedOrder;
    }
}
```
- Java 8의 LocalDateTime은 Jackson에서 기본 지원하지 않음 
- 따라서 `com.fasterxml.jackson.datatype.jackson-datatype-jsr310`을 추가 import하고 JavaTimeModule을 register 해야함

OrderSerdeProducer
```java
props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, OrderSerializer.class.getName());

KafkaProducer<String, Order> kafkaProducer = new KafkaProducer<String, Order>(props);
ProducerRecord<String,Order> producerRecord = new ProducerRecord<>(topic, key, value);
kakfaProducer.send(producerRecord);
```

<br>
<hr>


## Producer 메시지 파티셔닝

### key 값을 가지지 않는 메시지 전송
- 메시지는 producer를 통해 전송 시 partitioner를 통해 어떤 파티션으로 전송되어야 하는지 미리 결정됨
- key 값을 가지지 않는 경우 Round Robin, Sticky Partitioning 등의 전략이 선택되어 파티션 별로 메시지가 전송됨
- Topic이 여러 개의 파티션을 가질 때 `전송 순서가 보장되지 않은 채`로 consumer에서 읽혀질 수 있음


#### 라운드 로빈
- kafka 2.4 이전 default 파티셔닝 전략
- 최대한 메시지를 파티션에 균등하게 분배하려는 전략으로써 메시지 배치를 순차적으로 다른 파티션으로 전송
- 메시지가 배치 데이터를 빨리 채우지 못하면서 전송이 늦어지거나 배치를 다 채우지 못하고 전송하면서 성능 이슈

![img.png](https://imucoding.github.io/assets/images/roundrobin.png)

#### 스티키 파티셔닝
- kafka 2.4 이후 default 파티셔닝 전략
- 라운드 로빈의 성능을 개선하고자 특정 파티션으로 전송되는 하나의 배치에 메시지를 빠르게 먼저 채워서 보내는 방식

![img.png](https://imucoding.github.io/assets/images/sticky.png)


### key 값을 가지는 메시지 전송
- 특정 key 값을 가지는 메시지는 `특정 파티션으로 고정 전송됨`
- 특정 key 값을 가지는 메시지는 단일 파티션 내에서 전송 순서가 보장되어 consumer에서 읽힘
  - 하나의 파티션에서만 메시지 순서 보장

![img.png](https://imucoding.github.io/assets/images/producerkey.png)

#### key 메시지 전송
- 전송
```
kafka-console-producer --bootstrap-server localhost:9092 --topic 토픽명 --property key.separator=: --property parse.key=true
```
- 수신
```
kafka-console-consumer --bootstrap-server localhost:9092 --topic 토픽명 --property print.key=true --from-beginning
```

### DefaultPartitioner
- kafka는 기본적으로 DefaultPartitioner를 통해 partition 지정
- key를 가지는 메시지의 경우 key 값을 해싱하여 파티션별로 균일하게 전송

```java
public int partition(String topic, Object key, bytes[] keyBytes, Bytes[] valueBytes, Cluster cluster, int numPartition)
{
    if(keyBytes == null){
        return StickyPartitionCache.partition(topic,cluster);    
    }
    return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartition;
}
```

### Custom Partitioner 구현
- Partitioner interface 구현, partition() 메소드 구현 필요
```java
public class CustomPartitioner implements Partitioner {
  public static final Logger logger = LoggerFactory.getLogger(CustomPartitioner.class.getName());
  private final StickyPartitionCache stickyPartitionCache = new StickyPartitionCache();
  private String specialKeyName;

  @Override
  public void configure(Map<String, ?> configs) {
    specialKeyName = configs.get("custom.specialKey").toString();
  }

  @Override
  public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    List<PartitionInfo> partitionInfoList = cluster.partitionsForTopic(topic);
    int numPartitions = partitionInfoList.size();
    int numSpecialPartitions = (int)(numPartitions * 0.5);
    int partitionIndex = 0;

    if (keyBytes == null) {
      //return stickyPartitionCache.partition(topic, cluster);
      throw new InvalidRecordException("key should not be null");
    }

    if (((String)key).equals(specialKeyName)) {
      partitionIndex = Utils.toPositive(Utils.murmur2(valueBytes)) % numSpecialPartitions;
    }
    else {
      partitionIndex = Utils.toPositive(Utils.murmur2(keyBytes)) % (numPartitions - numSpecialPartitions) + numSpecialPartitions;
    }
    logger.info("key:{} is sent to partition:{}", key.toString(), partitionIndex);

    return partitionIndex;
  }

  @Override
  public void close() {

  }
}
```
- `configs.get("custom.specialKey").toString();` 
  - producer에 등록한 config 값을 가져올 수 있음
<br>
<hr>

### send() 메소드 호출 프로세스

![img.png](https://imucoding.github.io/assets/images/producer.png)
- Kafka Producer 전송은 Producer Client의 별도 Thread가 전송을 담당한다는 점에서 기본적으로 Thread간 Async 전송
- Main Thread가 send( ) 메소드를 호출하여 메시지 전송을 시작하지만 바로 전송되지 않으며 내부 Buffer에서 토픽 파티션에 따라 Record Batch 단위로 묶인 뒤 전송됨
- producer client의 내부 메모리 (**Record Accumulator**)에 여러 개의 batch들로 buffer.memory 설정 사이즈만큼 보관 가능
- 복수 개의 batch 한꺼번에 전송 가능
- ``batch.size``만큼 레코드가 쌓이면 sender thread를 통해 전송
- ``linger.ms``를 줄이면 batch 사이즈를 다 채우기 전에 전송할 수 있도록 할 수 있음 (20ms 이하 권장)

#### max.inflight.requests.per.connection
- 한 번에 전송 가능한 메시지 배치 개수 (default=5)
- 1보다 큰 경우 일부 배치의 전송 실패 시 순서 뒤집힐 수 있음
![img.png](https://imucoding.github.io/assets/images/outorder.png)

<br>
<hr>

### 메시지 동기식/비동기식 전송
#### Sync 방식
- Producer는 broker로부터 해당 메시지를 성공적으로 받았다는 Ack 메시지를 받은 후 메시지 전송
- 동기 처리시 1개씩 전송하며 ack를 받아야 하므로 batch 처리 불가능, 전송은 batch 레벨이지만 메시지는 1개

```java
Future<RecordMetadata> future = producer.send();
RecordMetadata metadata = future.get();

# inline
RecordMetadata metadata = producer.send().get();
```

<br>

#### Async 방식
- Callback을 이용한 비동기 메시지 전송 가능
  - 비동기적으로 메시지를 보내면서 RecordMetadata를 client가 받을 수 있는 방식 제공
- Async 전송 시 batch를 통해 전송함

```java
producer.send(producerRecord, new Callback() {
  @Override
  public void onCompletion( RecordMetadata metadata, Exception exception ) {
    if (exception == null) {
      System.out.println("received metadata \n" +
        "Topic:" + metadata.topic() + "\n" +
        "Partition:" + metadata.partition() + "\n" +
        "Offset:" + metadata.offset());
    } else {
        exception.printStackTrace();
    }
  }
```

<br>

#### Custom Callback 생성
- Callback 인터페이스 상속

```java
public class CustomCallback implements Callback{
  @Override
  public void onCompletion( RecordMetadata metadata, Exception exception ) {
      if (exception == null) {
          System.out.println("received metadata \n" +
                  "Topic:" + metadata.topic() + "\n" +
                  "Partition:" + metadata.partition() + "\n" +
                  "Offset:" + metadata.offset());
      } else {
          exception.printStackTrace();
      }
   }
}

```

<br>
<hr>

### 메시지 전송/재전송 시간 파라미터 이해
![img.png](https://imucoding.github.io/assets/images/retry.png)
- acks = 1 or all 인 동기식 전송에 적용됨
- max.block.ms
  - send() 호출 시 RecordAccumulator 입력이 block되는 최대 시간, 초과 시 TimeoutException
- linger.ms
  - sender Thread가 RecordAccumulator에서 배치 별로 가져가기 위한 최대 대기 시간
- request.timeout.ms
  - 전송에 걸리는 최대 대기 시간
  - 초과 시 retry 또는 timeout
- retry.backoff.ms
  - 재전송을 위한 대기 시간
- delivery.timeout.ms
  - 재전송을 포함하여 producer 메시지 전송에 허용된 최대 시간
  - 초과 시 TimeoutException
- `delivery.timeout.ms >= linger.ms + request.timeout.ms`

#### retries와 delivery.timeout.ms
```
retries = 2147483647 (MAX INT)
delivery.timeout.ms = 120000
```
- retries는 재전송 횟수를 결정
- delivery.timeout.ms는 메시지 재전송을 멈출 때까지의 시간
- 보통 retries는 무한대 값으로 설정하고 delivery.timeout.ms (기본 2분)를 조정하는 것을 권장

#### retries와 request.timeout.ms, retry.backoff.ms
```
retries = 10
retry.backoff.ms = 30
request.timeout.ms = 10000ms
```
- retry.backoff.ms는 재전송 주기 시간을 결정
- 위와 같이 설정 시 전송 후 10000ms 기다린 후 재전송 전 30ms 이후 재전송 시도
- 이와 같이 10회 수행 후 더 이상 retry 수행하지 않음
- 만약 10회 이내에 delivery.timeout.ms 도달 시 더 이상 재전송 하지 않음

<br>
<hr>

### producer 전송 방법

#### 최대 한 번 전송 (at most once)
- ack=0
- producer는 broker로부터 받는 ack 또는 에러메시지 없이 다음 메시지를 연속적으로 보냄
- 메시지는 소실될 수 있지만 중복 전송하지 않음

#### 적어도 한 번 전송 (at least once)
- acks=1,all, retries>0
- producer는 broker로부터 ack를 받은 다음에 다음 메시지 전송
- 메시지 소실은 없지만 중복 전송 가능
  - 실제로 broker에 전송 되었으나 ack만 전송되지 못한 경우

#### 정확히 한 번 전송 (exactly once)
- 정확히 한 번 전송 (트랜잭션 처리) != 중복 없이 전송 (멱등성)
- 중복 없이 전송 (Idempotence)
  - producer는 producer ID와 sequence를 Header에 담아 전송
  - 메시지 sequence는 0부터 시작하여 순차적으로 증가
  - producer ID는 producer 기동 시마다 새롭게 생성됨
  - 브로커에서는 만약 sequence가 중복인 경우 로그에 기록하지 않고 ack만 전송
  - 브로커는 자신이 가지고 있는 메시지의 sequence보다 1만큼 큰 경우에만 브로커에 저장

![img.png](https://imucoding.github.io/assets/images/nodup.png)


#### Idempotence 설정
- enable.idempotence=true
- acks=all
  - acks=1이면 leader가 복제 중 다운되면 메시지 소실
- retries는 0보다 큰 값
- max.in.flight.requests.per.connection은 1에서 5사이
  - producer가 전송 중인 메시지 상태를 추적해야 하므로 관리를 위함
  - 메시지 순서 바뀌는 것도 보장해야 하므로 상태 추적 필요
- kafka 3.0부터 producer 기본 설정이 idempotence
  - idempotence 적용 시 성능이 감소할 수 있지만 적용 권장
- 만약 기본 설정인 enable.idempotence=true를 유지하고 다른 파라미터를 잘못 설정하면 (예를 들면 acks=1) producer는 메시지 정상적으로 보내지만 idempotence가 적용되지 않음
- 하지만 명시적으로 enable.idempotence=true를 선언한 뒤에 다른 파라미터 설정 바꾸면 오류 발생

#### Idempotence 기반에서 메시지 전송 순서 유지
![img.png](https://imucoding.github.io/assets/images/idempotence.png)

주의
- Idempotence는 producer와 broker 사이 retry 시에만 중복 제거를 수행하는 메커니즘
- 동일 메시지를 send()로 재전송 하는 것은 producer가 재기동되어 PID가 달라지므로 적용되지 않음



{% endraw %}
