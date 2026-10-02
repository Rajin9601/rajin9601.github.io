---
layout: post
title:  "무중단 DB 업그레이드 - 3. PostgreSQL"
date:   2026-09-27 21:25:00 +0900
categories: dev
img-overlay: 0.1
comments: true
draft: true
series: zero-downtime-db-upgrade
---

# PostgreSQL DB 업그레이드

포트원의 PostgreSQL DB 들은 AWS Aurora 를 사용 중이었습니다. 다음 LTS 로 업그레이드하여 몇 년간은 다시 업그레이드를 할 필요가 없도록 할 생각으로, 대부분의 PostgreSQL DB 들을 업그레이드하였습니다.

## 새로운 DB 만들기

Aurora 이기 때문에 새로운 DB 를 만드는 것은 AWS 에서 잘 지원해주고 있습니다. Create Clone 을 하면 클릭 몇 번으로 현재 데이터를 가지고 있는 새로운 DB 를 만들 수 있습니다. 하지만 clone 된 DB 는 기존 DB 와 연결되어 있지 않은 상태로 만들어집니다. DB 업그레이드에 필요한 것은 이전 DB 로부터 replication 을 받고 있는 새로운 버전의 DB 이기에, 약간의 절차를 거쳐야 합니다.

### logical replication 조건 확인

다른 버전 간의 replication 을 하기 위해서는 logical replication 을 사용해야 합니다. 하지만 Aurora 의 기본 설정은 logical replication 이 꺼져 있기 때문에 이것을 켜야 합니다. (이것을 켜기 위해서는 DB 재시작이 필요하기 때문에, 만약 꺼져 있다면 이것을 무중단으로 켜기 위해 또 복잡한 과정을 거쳐야 합니다.) 포트원은 이미 켜놓은 상태로 DB 를 만들기 때문에 문제는 없었습니다. `show rds.logical_replication;` 으로 확인할 수 있습니다.

또 다른 조건으로는, UPDATE/DELETE 가 일어나는 모든 table 에 Primary Key(PK)가 있어야 합니다. 만약 PK 가 없는 table 이 있다면 PK 를 만들어주거나, `REPLICA IDENTITY FULL` 설정을 해줘야 합니다. 해당 설정은 모든 column 을 PK 처럼 취급하는 설정이므로 replication 데이터가 증가하고, replica DB 의 부하가 증가할 수 있는 부작용이 있긴 합니다.[^1]

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

### replication 을 받는 새로운 DB 만들기

Aurora cluster 로부터 logical replication 을 받는 새로운 Aurora cluster 를 만드는 방법은 다음과 같습니다.

1. 이전 DB 에 publication 과 replication slot 을 만들고, replication slot 의 LSN 을 기록해두기 (미리 만들어둬야 이 시점부터 WAL 이 보존됩니다.)

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 이전 DB 에서 실행
   CREATE PUBLICATION psql_upgrade_publication FOR ALL TABLES;
   SELECT pg_create_logical_replication_slot('psql_upgrade_replication_slot', 'pgoutput');
   ```

   </details>

2. AWS 를 통해 이전 DB 로부터 clone DB 만들기
3. 새로운 DB 의 초기 LSN 확인하기

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새로운 DB 에서 실행
   SELECT aurora_volume_logical_start_lsn();
   ```

   </details>

4. 새로운 DB 에 존재하는 publication 과 replication slot 삭제하기

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새로운 DB 에서 실행. Clone 할 때 이전 DB 의 것이 같이 복사되었기 때문에 지운다.
   SELECT pg_drop_replication_slot('psql_upgrade_replication_slot');
   DROP PUBLICATION psql_upgrade_publication;
   ```

   </details>

5. AWS 콘솔에서 새로운 DB 의 버전 업그레이드하기
6. 새로운 DB 에 이전 DB 로부터 replication 을 받는 subscription 생성하기 (`enabled = false` 로 만들어야 합니다.)

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새로운 DB 에서 실행. password 가 log 에 남지 않도록 log 설정을 잠시 끈다.
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

7. replication 을 받기 시작할 LSN 을 3번 단계에서 확인한 초기 LSN 으로 설정하기

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새로운 DB 에서 실행
   SELECT * FROM pg_replication_origin;

   -- <roname>: 위 쿼리의 roname 값
   -- <initial LSN>: 3번 단계에서 확인한 새로운 DB 의 초기 LSN
   SELECT pg_replication_origin_advance('<roname>', '<initial LSN>');
   ```

   </details>

8. subscription 활성화하기

   <details markdown="block">
   <summary>Query</summary>

   ```sql
   -- 새로운 DB 에서 실행
   ALTER SUBSCRIPTION psql_upgrade_subscription ENABLE;
   ```

   </details>

{% include viz/pg-logical-replication.html %}

## PgBouncer

PostgreSQL 용 DB Proxy 를 찾다 보니 완벽하게 적합한 Proxy 를 찾을 수는 없었습니다. ProxySQL 은 PostgreSQL 지원을 시작한 지 DB 업그레이드 시점 기준으로 너무 얼마 되지 않아 믿음직스럽지 않았고, PgBouncer 는 오래되었고 많은 사람들이 사용해서 신뢰도는 높았지만 무중단 업그레이드에 필요한 기능들이 매우 제한적으로만 존재했습니다. 안정성이 가장 중요했기에 PgBouncer 를 선택하였으나, 만약 뒤에서 설명할 PgBouncer 의 한계점들을 우회하지 못하였다면 다른 DB Proxy 를 고민했을 것 같습니다.

### Transaction pool mode

무중단 업그레이드 설계를 보면, 클라이언트와 DB Proxy 사이의 연결(앞단 연결)이 정상적인 상태에서 DB Proxy 와 DB 사이의 연결(뒷단 연결)이 바뀌어야 합니다. 그래야 서버(클라이언트) 입장에서는 아무 일도 없는 것처럼 느끼는 상태에서 뒷단의 DB 가 바뀔 수 있습니다. 하나의 앞단 연결에서 transaction 단위로 DB Proxy 뒷단의 연결을 자유롭게 바꿀 수 있는 기능을 PgBouncer 에서는 transaction pool mode 라고 합니다.

PgBouncer 는 transaction pool mode 를 사용하면 transaction 단위로 뒷단 연결이 달라질 수 있기 때문에 [여러 기능들이 제대로 작동하지 않는다고 경고](https://www.pgbouncer.org/features.html)합니다. ProxySQL 은 이런 기능들을 우회하거나, 이런 기능을 사용하는 순간 Multiplexing 을 꺼서 동작에 문제가 없도록 해줍니다. 하지만 PgBouncer 는 항상 Multiplexing 이 켜진 상태로 동작하기 때문에, 이런 기능을 사용하는 query 는 에러 없이 실행되더라도 결과가 서버 개발자의 의도와 다를 수 있습니다. 따라서 PgBouncer 의 transaction pool mode 를 사용할 때는 문제가 되는 query 가 존재하는지 확인하고, 존재한다면 PgBouncer 설정이 아니라 서버 애플리케이션을 고쳐서 우회해야 합니다.

문제가 되는 기능 중 서버가 사용하고 있던 것은 다행히 SET 하나뿐이었습니다.

1. JDBC driver 의 startup query 로 들어오는 `SET application_name = 'PostgreSQL JDBC Driver'`
   - application_name 이 제대로 설정되지 않아도 문제가 없기 때문에 무시할 수 있었습니다.
2. `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ` 같은, transaction isolation 을 설정하는 SET
   - transaction 내에서만 유효하고 transaction 이 끝나면 기본 isolation level 로 돌아가기 때문에 문제가 없습니다.

그리고 혹시 몰라서 `server_reset_query_always = 1` 도 설정하였습니다. 이 설정을 켜면 transaction pool mode 에서도 transaction 이 끝날 때마다 뒷단 연결에 `server_reset_query`(기본값 `DISCARD ALL`)를 실행합니다. 그래서 어떤 클라이언트가 우리가 모르는 session 단위의 SET 을 하더라도, 그 값이 같은 뒷단 연결을 이어서 쓰는 다른 클라이언트에게 퍼지지 않습니다. ([PgBouncer 문서](https://www.pgbouncer.org/config.html#server_reset_query_always))

다행히 서버에서 DB 를 사용하는 방식이 PgBouncer 의 transaction pool mode 와 충돌하지 않았기 때문에 PgBouncer 를 사용할 수 있었습니다.

### 그 외의 PgBouncer 관련 이슈들

1. **인증서**

   PgBouncer 를 EC2 에 띄웠는데, SSL 을 사용하려면 PgBouncer 가 SSL 연결을 받아줘야 합니다. 앞단에 NLB 같은 것을 두어 TLS 를 처리하고 싶을 수 있지만, PostgreSQL 의 SSL 연결은 일반적인 TLS 와 시작 방식이 다르기 때문에 그렇게 사용할 수 없습니다. PostgreSQL 은 연결 초기에 평문으로 `SSLRequest` 메시지를 주고받은 후에 TLS handshake 를 시작하기 때문에, 처음부터 TLS handshake 를 기대하는 NLB 같은 장비는 이를 처리하지 못합니다. 그래서 PgBouncer 에 인증서를 설정해야 하는데, trusted CA 로부터 인증서를 발급받는 것이 번거로웠기 때문에 root CA 를 직접 만들고, 클라이언트에서는 `sslmode` 를 `verify-ca` 보다 낮은 설정으로 잠시 설정해줬습니다.

2. **`tcp_keepidle`**

   `tcp_keepidle` 관련 설정을 하지 않으면 OS 의 기본값을 사용하는데, 이 값이 Aurora 의 기본값보다 너무 길기 때문에 문제가 생길 수 있습니다. PostgreSQL 에 설정되어 있던 `tcp_keepidle` 설정을 참고해서 PgBouncer 에 설정해줘야 합니다.

3. **`ignore_startup_parameters`**

   클라이언트는 연결 시 PostgreSQL 에 startup parameter 를 전송하는데, PgBouncer 가 모르는 parameter 를 받으면 연결을 거부합니다. 개발 환경에서 PgBouncer 를 사용하면서 PgBouncer 로그를 보면 어떤 startup parameter 때문에 문제가 생기는지 알려주기 때문에, 개발 환경에서 테스트하면서 `ignore_startup_parameters` 에 추가해줘야 합니다.

   ```
   # PgBouncer 로그에서 아래 메시지 확인
   # "unsupported startup parameter: <파라미터명>"
   ```

### 결론: pgbouncer.ini

최종적으로 사용했던 PgBouncer 설정 파일 하나를 참고용으로 첨부합니다.

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

## Switchover

### 이전 DB 사용 draining

PgBouncer 에서는 DB 사용을 draining 하는 것을 [`PAUSE`, `RESUME` 이라는 명령어로 지원](https://www.pgbouncer.org/usage.html#pause-db)해줍니다. `PAUSE <db>` 를 하면 진행 중인 transaction 이 모두 끝날 때까지 기다리고, 새로운 transaction 은 delay 시킵니다.

### 변경사항을 모두 따라잡았는지 확인

이전 DB 에 연결이 모두 사라지면, 더 이상 새로운 변경사항이 쌓이지 않습니다. 이제 남은 것은 이전 DB 의 변경사항들이 새로운 DB 에 모두 반영되었는지 확인하는 것입니다. MySQL 의 경우엔 새로운 DB 의 `Exec_Master_Log_Pos` 를 보면 어디까지 실행되었는지를 확인할 수 있었습니다. PostgreSQL 에서도 비슷하게 확인할 수 있습니다. 새로운 DB 는 이전 DB 의 변경사항을 어디까지 반영했는지를 이전 DB 에 계속 알려주고, 이 값은 이전 DB 에서 replication slot 의 `confirmed_flush_lsn` 으로 볼 수 있습니다. 따라서 이전 DB 의 현재 WAL 위치(`pg_current_wal_lsn()`)와 `confirmed_flush_lsn` 의 차이가 0 이 되면, 새로운 DB 가 변경사항을 모두 따라잡았다고 판단할 수 있습니다. (새로운 DB 에서는 `pg_replication_origin_status` 의 `remote_lsn` 으로 어디까지 반영했는지 볼 수 있습니다.)

여기에 더해 확실하게 하기 위해, DB 데이터로도 한 번 더 확인하였습니다. DB 에 `switchover_marker` 라는 table 을 만들고, 이전 DB 에 연결이 모두 사라진 후에 이전 DB 의 `switchover_marker` table 에 row 를 추가합니다. 그리고 해당 row 가 새로운 DB 에 추가되었는지를 확인하여 변경사항을 모두 따라잡았는지 확인하였습니다.

### Sequence 처리

PostgreSQL 에서 logical replication 으로는 sequence 가 동기화되지 않습니다. 따라서 sequence 는 Switchover 과정에서 따로 처리해줘야 합니다. 단순하게 모든 sequence 값을 가져와서 새로운 DB 의 sequence 값들을 변경해주면 됩니다.

### 결론: Switchover 과정

실제 Switchover 스크립트의 과정은 다음과 같습니다.

1. 이전 DB 사용 draining
   - PgBouncer 에서 `PAUSE <db>` ← 진행 중인 transaction 이 끝날 때까지 기다리고, 새로운 transaction 은 대기
   - 이전 DB 의 `pg_stat_activity` 에 서버 애플리케이션의 연결이 없는지 확인
2. 이전 DB 의 `switchover_marker` table 에 row 추가
3. 타임아웃 안에서 아래 조건들이 모두 만족될 때까지 polling
   - 이전 DB 의 replication slot 의 lag 가 0 (`pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)`)
   - 2번에서 추가한 marker row 가 새로운 DB 에 존재
4. 새로운 DB 사용
   - 이전 DB 의 sequence 값들을 새로운 DB 에 `setval` 로 반영
   - PgBouncer 설정 교체 ← `pgbouncer.ini` symlink 를 `green.pgbouncer.ini` 로 바꾸고 `RELOAD`
   - PgBouncer 에서 `RESUME <db>` ← 대기하던 transaction 들이 새로운 DB 로 전달됨
5. 1~3 단계가 타임아웃을 넘기거나 중간에 에러가 나면, PgBouncer 설정을 이전 DB 로 되돌리고 `RESUME <db>` ← 롤백

{% include viz/pg-switchover.html %}

# 마무리

MySQL 과 PostgreSQL DB 의 무중단 업그레이드를 실행하기 전에 상세한 runbook 을 만들어 놓았고, 실제 실행할 때에는 runbook 을 그대로 따라가며 실행하여 문제없이 모두 무중단 업그레이드를 완료하였습니다. 회사마다 DB 를 사용하는 방식이 모두 다르기 때문에, 이 글의 내용 외에도 고민해야 할 내용이 있을 것입니다. (예: data pipeline) 그래도 도움이 되었으면 좋겠습니다. 긴 글 읽어주셔서 감사합니다.

[^1]: `REPLICA IDENTITY FULL` 인 table 에 UPDATE/DELETE 가 일어나면, replica DB 는 바뀔 row 를 찾아야 합니다. PostgreSQL 15 이하에서는 이때 index 를 사용하지 못하고 table 전체를 sequential scan 하기 때문에, table 이 크다면 replica DB 의 부하와 replication lag 가 크게 늘어날 수 있습니다. PostgreSQL 16 부터는 replica DB 에 있는 B-tree index 를 사용하여 row 를 찾을 수 있습니다.

{% include series.html title="무중단 DB 업그레이드" %}
