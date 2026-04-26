---
layout: post
title: Kafka Cluster
subtitle:
excerpt_image: https://imucoding.github.io/assets/images/cluster.png
author: Hyunjune
categories: kafka
tags: [cluster, aws]
---
{% raw %}
### Multi Broker Cluster 구축
![img.png](https://imucoding.github.io/assets/images/cluster.png)

- 각 인스턴스 모두 동일 VPC에 속해야 함
- 보안 그룹 설정
  - 포트 범위 (0-65535) 소스 (172.31.0.0/16)
  - 포트 범위 (0-65535) 소스 (허가 IP 지정)
- local에서 admin 접근은 public IP로 접근
- DNS 설정
  - 각 노드의 `/etc/hosts` 파일 수정
```
172.31.3.209 client.foo.bar client
172.31.0.186 kafka1.foo.bar kafka1
172.31.12.195 kafka2.foo.bar kafka2
172.31.3.173 kafka3.foo.bar kafka3
172.31.5.59 zookeeper1.foo.bar zookeeper1
172.31.11.46 zookeeper2.foo.bar zookeeper2
172.31.8.78 zookeeper2.foo.bar zookeeper2
```
- ping으로 확인 
  - `ping -c 2 kafka1.foo.bar`

<hr>

### Zookeeper 설정 파일 수정
- 모든 주키퍼 노드의 `zookeeper.properties` 파일에 아래 추가
```
server.1 = zookeeper1.foo.bar:2888:3888
server.2 = zookeeper2.foo.bar:2888:3888
server.3 = zookeeper3.foo.bar:2888:3888
tickTime = 3000
initLimit = 10 
syncLimit = 10
```
- `tickTime` : 주키퍼 시간 단위 (ms), `initLimit` : 주키퍼에 연결하는 시간, `syncLimit` : 주키퍼 동기화 하는시간
- zookeeper.properties 설정 중 `dataDir`에 적힌 경로에 myid라는 파일 생성
- 각 노드에서 노드 1은 1, 노드 2는 2, 노드 3은 3 저장

<hr>

### Kafka 설정 파일 수정

- 모든 노드에서 `server.properties` 각각 수정
```
broker.id = 1
listeners = PLAINTEXT://:9092
advertiesd.listeners = PLAINTEXT://kafka1.foo.bar:9092
zookeeper.connect = zookeeper1.foo.bar:2181,zookeeper2.foo.bar:2181,zookeeper3.foo.bar:2181
```
```
broker.id = 2
listeners = PLAINTEXT://:9092
advertiesd.listeners = PLAINTEXT://kafka2.foo.bar:9092
zookeeper.connect = zookeeper1.foo.bar:2181,zookeeper2.foo.bar:2181,zookeeper3.foo.bar:2181
```
```
broker.id = 3
listeners = PLAINTEXT://:9092
advertiesd.listeners = PLAINTEXT://kafka3.foo.bar:9092
zookeeper.connect = zookeeper1.foo.bar:2181,zookeeper2.foo.bar:2181,zookeeper3.foo.bar:2181
```

- kafka-ui를 docker로 실행 시 docker는 /etc/hosts를 host에서 읽어가지 않으므로 `--add-host` 옵션으로 붙여줘야 함

{% endraw %}
