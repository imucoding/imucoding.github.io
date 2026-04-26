---
layout: post
title: Schema Registry
subtitle:
excerpt_image: https://imucoding.github.io/assets/images/driver.png
author: Hyunjune
categories: kafka
tags: [schema registry, backward, forward]
---
{% raw %}
### Schema Registry
- confluent kafka는 schema registry를 통해 schema 정보를 별도로 관리하는 기능 제공
- 토픽으로 전송되는 data의 schema는 schema registry에서 ID + version 별로 중앙 관리 되므로 레코드 별로 schema를 중복해서 전송할 필요가 없음

![img.png](https://imucoding.github.io/assets/images/schemaregistry.png)

#### Schema Registry 역할
- schema 전송 없이 레코드 값만 kafka로 전송할 수 있게 해줌
- kafka 메시지의 schema 제약 조건을 유지하면서 schema 변화에 대한 일정 수준 호환성을 제공

<br>
<hr>

### Avro 개요 및 장점
- 데이터 직렬화를 수행하는 시스템, 프레임워크
- 보다 컴팩트하고 빠른 binary 데이터 포맷 제공
- json 포맷으로 쉽게 스키마 메시지를 정의
- 다양한 데이터 타입을 제공 (string, bytes, long, float, double, boolean 뿐만 아니라 enum, arrays, maps 등의 복잡한 데이터 타입도 지원)

#### 메시지 포맷 비교
Json (schemaless)
```json
{
  "user_id": 53,
  "timestamp" : 1232144232,
  "address" : "경기도 성남시 ..."
}
```

Schema + Avro payload

![img.png](https://imucoding.github.io/assets/images/avro.png)
- avro payload는 바이너리 포맷이며 필드, 타입 정보 등이 포함되지 않음
- json + schema 보다 훨씬 빠르며 스키마 레지스트리에 최적화

Schema ID + Avro payload

![img.png](https://imucoding.github.io/assets/images/schemaid.png)


<br>
<hr>

### Avro 메시지를 Kafka Utility로 보내고 읽기
- avro 메시지는 avro 이진 파일 형태로 변환 된 후 직렬화되므로 별도의  class인 AvroSerializer 이용해야 함
- `kafka-avro-console-producer`, `kafka-avro-console-consumer` 유틸리티를 이용하면 손쉽게 avro 메시지를 kafka 토픽 메시지로 전송하고 읽을 수 있음
- `kafka-avro-console-producer`, `kafka-avro-console-consumer`는 모두 `schema registry` 기반 동작함

#### Avro와 Json의 차이
- avro는 json 포맷의 schema 정보를 가지고 payload를 byte 값으로 가진 형태이고, json은 그 자체는 schema가 없지만 JsonSerializer 사용에 스키마 설정을 하면 record에 schema와 payload가 모두 json 형식으로 붙어서 생성됨
- avro 사용 시 schema registry를 통해 schema를 따로 보관하고 schema ID + payload(byte)의 compact한 데이터로 전송이 가능

#### Avro 메시지 전송 및 읽기

```
avro 메시지 보내기

kafka-avro-console-producer --broker-list localhost:9092 --topic avro-test --property value.schema=
'
 {
    "type": "record",
    "name": "customer_short",
    "fields": [
                {"name": "customer_id", "type": "int"},
                {"name": "customer_name", "type": "string"}
              ]
 }
'
--property schema.registry.url=http://localhost:8081      

# enter 이후 메시지 전송
{ "customer_id" : 1, "customer_name" : "min"}
```


```
avro 메시지 읽기

kafka-avro-console-consumer --bootstrap-server localhost:9092 --topic avro_test 
--property schema.registry.url=http://localhost:8081 --from-beginning

# 출력
{ "customer_id" : 1, "customer_name" : "min"}
```

<br>
<hr>

### Schema Registry 기반의 Connector 설정

```
source connector 설정

"key.converter" : "io.confluent.connect.avro.AvroConverter",
"value.converter" : "io.confluent.connect.avro.AvroConverter",
"key.converter.schema.registry.url" : "http://localhost:8081",
"value.converter.schema.registry.url" : "http://localhost:8081",
```

```
sink connector 설정

"key.converter" : "io.confluent.connect.avro.AvroConverter",
"value.converter" : "io.confluent.connect.avro.AvroConverter",
"key.converter.schema.registry.url" : "http://localhost:8081",
"value.converter.schema.registry.url" : "http://localhost:8081",
```

- source connector에서 database.server.name으로 지정한 이름으로 topic이 만들어지고 여기에는 source DB의 DDL 정보가 저장됨
  - sink connector의 소비 대상 아님


#### Schema Registry 등록 주요 정보
- `subjects` 
  - schema registry에 등록된 토픽 별 (또는 토픽 레코드 유형 별) 기준 정보
    - schema 정보가 subject 안에 있음
  - subject 내의 스키마 호환성 체크
  - version은 subject 내에서 변경
  - schema가 진화할 때 동일한 subject에서 새로운 schema id와 version을 가짐
- `schemas` : schema 정보
- `config` : 전역 또는 subject 별 스키마 호환성 관련 정보 저장

<br>
<hr>

### REST API를 통한 Schema Registry 관리
- subject 및 config의 정보 추출 및 주요 속성에 대한 수정 / 변경 / 삭제 가능
- 가급적 delete는 자제. delete 수행 시 soft delete 이후 hard delete를 수행해야 완전 삭제
- config는 subject 별로 별도 설정하지 않으면 전역 config를 그대로 따름

#### 내부 토픽 _schemas 조회

```
kafkacat -b localhost:9092 -C -t _schemas -u -q | jq '.'
```
- 토픽 별 key, value에 대한 스키마 정보가 저장됨

#### schema registry에 등록된 모든 schema 정보 확인

```
http GET http://localhost:8081/schemas
```
- `_schemas`에 저장된 데이터 기반으로 가져옴
 
```json
{
  "id": 1,
  "schema" : 
  {
        "type":"record", 
        "name":"Key", 
        "namespace":"mysqlavro.oc.products",
        "fields":[{"name": "product_id", "type":"int"}],
        "connect.name": "mysqlavro.oc.products-key"
  },
  "version": 1
},

{
  "id": 2,
  "schema" :
  {
    "type":"record",
    "name":"Value",
    "namespace":"mysqlavro.oc.products",
    "fields":[{"name": "product_id", "type":"int"},
              {"name": "product_name", "type":["null","string"], "default":null},
              {"name": "product_category", "type":["null", "string"], "default":null}],
    "connect.name": "mysqlavro.oc.products-value"
  },
  "version": 1
}
```
- 만약 특정 토픽에서 스키마 정보가 변경된 경우 새로운 스키마 id 할당
- 해당 토픽 subject 명은 유지 되고, 그 안에서 버전이 바뀜 


#### schema registry에 등록된 모든 subjects 정보 확인

```
http GET http://localhost:8081/subjects
```
- subject 명은 topic 명 + '-key' 또는 '-value'로 명명됨

```json
{
  "mysqlavro-key",
  "mysqlavro-value",
  "mysqlavro.oc.customers-key",
  "mysqlavro.oc.customers-value"
          ...
}

```

#### 특정 id를 가지는 schema 정보 확인

```
http GET http://localhost:8081/schemas/ids/8
```
```json
{
  "schema" :
  {
    "type":"record",
    "name":"Value",
    "namespace":"mysqlavro.oc.products",
    "fields":[{"name": "product_id", "type":"int"},
      {"name": "product_name", "type":["null","string"], "default":null},
      {"name": "product_category", "type":["null", "string"], "default":null}],
    "connect.name": "mysqlavro.oc.products-value"
  }
}
```

#### 특정 subject의 버전 별 schema 정보 확인

```
http GET http://localhost:8081/subjects/mysqlavro.oc.customers-value/versions/1
```
```json
{
  "id": 4,
  "schema" :
  {
    "type":"record",
    "name":"Value",
    "namespace":"mysqlavro.oc.customers",
    "fields":[{"name": "customers_id", "type":"int"},
      {"name": "name", "type":"string"},
      {"name": "email", "type": "string"}],
    "connect.name": "mysqlavro.oc.customers-value"
  },
  "version": 1
}
```

#### 전역으로 등록된 config 정보 확인

```
http GET http://localhost:8081/config
```
```json
{
  "compatibilityLevel" : "BACKWARD"
}
```


#### 개별 subject 별 config 정보 확인

```
http GET http://localhost:8081/config/mysqlavro.ac.customers-value
```

<br>
<hr>

### Schema 개요와 필요성
- schema란 데이터의 구조와 제약 조건에 대한 규칙 정보를 기술한 메타 정보 
- 정해진 규칙대로 데이터를 입력 / 수정하지 않으면 잘못된 로직으로 데이터를 사용할 수 있음

#### 오픈소스 분산 DB 아키텍처의 특징
- 데이터 처리 성능 극대화를 위해 자유로운 확장이 가능한 분산 DB
- 초기 오픈 소스 분산 DB의 경우 schema가 없는 데이터 저장 구조 채택
- 현재도 NoSQL의 경우는 schema가 없는 데이터 구조로 발전

#### Schemaless 이슈
- 너무 자유로운 스키마 허용과 묵시적인 schema 추론은 복잡한 비즈니스 처리나 여러 사용자 / 팀이 개입된 데이터 처리 시 문제 발생시키기 쉬움
- sink에서 스키마 정보 없이 데이터 포맷 처리 어려움

#### NoSQL에서 스키마 변경에 따른 데이터 저장

![img.png](https://imucoding.github.io/assets/images/nosql.png)
- schemaless : 비슷한 유형의 데이터 그룹 들을 묶는 동적 스키마로 발전 필요

<br>
<hr>

### Avro 읽기 스키마와 쓰기 스키마
- 쓰기 스키마 : 애플리케이션이 데이터 전송 시 데이터를 부호화하기 위해 사용하는 스키마
- 읽기 스키마 : 애플리케이션이 데이터를 복화하하여 읽어들일 시 사용하는 스키마

![img.png](https://imucoding.github.io/assets/images/readwriteschema.png)
- 별도의 애플리케이션에서 쓰기 스키마와 읽기 스키마를 따로 가져갈 필요 있는가?
- 쓰기 스키마와 읽기 스키마가 다룰 수 있는가?

![img.png](https://imucoding.github.io/assets/images/schemachange.png)
- v1에서 v2로 스키마 변경된다면 수신 App은 당연히 동일 스키마로 변경되어야 하는가?
- 기존 수신 App은 새로운 스키마 변경을 어떻게 반영해서 프로그램 변경?
- 변경될 때마다 수신 App 적용하면 스키마 일관성은 어떻게 유지?

#### 호환성 없는 스키마 발전에 따른 메시지 변화

![img.png](https://imucoding.github.io/assets/images/messagechange.png)
- 수신 스키마가 변경되는 경우 호환이 가능하다면 읽으면서 차이 해소

#### Avro의 읽기 스키마와 쓰기 스키마 호환성
- avro는 읽기 스키마와 쓰기 스키마가 동일하지 않아도 되며 호환 가능하기만 하면됨
- 데이터를 복호화 (읽기) 할 때 avro는 쓰기 스키마와 읽기 스키마가 호환 가능하다면 읽어들이면서 그 차이를 해소
- 속성의 순서가 달라도 상관 없음
- 쓰기 스키마에는 있지만 읽기 스키마에는 없는 속성은 읽어 들일 때 속성 무시
- 쓰기 스키마에는 없지만 읽기 스키마에는 있는 속성은 읽어 들일 때 읽기 스키마에 선언된 기본 값으로 읽어들임


<br>
<hr>


### 스키마 호환성과 전체 메시지 일관성
- 스키마 호환성 지원을 통해 유연하게 상황 대처 가능 vs 스키마 호환성을 통해 전체 메시지는 데이터 일관성이 깨질 수 있음

#### 하위 호환성 (BACKWARD)
- 새로운 버전의 읽기 스키마는 예전 버전의 쓰기 스키마를 처리할 수 있음

case 1) 예전 버전에서 속성이 추가된 새로운 버전의 읽기 스키마는 기본 값으로 해당 속성을 읽음 <br>
따라서 읽기 스키마에 새로운 버전으로 속성이 추가될 경우 반드시 기본 값 설정이 필요

![img.png](https://imucoding.github.io/assets/images/backward1.png)

case 2) 예전 버전에서 속성이 삭제된 새로운 버전의 읽기 스키마는 해당 속성을 무시 <br>

![img.png](https://imucoding.github.io/assets/images/backward2.png)

#### 상위 호환성 (FORWARD)
- 예전 버전의 읽기 스키마는 새로운 버전의 쓰기 스키마를 처리할 수 있음

case 1) 새로운 버전 쓰기 스키마에 신규 속성이 추가되는 경우 예전 버전의 읽기 스키마는 해당 속성을 무시 <br>

![img.png](https://imucoding.github.io/assets/images/forward1.png)

case 2) 새로운 버전 쓰기 스키마에 기존 속성이 삭제되는 경우 예전 버전의 읽기 스키마는 해당 속성의 기본 값으로 읽어들임 <br>
따라서 삭제되는 기존 속성은 반드시 예전 버전에서 기본 값을 가지고 있어야 함

![img.png](https://imucoding.github.io/assets/images/forward2.png)

#### Avro 스키마 호환성 체크
- 새로 추가되는 컬럼이 기본 값을 가지고 있지 않거나 (하위 호환성 오류) 삭제되는 컬럼이 기본 값을 가지고 있지 않은 경우 (상위 호환성 오류) 호환성 오류 발생
- 기존 컬럼 타입 변경, 기존 컬럼 명 변경은 스키마 호환성 체크가 제대로 되지 않음

<br>
<hr>

### Subject 개요
- schema registry에 등록된 토픽 별 기준 정보 (schema namespace)
- subject 내의 스키마 호환성 체크
- version은 subject 내에서 변경
- schema가 진화할 때 동일한 subject에서 새로운 schema id와 version을 가짐

#### Subject의 Naming 유형
- `TopicNameStrategy`
  - subject의 이름을 정하는 기본 설정
  - 개별 토픽 별 토픽 key / value에 따라 스키마를 등록 
    - ex) oc.customer의 토픽 명인 경우 oc.customer-key, oc.customer-value
  - 토픽 내의 메시지 레코드가 동일한 스키마를 따르도록 설정
  - 스키마 호환성이 설정된 경우 해당 호환성에 적합하지 않은 스키마를 가지는 메시지는 토픽에 저장되지 않음
- `RecordNameStrategy`
  - 특정 레코드 타입별로 스키마를 따르도록 설정
  - 하나의 토픽 내 여러 타입의 이벤트가 있을 경우 이벤트 유형별로 스키마를 지정할 때 사용
    - event라는 토픽에 click, page_view 두 가지 이벤트 타입에 대한 레코드가 있을 수 있음
  - kafka 전체에서 특정 레코드 타입별로 스키마를 설정하므로 서로 다른 토픽에서 동일한 레코드 타입들이 있을 경우에도 동일 스키마가 적용됨
- `TopicRecordNameStrategy`
  - RecordNameStrategy와 유사하지만 토픽 내의 특정 레코드 타입 별로 스키마를 따르도록 설정
  - 개별 토픽의 특정 레코드 타입별로 스키마를 설정하므로 서로 다른 토픽에서 동일한 레코드 타입들이 있을 경우에는 서로 다른 스키마가 적용됨

#### Subject 스키마 호환성
- subject 내의 스키마 호환성 (compatibility) 체크

![img.png](https://imucoding.github.io/assets/images/subjectevolve.png)
- id는 key, value 따로 관리하지 않음, 스키마 저장 / 변경 시 새로 할당
- value의 schema 변경 되어 새로운 id가 할당되고 version이 업데이트 됨
- 스키마 호환성은 schema id 레벨로 체크되지 않고 subject 레벨로 체크됨

<br>
<hr>


### 스키마 발전과 호환성
- 업무가 계속 변함에 따라 스키마도 변경
- schema registry는 subject 단위로 변경된 스키마의 버전과 호환성을 관리
- schema registry에 subject 별로 호환성이 설정되어 있다면 해당 호환성에 맞지 않는 스키마 변경은 허용되지 않음

#### Schema Registry 기반의 Producer와 Consumer
![img.png](https://imucoding.github.io/assets/images/schemaproducerconsumer.png)
- schema registry를 이용한 producer와 consumer 애플리케이션은 별도의 스키마를 개별적으로 가지고 있음
- 스키마가 변경됨에 따라 producer 또는 consumer가 가지고 있는 스키마를 개별적으로 update 할 수 있음
- schema registry에 스키마에 대해 호환성이 설정되면 해당 호환성에 부합하는 스키마 변경이 되었을 때만 producer와 consumer 애플리케이션 스키마 update 가능



{% endraw %}
