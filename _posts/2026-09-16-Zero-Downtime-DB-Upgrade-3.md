---
layout: post
title:  "무중단 DB 업그레이드 - 3. PostgreSQL 편"
date:   2026-09-27 21:25:00 +0900
categories: dev
img-overlay: 0.1
comments: true
draft: true
series: zero-downtime-db-upgrade
---

포트원의 PostgreSQL DB 들은 AWS Aurora 를 사용중이였습니다. 다음 LTS 로 업그레이드를 진행하여 몇년간은 다시 업그레이드를 할 필요없도록 할 생각으로 대부분의 PostgreSQL DB 들을 업그레이드 하였습니다.

## 새로운 DB 만들기

Aurora 이기 때문에 새로운 DB를 만드는 것은 AWS 에서 잘 지원해주고 있습니다. Create Clone 을 하면 클릭 몇번으로 현재 데이터를 가지고 있는 새로운 DB 를 만드는 것이 가능합니다. 하지만 Clone 된 DB 와 기존 DB 와의 연결은 되어있지 않은 상태로 만들어지게 됩니다. DB 업그레이드를 하려면 이전 DB 로 부터 replication 이 되고 있는 새로운 버전의 DB 이기에 약간의 절차를 거쳐야 합니다.

### logical replication 조건 확인

다른 버전간의 replication 을 하기 위해서는 logical replication 을 사용해야 됩니다. 하지만 Aurora 의 기본 설정은 logical replication 이 꺼져있기 때문에 이것을 켜야합니다. (이걸 키기 위해서는 db restart 가 필요하기 때문에, 만약 꺼져있다면 이걸 무중단으로 켜기 위해서 또 복잡한 과정을 거쳐야 됩니다.) 포트원은 이미 켜놓은 상태로 RDS 를 만들기 때문에 문제는 없었습니다. `show rds.logical_replication;` 으로 확인할수 있습니다.

또 다른 조건으로는, 모든 테이블에 Primary Key(PK) 가 있어야 됩니다. 만약 없는 테이블이 있다면 PK 를 만들어주거나, `REPLICA IDENTITY FULL` 설정을 해줘야 합니다. 해당 설정은 모든 column 을 PK 취급하는 설정이므로 replication 의 data 가 증가하는 부작용이 있긴 합니다. 

<details markdown="block">
<summary>PK 검사하는 방법</summary>

```sql
SELECT
  'ALTER TABLE "' || schemaname || '"."' || tablename || '" REPLICA IDENTITY FULL;'
FROM pg_tables t
WHERE t.schemaname NOT IN ('pg_catalog', 'information_schema')
  AND NOT EXISTS (
    SELECT 1
    FROM pg_index i
    JOIN pg_constraint c ON i.indexrelid = c.conindid
    WHERE i.indrelid = (quote_ident(t.schemaname) || '.' || quote_ident(t.tablename))::regclass
    AND c.contype = 'p'
  )
  AND EXISTS (
    SELECT 1
    FROM pg_class c
    WHERE c.oid = (quote_ident(t.schemaname) || '.' || quote_ident(t.tablename))::regclass
    AND c.relreplident = 'd'
  )
ORDER BY schemaname, tablename;
```

</details>

### 새로운 DB 만들기

Aurora Cluster 로 부터 logical replication 을 받는 새로운 Aurora Cluster 를 만드는 방법은 다음과 같습니다.

1. 이전 DB 에 publication, replication_slot 을 만들어주면서, replication_slot 의 LSN 을 기록해두기 (미리 만들어둬야 WAL 이 이 시점부터 보존이 된다)

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 이전 DB 에서 실행
   CREATE PUBLICATION psql_upgrade_publication FOR ALL TABLES;
   SELECT pg_create_logical_replication_slot('psql_upgrade_replication_slot', 'pgoutput');
   ```

   </details>

2. 이전 DB 로 부터 Clone DB 를 AWS 통해서 만들기
3. 새 DB 의 초기 LSN 을 확인하기

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새 DB 에서 실행
   SELECT aurora_volume_logical_start_lsn();
   ```

   </details>

4. 새 DB 에 존재하는 publication, replication_slot 삭제하기

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새 DB 에서 실행. Clone 할 때 이전 DB 의 것이 같이 복사되었기 때문에 지운다.
   SELECT pg_drop_replication_slot('psql_upgrade_replication_slot');
   DROP PUBLICATION psql_upgrade_publication;
   ```

   </details>

5. 새 DB 의 버전 업그레이드를 AWS 콘솔에서 진행하기
6. 새 DB 에 이전 DB 로부터 replication 을 받도록 Subscription 생성 (enabled=false 로 만들어야 됨)

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새 DB 에서 실행. password 가 log 에 남지 않도록 log 설정을 잠시 끈다.
   BEGIN;
   SET LOCAL log_statement = 'none';
   SET LOCAL log_min_duration_statement = -1;

   CREATE SUBSCRIPTION psql_upgrade_subscription
   CONNECTION 'host=<hostname> dbname=<dbname> user=<user> password=''<password>'''
   PUBLICATION psql_upgrade_publication
   WITH (
     copy_data = false,
     create_slot = false,
     enabled = false,
     connect = true,
     slot_name = 'psql_upgrade_replication_slot'
   );

   COMMIT;
   ```

   </details>

7. replication 을 받기 시작할 LSN 을 step 3 에서 확인한 초기 LSN 으로 설정한다.

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새 DB 에서 실행
   SELECT * FROM pg_replication_origin;

   -- <roname>: 위 쿼리의 roname 값
   -- <initial LSN>: Step 3 에서 확인한 새 DB 의 초기 LSN
   SELECT pg_replication_origin_advance('<roname>', '<initial LSN>');
   ```

   </details>

8. 논리 복제 활성화.

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새 DB 에서 실행
   ALTER SUBSCRIPTION psql_upgrade_subscription ENABLE;
   ```

   </details>

<< visualization 여기 추가 >>

## PgBouncer

PostgreSQL 용 DB Proxy 를 찾다보니 완벽하게 적합한 Proxy 를 찾을순 없었습니다. ProxySQL 은 최근에 PostgreSQL 지원을 시작했지만 DB 업그레이드 시점에는 너무 최근에 지원을 시작했기에 믿음직스럽지 않았고, PgBouncer 는 오래되었고 사람들이 많이 사용해서 신뢰도는 높았지만 무중단 업그레이드에 필요한 기능들이 매우 제한적으로만 존재했습니다. 안정성이 가장 중요했기에 PgBouncer 를 선택하였으나, 만약 뒤에서 설명할 PgBouncer 의 한계점들을 우회하지 못하였다면 다른 DB Proxy 를 고민했을것 같습니다.

### Transaction pool mode

무중단 업그레이드 설계를 보면, 클라이언트와 DB Proxy 사이의 연결(앞단 연결)이 정상적인 상태에서 DB Proxy 와 DB 사이의 연결 (뒷단 연결)이 바뀌어야 합니다. 그래야지 서버(클라이언트)는 아무일이 없는것처럼 느껴지는 상태에서 뒤의 DB 가 바뀔수 있습니다. 하나의 앞단 연결에서 transaction 단위로 DB Proxy 뒷단의 연결을 마음대로 바꿀수 있는 기능을 PgBouncer 에서는 transaction pool mode 라고 합니다.

PgBouncer 에서 transaction pool mode 를 사용하면 transaction 단위로 뒷단의 DB 연결이 달라질수 있기 때문에 [여러 기능들이 제대로 작동하지 않는다고 경고](https://www.pgbouncer.org/features.html)합니다. ProxySQL 에서는 이것들을 우회하거나, 이런 기능을 사용하는 순간 Multiplexing 이 꺼지면서 동작에는 문제가 없도록 해주지만, PgBouncer 는 항상 Multiplexing 이 켜진 상태로 동작하기 때문에, 이런 기능을 사용하는 query 는 에러 없이 실행되더라도 결과는 서버 개발자의 의도와 다를 수 있습니다. 따라서 PgBouncer 의 transaction pool mode 를 사용할 땐 문제가 되는 query 가 존재하는지 확인하고, 존재하면 PgBouncer 설정이 아니라 서버 어플리케이션을 고쳐서 우회해야합니다.

문제가 되는 기능들 중 서버가 사용하고 있던 건 다행히 SET 하나 뿐이였습니다. 

1. jdbc driver 의 startup query 로 들어오는 `SET application_name = 'PostgreSQL JDBC Driver'`
   - application_name 이 설정이 잘 안되도 문제가 없기 때문에 무시할수 있었습니다. 
2. `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ` 같은 transaction isolation 을 설정하는 SET
   - transaction 내에서만 유효하고 transaction 이 끝나면 기본 isolation level 로 돌아가기 때문에 문제가 없습니다.

다행히 서버에서 DB 를 사용하는 방식이 PgBouncer 의 transaction pool mode 와 문제가 없는 방식으로 사용중이였기 때문에 PgBouncer 를 사용할수 있었습니다.

### 그 외의 PgBouncer 관련 이슈들

1. Certificate 
  PgBouncer 를 EC2 에 띄웠는데, SSL 를 사용하려면 PgBouncer 가 SSL 연결을 받아줘야 한다. 앞단에 NLB 같은걸로 TLS 로 처리를 하고 싶을수 있지만, PostgreSQL 의 SSL 은 TLS 와 별도로 구현되어있기에 사용할 수 없습니다. PgBouncer 에 그래서 Certificate 를 설정을 해줘야 하는데, trusted CA 로부터 만든 Certificate 를 발급받는게 귀찮았기 때문에, 직접 root CA 도 만들고, client 에서는 ssl mode 를 verify-ca 보다 낮은 설정으로 잠시동안 설정해줬습니다.
2. tcp keepidle
  tcp keepidle 관련 설정을 하지 않으면 OS 의 기본값을 사용하는데, Aurora RDS 기본 값보다 너무 길기 때문에 문제가 생길수 있습니다. PostgreSQL 에 설정되어있던 tcp keepidle 설정을 참고해서 PgBouncer 에 설정을 해줘야 됩니다. 
3. `ignore_startup_parameters`
  앱 연결 시 PostgreSQL에게 파라미터를 전송하는데 PgBouncer가 모르는 파라미터를 받으면 연결을 거부합니다. 개발환경에서 pgbouncer 를 사용하면서 pgbouncer 로그를 보면 어떤 startup parameter 때문에 문제가 생기는지 알려주기 때문에 개발 환경에서 테스트를 하면서 ignore_startup_parameters 에 추가해줘야 합니다.
  # PgBouncer 로그에서 아래 메시지 확인
  # "unsupported startup parameter: <파라미터명>"

## 결론 : PgBouncer.ini

결론적으로 사용했던 PgBouncer 설정 파일 하나를 참고용으로 첨부합니다.

<details markdown="block">
<summary>pgbouncer.ini</summary>

```ini
[databases]
* = pool_mode=transaction host=<blue db host>

[pgbouncer]
...

tcp_keepalive = 1
tcp_keepidle = 300

...

max_prepared_statements = 100
ignore_startup_parameters = extra_float_digits,options
server_reset_query_always = 1
```

</details>

## SwitchOver

### 이전 DB 사용 draining 

PgBouncer 에서는 DB 사용을 draining 하는것을 [pause, resume 이란 명령어로 지원](https://www.pgbouncer.org/usage.html#pause-db)해줍니다. `pause [db]` 를 하면 진행중인 transaction 이 모두 끝날때까지 기다리고, 새로운 transaction 은 delay 시킵니다.

### 변경사항을 모두 따라잡았는지 확인

이전 DB 에 연결이 모두 사라지면, 더 이상 새로운 변경사항이 쌓이지는 않습니다. 이제 남은것은 이전 DB 의 변경사항들이 새로운 DB 에 모두 반영되었는지를 확인해야합니다. MySQL 의 경우엔 새로운 DB 의 Exec_Master_Log_Pos 를 보면, 어디까지 실행되었는지를 확인할수 있었습니다. 하지만 PostgreSQL 에서는 새로운 DB 가 WAL 에서 어느 LSN 까지 받았는지만 볼수 있지, 어디까지 실행을 했는지는 볼수 없었습니다.

그래서 확실하게 알기 위해서, DB 데이터를 통해 판단하기로 하였습니다. DB 에 switchover_marker 라는 table 을 만들고, 이전 DB 에 연결이 모두 사라진 후에 이전 DB 의 switchover_marker table 에 row 를 추가하고, 해당 row 가 새로운 DB 에 추가가 되었는지를 확인하여 변경사항을 모두 따라잡았는지 확인하였습니다. 

### Sequence 처리

PostgreSQL 에서 logical replication 을 통해서 Sequence 는 동기화가 되지 않습니다. 따라서 Sequence 는 따로 SwitchOver 에서 처리를 해줘야 합니다. 단순하게 모든 Sequence 값을 가지고 와서 새로운 DB 의 Sequence 값들을 변경시켜주면 됩니다.

### 결론 : SwitchOver 과정

실제 Switchover 스크립트의 과정은 다음과 같습니다.

1. 이전 DB 사용 draining
   - PgBouncer 에서 `PAUSE <db>` ← 진행중인 transaction 이 끝날때까지 기다리고, 새로운 transaction 은 대기
   - 이전 DB 의 `pg_stat_activity` 에 서버 어플리케이션의 연결이 없는지 확인
2. 이전 DB 의 switchover_marker table 에 row 추가
3. 타임아웃 안에서 아래 조건들이 모두 만족될 때까지 polling
   - 이전 DB 의 replication slot 의 lag 가 0 (`pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)`)
   - 2번에서 추가한 marker row 가 새로운 DB 에 존재
4. 새로운 DB 사용
   - 이전 DB 의 Sequence 값들을 새로운 DB 에 `setval` 로 반영
   - PgBouncer 설정 교체 ← `pgbouncer.ini` symlink 를 `green.pgbouncer.ini` 로 바꾸고 `RELOAD`
   - PgBouncer 에서 `RESUME <db>` ← 대기하던 transaction 들이 새로운 DB 로 전달됨
5. 1~3 단계가 타임아웃을 넘기거나 중간에 에러가 나면, PgBouncer 설정을 이전 DB 로 되돌리고 `RESUME <db>` ← 롤백

<< visualization 추가하기 >>

# 총 마무리

MySQL 과 PostgreSQL DB 의 무중단 업그레이드를 실행하기 전에 상세한 RunBook 을 만들어놓아서, 실제 실행을 할 때에는 RunBook 을 그대로 따라가면서 실행을 하였고 문제없이 모두 무중단 업그레이드를 완료하였습니다. 회사마다 DB 를 사용하는 방식이 모두 다르기 때문에 해당 블로그 내용 외적으로도 고민해야될 내용이 있을것입니다. (예: Data Pipeline) 그래도 도움이 되었으면 좋겠습니다. 긴 글 읽어주셔서 감사합니다.

{% include series.html title="무중단 DB 업그레이드" %}
