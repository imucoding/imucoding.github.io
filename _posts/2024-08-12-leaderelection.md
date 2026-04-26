---
layout: post
title: Preferred, Unclean Leader Election
subtitle:
excerpt_image: https://imucoding.github.io/assets/images/insync.png
author: Hyunjune
categories: kafka
tags: [preferred, unclean, leader election]
---
{% raw %}
### Preferred Leader Election
- 파티션 별로 최초 할당된 leader/follower broker 설정을 preferred broker로 그대로 유지
- broker가 shutdown 후 재기동 될 때 preferred leader broker를 일정 시간 이후에 재선출
- `auto.leader.rebalance.enable=true`로 설정하고, `leader.imbalance.check.interval.seconds`를 일정 시간으로 설정 (기본 300초)

![img.png](https://imucoding.github.io/assets/images/normal.png) <br>
![img.png](https://imucoding.github.io/assets/images/preferred.png)
- preferred leader election 적용 시 broker가 shutdown되면 leader는 변함 없지만 ISR은 변함

<br>
<hr>

### Unclean Leader Election
- broker#2, #3 모두 shutdown되고 시간이 지나면 partition #1,#2,#3의 leader broker는 broker#1이 됨
- broker#1에 메시지가 추가로 계속 유입된 후 broker#1까지 shutdown되면 broker#2,#3이 재기동되어도 leader 될 수 없음
- 기존의 leader broker가 오랜 기간 살아나지 않는 경우 복제가 완료되지 않은 follower broker가 leader가 될 지 결정해야 함
- 메시지 손실 감수하고 복제 완료되지 않은 follower가 leader가 되려면 `unclean.leader.election.enable=true` 설정


{% endraw %}
