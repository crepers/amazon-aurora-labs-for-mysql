# DB 클러스터 역추적

이 실습에서는 DB 클러스터를 역추적하는 과정을 안내합니다. <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Backtrack.html" target="_blank">역추적 </a>은 DB 클러스터를 지정한 시간으로 "되감습니다". DR 목적으로 DB 클러스터 백업을 대체하는 것은 아니지만 역추적을 사용하면 실수를 빠르게 실행 취소하거나 이전 데이터 변경 사항을 탐색할 수 있습니다.

이 실습에는 다음 작업이 포함됩니다.

1. 의도하지 않은 데이터 변경
2. 의도하지 않은 변경을 복구하기위한 역추적

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/)
* [Session Manager 워크스테이션에 연결](/prereqs/connect/)
* [새 DB 클러스터 생성](/provisioned/create/) (클러스터를 수동으로 생성하려는 경우)
* [DB 연결, 데이터 로드 및 오토 스케일](/provisioned/interact/) (DB 연결 및 데이터 로드 섹션만 해당)


## 1. 의도하지 않은 데이터 변경
Session Manager 워크스테이션 명령줄에 아직 연결되어 있지 않은 경우 [다음](/prereqs/connect/) 지침에 따라 연결하십시오. 그런 다음 이전 실습을 완료한 후 아직 연결되지 않은 경우 다음을 실행하여 MySQL 클라이언트를 사용하여 DB 클러스터 엔드포인트에 연결합니다.


```shell
mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab
```

다음으로 `sbtest1` 테이블을 삭제합니다.

!!! 주의
    아래의 명령을 한 번에 하나씩 실행하고 각 명령 사이에 몇 초 동안 기다리십시오. 이렇게 하면 역추적을 테스트하기 위한 적절한 시점을 쉽게 결정할 수 있습니다. 실제 상황에서 의도하지 않은 변경이 언제 발생했는지 확인할 수 있는 명확한 마커가 항상 있는 것은 아닙니다. 따라서 적절한 시점을 찾기 위해 몇 번 역추적을 해야 할 수도 있습니다.


```sql
SELECT current_timestamp();

DROP TABLE sbtest1;

SELECT current_timestamp();

quit;
```

위의 명령으로 표시된 시간 마커를 기억하거나 저장하십시오. 나중에 참조로 사용하여 데모 목적으로 역추적 할 적절한 시점을 쉽게 결정할 수 있습니다.


<span class="image">![Drop Table](1-drop-table.png?raw=true)</span>


이제 다음 명령을 실행하여 sysbench 명령을 사용하여 삭제된 테이블을 교체하고 ==[clusterEndpoint]== DB 클러스터의 클러스터 엔드포인트 값으로 변경합니다.


```shell
sysbench oltp_write_only \
--threads=1 \
--mysql-host=[clusterEndpoint] \
--mysql-user=$DBUSER \
--mysql-password="$DBPASS" \
--mysql-port=3306 \
--tables=1 \
--mysql-db=mylab \
--table-size=1000000 prepare
```

??? tip "이 파라미터들은 무엇을 의미하나요?"
    파라미터 | 설명
    --- | ---
    --threads | 동시 스레드 개수입니다.
    --mysql-host | Aurora DB 클러스터의 클러스터 엔드포인트 입니다.
    --mysql-user | 인증할 MySQL 사용자의 사용자 이름입니다.
    --mysql-password | 인증할 MySQL 사용자의 비밀번호입니다.
    --mysql-port | Aurora 데이터베이스 엔진이 수신하는 포트입니다.
    --tables | 만들 테이블 개수입니다.
    --mysql-db | 기본적으로 사용할 스키마(데이터베이스)입니다.
    --table-size | 테이블에서 생성할 행 수입니다.

<span class="image">![Sysbench Prepare](1-sysbench-prepare.png?raw=true)</span>

DB 클러스터에 다시 연결하고 체크섬 테이블 작업을 실행합니다. 체크섬 값은 [DB 클러스터 복제 실습](/provisioned/clone/#2-verify-that-the-data-set-is-identical) 에서 계산된 원본 클러스터 값과 달라야합니다.


```
mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab

checksum table sbtest1;

quit;
```

## 2. 의도하지 않은 변경을 복구하기위한 역추적

두 번째 시간보다 약간 뒤의 시간으로 데이터베이스를 역추적합니다.(테이블을 삭제한 직후) 

!!! 주의
    역 추적 작업은 DB 클러스터 수준에서 발생하며이 실습의 예에서는 개별 테이블에 대한 작업의 영향을 보여 주지만 전체 데이터베이스 상태가 지정된 시점으로 롤백됩니다.

```shell
aws rds backtrack-db-cluster \
--db-cluster-identifier auroralab-mysql-cluster \
--backtrack-to "yyyy-mm-ddThh:mm:ssZ"
```

<span class="image">![Backtrack Command](2-backtrack-command.png?raw=true)</span>

역추적 작업의 진행 상황을 추적하려면 아래 명령을 실행하십시오. 필요한 경우 명령을 여러 번 반복하십시오. 작업은 몇 분 안에 완료됩니다.

```shell
aws rds describe-db-clusters \
--db-cluster-identifier auroralab-mysql-cluster \
| jq -r '.DBClusters[0].Status'
```

<span class="image">![Backtrack Status](2-backtrack-status.png?raw=true)</span>

데이터베이스에 다시 연결하십시오. `sbtest1` 테이블은 데이터베이스에서 없어야합니다.


```
mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab

show tables;

quit;
```

<span class="image">![Show Tables](2-show-tables.png?raw=true)</span>

이제 위의 첫 번째 시간보다 약간 앞의 시간으로 다시 되돌아갑니다.(테이블을 삭제하기 전)


```shell
aws rds backtrack-db-cluster \
--db-cluster-identifier auroralab-mysql-cluster \
--backtrack-to "yyyy-mm-ddThh:mm:ssZ"
```

아래 명령을 사용하여 역추적 작업의 진행 상황을 추적하십시오. 작업은 몇 분 안에 완료됩니다. 필요한 경우 명령을 여러 번 반복하십시오.


```shell
aws rds describe-db-clusters \
--db-cluster-identifier auroralab-mysql-cluster \
| jq -r '.DBClusters[0].Status'
```

데이터베이스에 다시 연결하십시오. 이제 `sbtest1` 테이블을 데이터베이스에서 다시 사용할 수 있지만 원래 데이터 세트가 포함됩니다.


```
mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab

show tables;

checksum table sbtest1;

quit;
```
