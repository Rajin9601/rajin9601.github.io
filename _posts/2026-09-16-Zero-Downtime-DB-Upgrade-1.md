---
layout: post
title:  "무중단 DB 업그레이드 - 1. 설계"
date:   2026-09-27 21:25:00 +0900
categories: dev
img-overlay: 0.1
comments: true
draft: true
---

AWS Managed DB 를 사용하다보면, AWS 에서 DB 버전 업그레이드를 하라는 알림을 받게 됩니다. 업그레이드를 제때 하지 않으면 Extended Support 비용을 내야되고, 그렇다고 업그레이드를 하려면 결국 다운타임이 생기기에 서비스 점검 시간을 걸게 됩니다. (만약 LTS 버전이 아니라면 업그레이드를 거의 1년에 한번씩 하게 됩니다) 그렇기에 많은 회사에서는 서비스 점검시간을 새벽에 가지면서 서비스 공지를 띄우고 작업을 합니다.

현재 제가 다니고 있는 [포트원](https://www.portone.io/)은 다른 회사에 결제 시스템을 제공하는 회사이기에, DB 업그레이드를 위해 점검 시간을 걸게 되면, 포트원을 사용하는 모든 회사들에게 점검시간이 생기는 효과가 생기기에 점검시간을 걸겠다는 결정은 상당히 비싼 결정입니다. 따라서 저희는 무중단 업그레이드를 목표로 잡고 DB 업그레이드를 진행하였습니다.

# AWS RDS Blue-Green Deployments

현재, AWS 측에서 제공하는 DB 업그레이드 중 가장 다운타임을 최소화 할수 있는건 Blue-Green Deployments 입니다. 원리는 다음과 같습니다.

1. 새로운 DB (Green DB)를 만들어서 이전 DB (Blue DB)로부터 replication 을 설정하여 이전 DB의 변경사항을 따라잡도록 설정합니다.
2. 새로운 DB 를 read_only 로서 여러가지 테스트를 해봅니다.
3. 문제가 없을것 같으면 새로운 DB 로 Switchover 를 진행합니다.
   1. 이전 DB 에 있는 연결들을 모두 끊습니다.
   2. 새로운 DB 가 이전 DB 의 변경사항을 모두 따라잡을때까지 기다립니다.
   3. DB 연결에 사용되던 endpoint 의 DNS record 들을 변경하여 새로운 DB 를 가르키도록 만듭니다.

{% include viz/rds-blue-green.html %}

이 방법을 쓰면 다운 타임을 최소화는 되겠지만, 일반적인 서버 프로그램에서는 다운타임을 없애긴 쉽지 않습니다. 그 이유는 총 2가지입니다.

첫번째 이유는 이전 DB 에 있는 연결들을 모두 끊기 때문에, 서버에서 DB 연결이 끊기더라도 자동 재시도를 하는 로직이 잘짜여져 있지 않는 이상, 진행중인 요청처리에 문제가 생깁니다. 재시도를 구현하기 위해서는 서버에서 여러 원칙들을 지켜놓아야 가능하기 때문에, 아닌 프로그램들도 상당히 많을것입니다. (DB 연결이 끊겼을 때, DB에 보낸 마지막 요청이 성공했는지, 실패했는지는 DB 를 재조회하지 않는 이상 알기 힘들다. ....)
<< 여기에 왜 힘든지 설명을 쓸까 말까 고민중 >>

두번째 이유는 새로운 DB 로 연결을 바꾸는 것을 DNS Record 변경을 통해서 한다는 점입니다. 서버에 있는 DNS Cache 문제 때문에 DNS cache 가 날라가는것을 기다려야 됩니다. 어떨때는 서버 프로그램내에 DB의 endpoint 에 해당하는 IP 가 바뀌지 않는다는 가정하에 쓰여져 있는 코드가 있을 수도 있습니다. (예: sharding 을 프로그램내에서 구현하였을때)

# 무중단 업그레이드 설계

AWS RDS Blue-Green Deployment 에 있는 문제들을 해결해야만 무중단 DB 업그레이드가 가능해집니다. 두번째 이유인 DNS 레코드의 문제는 NLB 같은 Proxy 를 하나두면 해결이 됩니다. 하지만 첫번째 문제인 "이전 DB 에 있는 연결을 모두 끊는다" 를 해결하기 위해서는 새로운 컴포넌트가 필요합니다. 그것은 "DB 연결에 대해 잘 알고 있는 DB Proxy" 입니다. DB Proxy 가 transaction 같은 DB 연결에 대한 정보들을 알고 있다면, 클라이언트 입장에서는 DB Proxy 를 사용하고, DB Proxy 뒷단에 연결되어있는 DB 만 교체가 가능하다면 무중단 업그레이드가 가능해집니다. 무중단 업그레이드의 스텝은 다음과 같습니다.

1. 새로운 DB (Green DB)를 만들어서 이전 DB (Blue DB)로부터 replication 을 설정하여 이전 DB의 변경사항을 따라잡도록 설정합니다.
2. 클라이언트(서버)들은 DB Proxy 를 바라보도록 하고, DB Proxy 는 이전 DB 로 연결시켜줍니다. 
3. 새로운 DB 를 read_only 로서 여러가지 테스트를 해봅니다.
4. 문제가 없을것 같으면 새로운 DB 로 Switchover 를 진행합니다.
   1. DB Proxy 에서 이전 DB 사용을 draining 합니다.
      1. 진행중인 transaction 이 있는 connection 은 해당 transaction 이 끝날때까지 기다립니다.
      2. Client 에서 새로운 transaction 을 열려고 하면, DB Proxy 가 뒷단 DB 에 전달을 안해주고 가지고 있습니다.
      3. 진행중인 transaction 이 모두 끝날때까지 대기하여 draining 완료를 기다립니다.
   2. 새로운 DB 가 이전 DB 의 변경사항을 모두 따라잡을때까지 기다립니다.
   3. DB Proxy 에서 새로운 DB 를 사용하도록 합니다.
      1. DB Proxy 가 뒷단 DB 에 전달을 안해주고 있던 query 를 새로운 DB 에 이제서야 전달해줍니다.
      2. 새로운 DB 에서 응답이 온것을 Client 에 전달을 정상적으로 해주기 때문에, Client 입장에서 단순히 query 가 느려진 것처럼 느껴집니다.

{% include viz/db-proxy-switchover.html %}

이렇게 업그레이드를 하게 되면, draining 되는데 시간이 2초 걸렸다고 했을 때, 그때의 서버의 모든 요청의 latency 가 2초로 증가하긴 하지만 서버의 응답은 정상적으로 나올것이기에 다운타임이 생기진 않게 됩니다. 이런 무중단 업그레이드를 하기 위해서는 약간의 서버에 대한 가정들이 필요합니다.

1. 서버에서는 DB 에 transaction 을 길게 잡고 있지 않아야 합니다.
   - draining 할때 기다리는 시간이 너무 길어지면, 요청의 latency 가 그만큼 길어지기 때문에 장애상황과 동일한 수준으로 길어지면 문제가 됩니다.
2. DB 에 transaction 을 잡고 있으면서 새롭게 DB transaction 을 여는 코드가 Switchover 시, 동작하면 안됩니다.
   - Switchover 를 할때, 새로운 DB transaction 에 열려야 이전 transaction 이 끝나는데, DB Proxy 는 새로운 transaction 은 딜레이 시키고, 기존 transaction 은 끝날때까지 기다리면서 일종의 deadlock 이 발생하면서 draining 이 실패하게 됩니다.

이 두가지 가정들에 해당하는 서버 프로그램들은 상대적으로 많을것이고, 또한 항상 이 두가정을 만족해야 되는것이 아니라 Switchover 하는 순간의 요청들에 대해서만 만족하면 되는 가정들이기에 두 가정에 해당하는 request 종류가 희귀하다면, 확률을 믿고 Switchover 를 될때까지 시도를 하는 방법도 존재합니다.

포트원의 경우 MySQL, PostgreSQL DB 둘다 사용하고 있기 때문에, 무중단 업그레이드 과정을 두 DB 타입에 대해서 모두 설계해서 실행하였습니다.
