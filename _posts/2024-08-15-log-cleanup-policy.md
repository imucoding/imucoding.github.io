---
layout: post
title: Log Cleanup Policy
subtitle:
excerpt_image: https://imucoding.github.io/assets/images/active.png
author: Hyunjune
categories: kafka
tags: [delete, compact, retention]
---
{% raw %}
### Log Cleanup Policy
- 카프카 브로커는 오래된 메시지를 관리하기 위한 정책을 `log.cleanup.policy`로 설정
  - topic 레벨은 `cleanup.policy`
- `log.cleanup.policy=delete`로 설정하면 closed segment 삭제함
- `log.cleanup.policy=compact`로 설정하면 segment를 key 레벨로 가장 최신 메시지만 유지하도록 재구성
- `log.cleanup.policy=[delete,compact]`로 설정 시 함께 적용

#### Log Cleanup Policy Delete 
- log.retention.hours(ms)
  - 개별 segment가 삭제되기 전 유지하는 시간 (default 7일)
  - 크게 설정 시 디스크 공간이 더 필요하고 작게 설정 시 오래된 segment 조회 불가
  - topic config는 `retention.ms`이며 기본 값은 log.retention.hours를 따름
- log.retention.bytes
  - segment 삭제 조건이 되는 파티션 단위 파일 크기 설정
  - 기본 값은 -1로 무한대
  - 적정 디스크 공간 사용량 제약을 위해 보통 설정
  - topic config는 `retention.bytes`이며 기본 값은 log.retention.bytes를 따름
- log.retention.check.interval.ms
  - segment 삭제 대상을 찾기 위한 주기

####  Log Cleanup Policy Compaction
- key 값이 null인 경우 적용 불가
- 백그라운드 thread 방식으로 별도 I/O 작업 수행하므로 추가적인 부하가 소모됨
- active segment는 compact 대상 제외
- compaction은 파티션 레벨에서 수행되며, 개별 segment 들을 새로운 segment로 재생성함

![img.png](https://imucoding.github.io/assets/images/active.png)

- `log.cleaner.enable=true`로 설정 필요 (default true)
  - 세그먼트 삭제, 압축을 자동으로 수행
  - false면 수동으로 지워야 함

![img.png](https://imucoding.github.io/assets/images/cleandirty.png)
- compaction이 적용된 segment는 clean영역, 적용되지 않은 segment는 dirty영역이라고 부름

#### Log Compaction 수행 후
- 메시지 순서는 여전히 유지
- 메시지 offset 변하지 않음
- consumer는 동일 key 가장 최신 메시지를 읽음


#### Log Cleanup, Retention 관련 설정 
```
kafka-configs --bootstrap-server localhost:9092 --entity-type topics --entity-name 토픽명 --alter
--add-config cleanup.policy=compact

kafka-configs --bootstrap-server localhost:9092 --entity-type brokers --entity-name 브로커번호 --alter
--add-config cleanup.policy=compact
```

- 적용된 설정 값 확인

```
kafka-configs --bootstrap-server localhost:9092 --entity-type brokers --entity-name 브로커번호 --all --describe
| grep {policy or retention or cleaner}
```

#### Log Compaction 수행 시점
- log.cleaner.min.cleanable.ratio
  - log cleaner가 compaction을 수행하기 위한 파티션 내의 dirty 데이터 비율 (dirty/total)
  - 기본 값은 0.5이며 값이 적을 수록 log cleaner가 더 빈번하게 compaction을 수행
  - topic config는 `min.cleanable.dirty.ratio`
- log.cleaner.min.compaction.lag.ms
  - 메시지가 생성된 후 최소한 `log.cleaner.min.compaction.lag.ms`가 지나야 compaction 대상이 됨
  - topic config는 `min.compaction.lag.ms`
- log.cleaner.max.compaction.lag.ms
  - dirty ratio 이하여도 `log.cleaner.max.compaction.lag.ms`가 지나면 compaction 대상이 됨
  - 기본 값은 무한이고 topic config는 `max.compaction.lag.ms`
- dirty ratio 변경
```
kafka-configs --bootstrap-server localhost:9092 --entity-type topics --entity-name 토픽명 --alter
--add-config min.cleanable.dirty.ratio=0.1
```

#### Compaction 수행 시 메시지의 삭제
![img.png](https://imucoding.github.io/assets/images/compaction.png)
- key는 있지만 value가 null인 경우 tombstone으로 내부적 표시
- `log.cleaner.delete.ms`가 지나면 tombstone 메시지 삭제됨


{% endraw %}


