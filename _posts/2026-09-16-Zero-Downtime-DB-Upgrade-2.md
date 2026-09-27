---
layout: post
title:  "무중단 DB 업그레이드 - 2. MySQL 편"
date:   2026-09-27 21:25:00 +0900
categories: dev
img-overlay: 0.1
comments: true
---

# MySQL DB Upgrade

포트원의 MySQL DB 는 5.5 로 상당히 오래된 버전을 사용하고 있었기에 8.0 으로 업그레이드를 진행하였습니다. (참고: 블로그 글은 지금 쓰지만 사실 작년 초에 진행하였습니다.)

## 새로운 DB 만들기

이전 DB 와 연결되어 있는 새로운 버전의 DB 를 만드는 것은 AWS Managed RDS 나 Aurora 를 사용한다면 비교적 쉽게 가능합니다. 포트원의 경우 EC2 에 직접 운영되고 있던 MySQL DB 가 있었기 때문에 xtrabackup 을 통해 새로운 DB 를 만들고, 해당 DB 를 5.5 로부터 minor 버전을 하나씩 올려가며 8.0 까지 업그레이드를 진행하였습니다. minor 버전 업그레이드를 한 버전씩 올려가며 Changelog 를 보며 Breaking Change 들을 파악해가며 문제가 없도록 DB Column type 조작, System Variable 변경, charset 변경 등을 진행하였습니다. (해당 과정은 다른 팀원이 담당해주셨기에 자세한 내용은 적지 않습니다) 업그레이드 된 DB 에서 서버에 문제가 없는지, 퍼포먼스에는 문제가 없는지 등등을 서버 개발자가 확인을 해가며 필요한 서버 작업들을 해주셨습니다. 

## 번외 : DB 사용하는 곳 트래킹 하기

해당 MySQL DB 는 인프라의 최신화가 되기 전부터 사용되던 DB 이고, 포트원에 레거시 시스템이 존재했기 때문에 해당 DB 를 어디서 사용하고 있는지 불분명한 부분들이 있어 해당 DB 를 사용하는 곳들을 어느 서버들인지 확실하게 트래킹해볼 필요가 있었습니다. 따라서 init_connect 와 procedure 를 통해, 연결이 생길 때마다 MySQL table 에 흔적을 남기도록 하였습니다. show processlist 에는 현재 존재하는 연결만 보이기 때문에, 만약 어느 서버에서 DB 연결이 필요할때만 연경를 맺고 사용하는 형태로 DB 를 사용하고 있었다면(Connection Pool 같은 것이 없이), `show processlist` 만으로는 확신하기가 어려웠습니다.

<details>
<summary>구체적인 MySQL query 문들</summary>

```sql
-- init_connect 는 super user 는 실행이 안되기 때문에, root 빼고는 다 revoke 하자
-- super 권한을 없애주는 쿼리 뽑아내기. 뽑은걸 실행한다 
SELECT CONCAT("Revoke super on *.* from'", user, "'@'", host, "';") AS query
  FROM mysql.user
 WHERE Super_priv != 'N';

-- 연결이 생길때마다 트래킹해줄 테이블
CREATE TABLE login_tracking (
  user VARCHAR(16)
, host VARCHAR(60)
, ts TIMESTAMP
, PRIMARY KEY (user)
);

-- login_tracking 테이블에 자신의 연결정보를 넣어주는 procedure
DELIMITER //

CREATE PROCEDURE login_trigger()
SQL SECURITY DEFINER
BEGIN
  INSERT INTO login_tracking (user, host, ts)
  VALUES (SUBSTR(USER(), 1, instr(USER(), '@')-1), substr(USER(), instr(USER(), '@')+1), NOW())
  ON DUPLICATE KEY UPDATE host = substr(USER(), instr(USER(), '@')+1), ts = NOW();
END;

//
DELIMITER ;

-- procedure 실행 권한을 주는 쿼리 뽑아내는 방법. 뽑은걸 실행한다.
SELECT CONCAT("GRANT EXECUTE ON PROCEDURE {db_name}.login_trigger TO '", user, "'@'", host, "';") AS query
  FROM mysql.user
 WHERE Super_priv = 'N';

-- 각 user 로 로그인해서 문제없이 procedure 실행되는지 확인
Call {db_name}.login_trigger();

-- 잘 기록되었느지 확인해보기
select * from {db_name}.login_tracking order by ts desc limit 100;

-- init_connect 수정
show global variables like "init%"; 
SET GLOBAL init_connect="SET NAMES utf8; CALL {db_name}.login_trigger()"; 

```

</details>

## ProxySQL

위에서는 DB 에 대해서 잘 아는 DB Proxy 가 필요하다 라는 정도로만 설명하였지만, DB Proxy 가 무중단 업그레이드의 핵심입니다. 연결을 draining 하는 기능, 클라이언트와 DB Proxy 사이의 연결은 유지하면서 뒷단의 DB Proxy 와 MySQL DB 의 연결은 교체하는 기능이 있는 Proxy 가 있어야 되고, 해당 Proxy 의 안정성에 대한 신뢰가 있는 DB Proxy 가 필요합니다. MySQL 에서는 ProxySQL 이 모든 조건을 충족하였기에 ProxySQL 을 사용하기로 결정하였습니다. 하지만 ProxySQL 을 사용하는 것만으로는 원했던 draining 이 안되었고, 여러 설정들을 해줘야 되었습니다. 그 중, 가장 중요한 설정 2가지를 설명하고자 합니다.

### 1. Multiplexing

무중단 업그레이드 설계를 보면, 클라이언트와 ProxySQL 사이의 연결(앞단 연결)이 정상적인 상태에서 ProxySQL 과 MySQL DB 사이의 연결 (뒷단 연결)이 바뀌어야 합니다. 그래야지 서버(클라이언트)는 아무일이 없는것처럼 느껴지는 상태에서 뒤의 DB 가 바뀔수 있습니다. 하나의 앞단 연결에서 transaction 단위로 ProxySQL 뒷단의 연결을 마음대로 바꿀수 있는 기능을 ProxySQL 에서는 Multiplexing 이라고 합니다. 

Multiplexing 이 되기 위해서는 앞단 연결에서 주고받는 query 에 connection 단위로 작동하는 개념이 존재하는 query 가 존재하면 안됩니다. 예시로는 session variable, user variable 이 있습니다. 서버에서 하나의 앞단 연결에서 다음과 같은 query 들을 날렸는데, Multiplexing 이 되어서 각 query 들이 실제로는 뒷단 DB로의 연결 A, B, C 에 각각 들어갔다고 한다면, ProxySQL 이 없었을 때랑은 다른 결과가 나올것입니다.

```
SELECT @@session.time_zone;  -- 연결 A. DB 기본 timezone 인 UTC 가 반환됨

SET SESSION time_zone = '+09:00'; -- 연결 B의 timezone 이 변경.

SELECT NOW(); -- ProxySQL 이 없었다면, 해당 연결의 timezone 이 +09:00 이여서 한국 시간으로 나온다.
              -- 하지만 연결 C 로 간다면 DB 기본 timezone 인 UTC 로 나오게 된다.
```

<< 이것도 visualization >>

이러한 문제를 방지하기 위해서, ProxySQL 은 session variable 을 사용만 하더라도(SELECT 만 하더라도) 해당 앞단 연결에 대해서는 Multiplexing 을 꺼버립니다. 그렇게 되면 해당 앞단 연결에 대해서는 하나의 뒷단 연결을 독점으로 배정해주게 됩니다. 이 외에도 Multiplexing 이 disable 되는 조건들이 있기에 개발환경에서 ProxySQL 의 앞단 연결들이 Multiplexing 이 disable 되었는지, 되었다면 왜 되었는지 알아내고 그것들을 해결을 해줘야 됩니다. 이는 ProxySQL 의 설정과 stats 테이블을 통해 알아낼 수 있습니다.

```
update global_variables set variable_value=1 where variable_name = 'mysql-show_processlist_extended';
-- << query 몇개 적어주기 >>
stats_mysql_query_digest 에서 query digest 를 볼수 있다.
```

포트원의 경우, Multiplexing 이 disable 되는 경우가 크게 2개가 있었습니다.
첫번째는 mysql driver 문제였습니다. mysql driver 에서 연결을 맺을 때, 해당 연결의 session variable 들을 읽어오면서 Multiplexing 이 꺼졌습니다. (예시: [mysql-connector-j](https://github.com/mysql/mysql-connector-j/blob/1c3f5c149e0bfe31c7fbeb24e2d260cd890972c4/src/main/core-impl/java/com/mysql/cj/NativeSession.java#L494-L503)) mysql-connector-j 에서는 session variable 들을 Select 를 해서 가져와서 그 값을 저장해놓고, 필요할 때, 해당 값들을 메모리에서 읽어오고 있었습니다. 이 문제의 해결책은 "해당 값들은 Select 만 했을때는 Multiplexing 이 꺼지지 않아도 설정하는 것"입니다. 왜냐하면 만약 누군가 변경을 하는 경우엔 Multiplexing 이 꺼지게 되면서 해당 뒷단 연결은 변경을 한 앞단 연결에 독점배정이 되면서, 해당 앞단 연결이 사라지면 뒷단 연결도 사라지기에, Multiplexing 이 가능한 뒷단의 session variable 들은 모두 DB 의 연결 초기값일 것입니다. 따라서 독점배정이 안된 뒷단 연결들 사이의 해당 session variable 들은 모두 일치할것이기 때문에 Select 에 한해서는 Multiplexing 이 꺼지지 않아도 됩니다.
이런 경우를 처리하기 위해 ProxySQL 에는 mysql_query_rules 라는 기능이 있습니다. mysql_query_rules 를 통해 "특정 regex 에 해당하는 query 에 대해서는 Multiplexing 을 끄지 않는다"는 설정을 할수 있고, [mysql driver 같은 문제의 해결책으로 이를 사용하는걸 권장합니다.](https://github.com/sysown/proxysql/issues/1632#issuecomment-414032364)

두번째는 auto_increment_delay 입니다. 서버 어플리케이션 중에 Insert query 후에 LAST_INSERT_ID() 를 조회해서 해당 값을 사용하는 경우가 있는데, 이 경우 때문에, ProxySQL 은 insert 쿼리를 실행한 후에, query 5개를 후속으로 실행하기 전까지는 Multiplexing 을 잠시 꺼놓는 기능이 기본으로 설정되어 있습니다. ([ProxySQL 문서](https://proxysql.com/documentation/global-variables/mysql-variables/#mysql-auto_increment_delay_multiplex)) 저의 경우엔, 코드 분석 & ProxySQL에 남아있는 query 분석용 데이터 (stats_mysql_query_digest)를 통해서 LAST_INSERT_ID 를 사용하지 않는것을 확인하였고, mysql-auto_increment_delay_multiplex 설정을 0 으로 하여 이 기능을 끔으로서 해결하였습니다.

만약, 이 외적으로 Multiplexing 이 꺼지는 경우가 보이거나 한다면, 잘 분석해서 어떻게 그 문제를 회피할수 있을지 고민하여 회피해야 합니다. 하지만 특별한 경우가 아니라면 여기의 가이드로 해결이 되지 않을까 생각합니다.

### 2. MySQL Version 

MySQL DB 연결 protocol 을 보면 DB 와 클라이언트 간의 handshake 단계에서 DB 는 자신의 버전을 알려주도록 되어있습니다. 그러다보니 ProxySQL 은 Client 와 연결을 맺을 때부터, MySQL 의 버전을 알려줘야 하기 때문에, 자신의 뒷단에 있는 DB 들의 버전을 가지고 와서 넘겨준다거나 하는 동작이 불안정합니다. 따라서 [ProxySQL 설정](https://proxysql.com/documentation/global-variables/mysql-variables/#mysql-server_version)에는 클라이언트 에게 알려준 버전을 설정해주도록 하였습니다. 포트원에서는 5.5 를 8.0 으로 옮기는 업그레이드이고, 모든 프로그램들이 그렇듯 DB 역시 Backward compatibility 를 지키려고 하기 때문에 ProxySQL 에 설정하는 버전은 5.5 로 두는것이 안전합니다.

그렇다는건, 클라이언트 입장에서는 5.5 라고 생각하며 query 를 날리는 데, 이것이 뒷단의 8.0 DB 에 전달이 될수 있다는 것이고, 이 때문에 8.0 에 생긴 breaking changes 에 영향을 받을수 밖에 없습니다. 문제가 된 변경들은 다음과 같습니다. 

- query_cache_size, query_cache_type : 사라짐.
- tx_isolation : transaction_isolation 으로 변경

해당 문제들도 mysql_query_rules 기능을 통해서 해결 가능합니다. match_pattern, replace_pattern 을 통해 query 의 특정 부분을 변경하는 것을 지원합니다. 이 기능을 통해 문제가 된 query 들을 바꾸는 규칙을 Switchover 할때 같이 생성하는 방식으로 해결할 수 있지만, 포트원의 경우 문제가 되는 query 들이 driver 의 연결때만 있는 문제였기 때문에, 좀 더 쉽게, mysql_query_rules 에서 해당 값들을 하드코딩 시켜버리는 식으로 해결하였습니다. (뒤의 query_rules 참고)

### 결론 : ProxySQL 설정

결론적으로 문제들을 봤을 때, mysql_query_rules 와 적당한 설정들로 문제들을 다 해결할 수 있었습니다. DB 를 어떻게 사용하느냐에 따라 모두 다르겠지만, 저희의 경우가 일반적인 경우이지 않을까 생각합니다. 만약 다른 문제들이 있다면 각 문제를 나름의 방식대로 해결을 해야되지 않을까 생각합니다. 글을 이해하는데 도움이 되었으면 하여, 결론적으로 사용된 mysql_query_rules 를 첨부해 놓습니다.

<details>
<summary>mysql_query_rules</summary>
```
INSERT INTO mysql_query_rules (
    rule_id,
    active,
    flagIN,
    match_digest,
    match_pattern,
    flagOUT,
    replace_pattern,
    multiplex,
    apply,
    comment
)
VALUES
(
    1,
    1,
    0,
    '^SELECT @@session.auto_increment_increment AS auto_increment_increment,@@character_set_client AS character_set_client.*',
    '@@query_cache_size AS query_cache_size, @@query_cache_type AS query_cache_type',
    10,
    '0 AS query_cache_size, ''OFF'' AS query_cache_type',
    2,
    0,
    'for multiplexing & mysql 8.0 backend'
),
(
    2,
    1,
    0,
    '^SELECT @@session.autocommit',
    NULL,
    NULL,
    NULL,
    2,
    1,
    'for multiplexing'
),
(
    3,
    1,
    0,
    '^SELECT @@session.tx_isolation',
    '@@session.tx_isolation',
    NULL,
    '''REPEATABLE-READ'' as ''@@session.tx_isolation''',
    2,
    1,
    'for multiplexing & mysql 8.0 backend'
),
(
    4,
    1,
    0,
    '^SELECT @@tx_isolation AS i,@@innodb_lock_wait_timeout AS l,@@version_comment AS v',
    '@@tx_isolation',
    NULL,
    '''REPEATABLE-READ''',
    2,
    1,
    'for multiplexing & mysql 8.0 backend'
),
(
    5,
    1,
    10,
    '^SELECT @@session.auto_increment_increment AS auto_increment_increment,@@character_set_client AS character_set_client.*',
    '@@tx_isolation',
    NULL,
    '''REPEATABLE-READ''',
    2,
    1,
    'for multiplexing & mysql 8.0 backends. this is chain rule from rule 1.'
);
```
</details>

## SwitchOver 스크립트

SwitchOver 의 과정을 자세히 설명하면 다음과 같습니다.

1. 5초 타이머를 시작합니다. 만약 밑 과정이 5초를 넘기게 되면, switchover 를 취소하고 다시 이전 DB 를 사용하도록 합니다.
  1. DB Proxy 에서 이전 DB 사용을 draining 합니다.
  2. 새로운 DB 가 이전 DB 의 변경사항을 모두 따라잡을때까지 기다립니다.
2. DB Proxy 에서 새로운 DB 를 사용하도록 합니다.

### 이전 DB 사용 draining 

여기에서 Draining 을 하는 건, ProxySQL 의 OFFLINE_SOFT 라는 상태를 사용하면 됩니다. ProxySQL 은 여러 DB 들을 backend 로 등록할 수 있고, 등록된 DB 의 status 를 조절해서 해당 DB 를 사용할지 말지를 결정할수 있습니다. ONLINE 상태는 해당 DB 를 뒷단 연결로서 사용할 수 있는 상태이고, OFFLINE 은 OFFLINE_HARD 와 OFFLINE_SOFT, 두가지가 있습니다.
OFFLINE_SOFT 의 경우, 이미 해당 DB 를 사용하고 있는 연결은 게속 해당 연결을 사용하는걸 허용하되, 새로운 연결을 받지 않는 상태이고, OFFLINE_HARD 은 이미 존재하는 연결도 강제로 끊습니다. 무중단 업그레이드에서는 OFFLINE_SOFT 를 사용하면 원하는 draining 을 할수 있습니다.

OFFLINE_SOFT 로 설정한 후, 모든 connection 이 draining 되었는지 확인하려면 이전 DB 의 연결이 존재하는지 확인하면 됩니다. 이는 ProxySQL 에서 확인할수도 있고, 이전 DB 의 processlist 으로 확인할수도 있습니다. 실제 Switchover 스크립트에서는 둘다 polling 을 해서 연결이 없는걸 확인하였습니다.

### 변경사항을 모두 따라잡았는지 확인

이전 DB 에 연결이 모두 사라지면, 더이상 새로운 변경사항이 쌓이지는 않습니다. 이제 남은것은 이전 DB 의 변경사항들이 새로운 DB 에 모두 반영되었는지를 확인해야합니다. 가장 확실한 것은 binlog replication 의 정보를 보면 됩니다. 이전 DB 의 binlog position 와 새로운 db 의 replication status 의 Read_Master_Log_Pos, Exec_Master_Log_Pos, 총 3개의 값이 모두 같은지를 polling 하면서 확인하였습니다. 만약 GTID 가 설정이 되어있다면 GTID 로 확인하는것이 확실하지만 5.5 에는 GTID 가 없기 때문에 선택할수 없었습니다.

# 실제 실행

해당 switchover 스크립트를 실행하는 중에는 새로운 DB query 들이 모두 delay 되기 때문에, 이 시간을 최소화 하고 싶었습니다. 그렇다고 너무 짧게 잡으면 모든 과정이 끝나는게 불가능하게 되기 때문에 적당한 값으로 타협하는것이 중요했습니다. 목표치는 3초, 최대 5초라고 생각했었고 5초를 넘어가면 일반적인 latency 를 훨씬 뛰어넘게되면서 포트원을 사용하는 회사의 respones timeout 에 걸릴까봐 그 이상으로 높이고 싶진 않았습니다. 
Switchover 실행 전에, 3초가 실제로 가능한지 알아보기 위해서 테스트를 하고 싶었습니다. 그래서 Switchover 스크립트에서 새로운 DB 를 사용하는 2번 단계를 없앤 상태로 트래픽이 제일 적은 새벽 시간대에 여러번 실행해보며 실험해봤습니다. 그 결과, 일반적으로 2초 이하로 걸리는걸 볼수 있어 안심하고 Switchover 를 진행할수 있었습니다. 실제로 Switchover 를 했을때에도, 2초 이하로 걸려 문제없이 무중단 DB 업그레이드를 완수할수 있었습니다.

# 추가적인 내용

## MySQL 8.0 테스트

서버 개발자와 QA 팀에서 MySQL 8.0 을 사용했을 때, 문제가 없는지 확인을 해주시긴 했지만 실제 운영환경에서 들어오는 request 유형에 대해서 모든걸 테스트 해본다는건 사실 불가능에 가깝기도 하고, 운영환경에서 들어오는 request 의 조합, 빈도 같은 것들로 그대로 들어왔을때의 DB 의 부하 등등을 체크하기 위해서는 운영환경에서 실행되고 있는 DB query 들에 대해서 테스트를 해보고 싶었습니다. 이를 위해서 ProxySQL 의 Mirroring 기능을 사용하였습니다.

ProxySQL Mirroring 을 사용하면 사용중인 DB 와 클라이언트에 영향없이 들어오는 모든 query 들을 그대로 다른 DB 에 미러링할수 있습니다. DB 의 정합성 문제, transaction 같은 것들을 모두 지켜가며 미러링하는것은 불가능하지만, 미러링의 목적이 부하테스트 이자 query 에 문제가 있는지 확인하는 용도이기 때문에 큰 문제는 없습니다. 그래서 총 2가지의 mirroring 실험을 진행했습니다. replication 진행중인 replica 로 select query 만 mirroring 해보기, 그리고 replication 을 끊고 insert, update, delete 문 전부를 mirroring 해보기. Mirroring 을 한 후, ProxySQL 의 stats_mysql_query_digest 에서 response time 과 sum_rows_affected, stats_mysql_errors 에서 error 가 뜨는지 확인 등을 하여 query 중에 MySQL 8.0 에서 문제가 생기는 query 가 있는지 확인하였고, DB 의 cpu utilization 같은 지표들을 통해 문제가 없는지 파악했습니다.

ProxySQL Mirroring 은 아니지만 개발자가 만들어주신 benchmark 를 통해 알게된 문제가 있었습니다. benchmark 를 돌려보니 5.5 DB 에서는 문제없이 돌았는데, 같은 부하로 8.0 DB 에 돌려보니, 버티지 못했습니다. 이유를 찾아보니 query cache 라는 기능이였습니다. MySQL 의 query cache 는 8.0 에서 완전히 사라진 기능으로, select 문에 대해서 결과값을 caching 해놓고, 똑같은 Select 문이 들어왔을 때 caching 된 값을 바로 반환해주는 기능입니다. (Cache invalidation 은 table 단위로 일어나며 select 문이 읽는 table 중 하나라도 변경이 생기면 invalidate 되었습니다) 이전에 index 를 타지 않는 query 가 있었는데 해당 테이블이 변화가 많이 생기지 않는 테이블이다 보니 query cache 의 수혜를 많이 받아 5.5 에서는 문제가 없었는데 8.0 에서는 문제가 생긴 것이였습니다. 해당 table 에 적당한 index 를 걸어줌으로서 해결하였습니다.

## Rollback Plan

만약 8.0 으로 업그레이드를 했는데, 이전에 발견하지 못한 큰 문제가 있어서 롤백을 해야될수도 있다고 생각했습니다. 그 때를 대비하기 위해서는 8.0 DB 로부터 replication 을 받는 5.5 DB 를 만들어놓아야지만, 정합성문제 없이 DB 롤백이 가능해집니다. 하지만 MySQL 의 binlog replication 은 원래 minor 버전 한 단계 위 버전에서 받아오는것만 정식지원합니다. 즉, 업그레이드 하는 방향으로는 한 버전씩 replication 을 정식지원하지, 역방향, 2 단계 이상씩 뛰어넘는 replication 을 정식 지원하지 않습니다. 그렇기에 정방향이긴 하지만 5.5 -> 8.0 의 replication 도 정식지원을 따르기 위해서는 원래는 중간 단계의 DB 를 하나씩 더 만들어서 5.5 -> 5.6 -> 5.7 -> 8.0 으로 한단계씩 진행해야되는게 정확하긴 합니다.
5.5 -> 8.0 의 경우, replication 설정을 하고, 정합성 체크를 여러번 돌려서 문제가 없다는걸 확인하여 진행할수 있었지만, 8.0 -> 5.5 는 binlog replication 이 안되었습니다. 그래서 AWS DMS 를 사용해서 replication 을 진행시켰습니다. AWS DMS 는 약간의 딜레이가 있기 때문에 Replication lag 를 따라잡는데 5초 이상 걸렸지만, 이 rollback 을 사용할 정도의 큰 문제가 생긴 이상, 5초 이상의 replication lag 는 감수하기로 결정했었습니다. 실제로는 이 롤백 플랜이 사용되진 않았습니다. 
