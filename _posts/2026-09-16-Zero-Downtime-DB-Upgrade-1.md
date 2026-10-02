---
layout: post
title:  "무중단 DB 업그레이드 - 1. 설계"
date:   2026-09-27 21:25:00 +0900
categories: dev
img-overlay: 0.1
comments: true
draft: true
series: zero-downtime-db-upgrade
---

AWS Managed DB 를 사용하다 보면, AWS 에서 DB 버전 업그레이드를 하라는 알림을 받게 됩니다. 업그레이드를 제때 하지 않으면 Extended Support 비용을 내야 하고, 그렇다고 업그레이드를 하려면 결국 다운타임이 생기기에 서비스 점검 시간을 걸게 됩니다. (Aurora 기준으로, LTS 버전이 아니라면 업그레이드를 거의 1년에 한 번씩 하게 됩니다.) 그렇기에 많은 회사에서는 새벽에 서비스 점검 시간을 가지면서 서비스 공지를 띄우고 작업을 합니다.

현재 제가 다니고 있는 [포트원](https://www.portone.io/)은 다른 회사에 결제 시스템을 제공하는 회사입니다. 그래서 DB 업그레이드를 위해 점검 시간을 걸면 포트원을 사용하는 모든 회사에 점검 시간이 생기기 때문에, 점검 시간을 걸겠다는 결정은 상당히 비싼 결정입니다. 따라서 저희는 무중단 업그레이드를 목표로 잡고 DB 업그레이드를 진행하였습니다.

# AWS RDS Blue/Green Deployments

현재 AWS 에서 제공하는 DB 업그레이드 방법 중 다운타임을 가장 최소화할 수 있는 것은 Blue/Green Deployments 입니다. 원리는 다음과 같습니다.

1. 새로운 DB (Green DB)를 만들고, 이전 DB (Blue DB)로부터 replication 을 설정하여 이전 DB 의 변경사항을 따라잡도록 합니다.
2. 새로운 DB 를 `read_only` 상태로 두고 여러 가지 테스트를 해봅니다.
3. 문제가 없을 것 같으면 새로운 DB 로 Switchover 를 진행합니다.
   1. 이전 DB 에 있는 연결들을 모두 끊습니다.
   2. 새로운 DB 가 이전 DB 의 변경사항을 모두 따라잡을 때까지 기다립니다.
   3. DB 연결에 사용되던 endpoint 의 DNS record 들을 변경하여 새로운 DB 를 가리키도록 만듭니다.

{% include viz/rds-blue-green.html %}

이 방법을 쓰면 다운타임은 최소화되지만, 일반적인 서버 프로그램에서는 다운타임을 없애기가 쉽지 않습니다. 그 이유는 총 2가지입니다.

첫 번째 이유는 이전 DB 에 있는 연결들을 모두 끊기 때문입니다. 서버에 DB 연결이 끊기더라도 자동으로 재시도하는 로직이 잘 짜여 있지 않은 이상, 진행 중인 요청 처리에 문제가 생깁니다. 하지만 재시도를 구현하려면 단순히 재시도만 하는 것이 아니라 여러 가지를 고려해야 합니다. DB 연결이 끊겼을 때 서버는 해당 transaction 의 commit 이 성공했는지 실패했는지를 구분할 수 없기 때문에, transaction 은 idempotent 해야 하고, transaction 중간의 서버 코드에 외부 side effect 가 없어야 하는 등의 원칙이 있어야만 재시도를 할 수 있습니다.

두 번째 이유는 새로운 DB 로 연결을 바꾸는 것을 DNS record 변경을 통해서 한다는 점입니다. 서버에 있는 DNS cache 때문에, cache 가 만료되기를 기다려야 합니다. cache 를 중간에 직접 비우는 방식으로 보완할 수는 있겠지만, DNS 를 통해서 DB 를 교체한다면 무중단으로 하는 것은 쉽지 않을 것입니다.

# 무중단 업그레이드 설계

AWS RDS Blue/Green Deployments 에 있는 문제들을 해결해야만 무중단 DB 업그레이드가 가능해집니다. 두 번째 이유인 DNS record 문제는 NLB 같은 Proxy 를 하나 두면 해결됩니다. 하지만 첫 번째 문제인 "이전 DB 에 있는 연결을 모두 끊는다"를 해결하기 위해서는 "DB 연결에 대해 잘 알고 있는 DB Proxy"가 필요합니다. DB Proxy 가 transaction 같은 DB 연결에 대한 정보들을 알고 있어서 DB Proxy 뒷단에 연결되어 있는 DB 만 교체할 수 있다면, 클라이언트 입장에서는 DB 변경을 인지하지 못하는 무중단 업그레이드가 가능해집니다. 무중단 업그레이드의 단계는 다음과 같습니다.

1. 새로운 DB (Green DB)를 만들고, 이전 DB (Blue DB)로부터 replication 을 설정하여 이전 DB 의 변경사항을 따라잡도록 합니다.
2. 클라이언트(서버)들은 DB Proxy 를 바라보도록 하고, DB Proxy 는 이전 DB 로 연결시켜줍니다.
3. 새로운 DB 를 `read_only` 상태로 두고 여러 가지 테스트를 해봅니다.
4. 문제가 없을 것 같으면 새로운 DB 로 Switchover 를 진행합니다.
   1. DB Proxy 에서 이전 DB 사용을 draining 합니다.
      1. 진행 중인 transaction 이 있는 연결은 해당 transaction 이 끝날 때까지 기다립니다.
      2. 클라이언트가 새로운 transaction 을 열려고 하면, DB Proxy 는 이를 뒷단 DB 에 전달하지 않고 붙잡아 둡니다.
      3. 진행 중인 transaction 이 모두 끝날 때까지 기다려 draining 을 완료합니다.
   2. 새로운 DB 가 이전 DB 의 변경사항을 모두 따라잡을 때까지 기다립니다.
   3. DB Proxy 에서 새로운 DB 를 사용하도록 합니다. (Switchover)
      1. DB Proxy 가 뒷단 DB 에 전달하지 않고 있던 query 를 이제야 새로운 DB 에 전달합니다.
      2. 새로운 DB 의 응답을 클라이언트에 정상적으로 전달하기 때문에, 클라이언트 입장에서는 단순히 query 가 느려진 것처럼 느껴집니다.

{% include viz/db-proxy-switchover.html %}

이렇게 업그레이드를 하면, draining 에 2초가 걸렸다고 했을 때 그 시점에 서버로 들어온 모든 요청의 latency 가 최대 2초 증가하긴 하지만, 서버의 응답은 정상적으로 나오기 때문에 다운타임은 생기지 않습니다. 이런 무중단 업그레이드를 하기 위해서는 서버에 대한 몇 가지 가정이 필요합니다.

1. 서버에서는 DB 에 transaction 을 길게 잡고 있지 않아야 합니다.
   - draining 할 때 기다리는 시간만큼 요청의 latency 가 길어지기 때문에, 이 시간이 장애 상황과 동일한 수준으로 길어지면 문제가 됩니다.
2. DB 에 transaction 을 잡고 있으면서 새롭게 DB transaction 을 여는 코드가 Switchover 중에 동작하면 안 됩니다.
   - 이런 코드는 새로운 transaction 이 열려야 이전 transaction 이 끝납니다. 그런데 Switchover 중의 DB Proxy 는 새로운 transaction 은 delay 시키고 기존 transaction 은 끝날 때까지 기다리기 때문에, 일종의 deadlock 이 발생하면서 draining 이 실패하게 됩니다.

이 두 가지 가정을 만족하는 서버 프로그램은 상대적으로 많을 것입니다. 또한 항상 이 두 가정을 만족해야 하는 것이 아니라 Switchover 하는 순간의 요청들만 만족하면 되기 때문에, 두 가정을 만족하지 않는 요청 종류가 드물다면 확률을 믿고 Switchover 가 성공할 때까지 시도하는 방법도 있습니다.

포트원의 경우 MySQL 과 PostgreSQL 을 모두 사용하고 있기 때문에, 무중단 업그레이드 과정을 두 DB 에 대해서 모두 설계하고 실행하였습니다. 다음 글들에서 각 DB 에서 어떻게 무중단 업그레이드를 구현했는지 설명하겠습니다.

{% include series.html title="무중단 DB 업그레이드" %}
