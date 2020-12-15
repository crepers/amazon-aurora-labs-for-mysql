# DB 클러스터 복제

이 실습에서는 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Clone.html" target="_blank">DB 클러스터 복제</a> 프로세스를 학습합니다. 복제는 데이터 세트를 복제한 시점의 일관된 복사본을 사용하여 별도의 독립적인 DB 클러스터를 생성합니다. 데이터베이스 복제는 원본 데이터베이스 또는 복제 데이터베이스에서 데이터가 변경될때 데이터가 복사되는 쓰기중 복사 프로토콜을 사용합니다. 두 클러스터는 격리되어 있으며 복제본에 대한 데이터베이스 작업이나 그반대의 경우 원본 DB 클러스터에 성능 영향이 없습니다.

이 실습에는 다음 작업이 포함됩니다.

1. 복제 DB 클러스터 생성
2. 데이터 세트가 동일한지 확인
3. 클론의 데이터 변경
4. 데이터가 다른지 확인
5. 실습 리소스 정리

이 실습에는 다음 전제 조건이 필요합니다.


* [시작](/prereqs/environment/)
* [Session Manager 워크스테이션에 연결](/prereqs/connect/)
* [새 DB 클러스터 생성](/provisioned/create/) (클러스터를 수동으로 생성하려는 경우)
* [DB 연결, 데이터 로드 및 오토 스케일](/provisioned/interact/) (DB 연결 및 데이터 로드 섹션만 해당)


## 1. 복제 DB 클러스터 생성

!!! warning "워크로드 상태 확인"
	DB 클러스터를 복제하기 전에 이전 실습에서 부하 생성 작업을 중지하고 `quit;`을 사용하여 MySQL 클라이언트 명령줄을 종료했는지 확인합니다.

Session Manager 워크스테이션 명령줄에 아직 연결되어 있지 않은 경우 [다음](/prereqs/connect/) 지침에 따라 연결하십시오. 연결되면 아래 명령을 실행하여 ==[dbSecurityGroup]== 과 ==[dbSubnetGroup]== 를 응답 결과에서 적절한 값으로 변경합니다. CloudFormation 스택의 출력 또는 공식 워크숍에 참여하는 경우 Event Engine Team Dashboard 에서도 찾을 수 있습니다.

```shell
aws rds restore-db-cluster-to-point-in-time \
--restore-type copy-on-write \
--use-latest-restorable-time \
--source-db-cluster-identifier auroralab-mysql-cluster \
--db-cluster-identifier auroralab-mysql-clone \
--vpc-security-group-ids [dbSecurityGroup] \
--db-subnet-group-name [dbSubnetGroup] \
--backtrack-window 0
```

다음으로 다음 명령을 사용하여 복제 생성 상태를 확인합니다. 복제 프로세스를 완료하는데 몇 분 정도 걸릴 수 있습니다. 아래의 예제 출력을 참조하십시오.

```shell
aws rds describe-db-clusters \
--db-cluster-identifier auroralab-mysql-clone \
| jq -r '.DBClusters[0].Status, .DBClusters[0].Endpoint'
```

출력에서 ==status== 와 ==endpoint== 두 결과를 다 기록해 두십시오. 필요한 경우 명령을 여러번 반복하십시오. 상태가 available 되면 클러스터에 DB 인스턴스를 추가할 수 있으며 DB 인스턴스가 추가되면 클러스터 엔드포인트를 나타내는 엔드포인트 값을 통해 클러스터에 연결할 수 있습니다.

<span class="image">![DB Cluster Status](1-describe-cluster.png?raw=true)</span>

??? tip "DB 클러스터 클론을 통한 비용 최적화"
    소스 클러스터를 크게 변경하려는 경우 (예 : 업그레이드 또는 위험한 DDL 작업) DB 인스턴스를 추가하지 않고도 DB 클러스터 복제본을 생성하는 것이 유용하고 비용 효율적인 안전한 조치가 될 수 있습니다. 그러면 작업의 결과로 소스에서 문제가 발생하는 경우 클론이 빠른 롤백 대상이 됩니다. 이러한 이벤트에서 DB 인스턴스를 추가하고 애플리케이션이 복제본을 가리키도록 하기만 하면 됩니다. 동일한 소스에서 직접 또는 간접적으로 파생된 15 개의 클론으로 제한되므로 스냅샷 도구로 클론을 사용하지 않는 것이 좋습니다.


다음 명령을 사용하여 클러스터 상태가 **available** 되면 클러스터에 DB 인스턴스를 추가합니다.


```shell
aws rds create-db-instance \
--db-instance-class db.r5.large \
--engine aurora-mysql \
--db-cluster-identifier auroralab-mysql-clone \
--db-instance-identifier auroralab-mysql-clone-instance
```

다음 명령을 사용하여 클러스터내 DB 인스턴스 생성을 확인합니다.

```shell
aws rds describe-db-instances \
--db-instance-identifier auroralab-mysql-clone-instance \
| jq -r '.DBInstances[0].DBInstanceStatus'
```

<span class="image">![DB Instance Status](1-describe-instance.png?raw=true)</span>

생성 상태를 모니터링하려면 명령을 반복합니다. **상태** 가 **creating** 에서 **available** 으로 변경되면 작동중인 클론이 생성됩니다. 클러스터에 노드를 만드는데도 몇 분이 걸립니다.



## 2. 데이터 세트가 동일한지 확인

데이터를 변경하기전에 원본 및 복제된 DB 클러스터 모두에서 데이터 세트가 동일한지 확인하십시오. **sbtest1** 테이블 에서 체크섬 작업을 수행하여 확인할 수 있습니다.

다음 명령을 사용하여 복제된 데이터베이스에 연결합니다.(`describe-db-cluster`  명령에서 검색한 엔드포인트 사용)


```shell
mysql -h[cluster endpoint of clone] -u$DBUSER -p"$DBPASS" mylab
```

!!! 주의
	연결하는데 사용하는 데이터베이스 자격증명은 원본 DB 클러스터와 동일합니다. 이는 원본의 정확한 복제본이기 때문입니다.


다음으로 클론에서 다음 명령을 실행합니다.

```sql
checksum table sbtest1;
```

명령의 출력은 아래 예와 유사해야합니다. 특정 복제 클러스터의 값을 기록해 두십시오.

<span class="image">![Checksum on clone](2-checksum-clone.png?raw=true)</span>

이제 클론에서 연결을 끊고 다음 순서로 소스 클러스터에 연결합니다.

```
quit;

mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab
```

클론에서 실행한 것과 동일한 체크섬 명령을 실행합니다.

```sql
checksum table sbtest1;
```

특정 소스 클러스터의 값을 기록해 두십시오. 체크섬 값은 위의 복제된 클러스터와 동일해야합니다.


## 3. 클론의 데이터 변경

원래 클러스터에서 연결을 끊고(아직 연결되어있는 경우) 다음 순서로 복제 클러스터에 연결합니다.

```
quit;

mysql -h[cluster endpoint of clone] -u$DBUSER -p"$DBPASS" mylab
```

데이터 행을 삭제하고 체크섬 명령을 다시 실행하십시오.

```sql
delete from sbtest1 where id = 1;

checksum table sbtest1;
```

명령의 출력은 아래 예와 유사해야합니다. 체크섬 값이 변경되었으며 더 이상 위의 섹션 2에서 계산된 것과 동일하지 않습니다.

<span class="image">![Checksum on clone changed](3-checksum-clone-changed.png?raw=true)</span>


## 4. 데이터가 다른지 확인

클론에 대한 삭제 작업의 결과로 소스 클러스터에서 체크섬 값이 변경되지 않았는지 확인합니다. 클론에서 연결을 끊고(아직 연결되어있는 경우) 다음 순서로 소스 클러스터에 연결합니다.

```
quit;

mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab
```

클론에서 실행한 것과 동일한 체크섬 명령을 실행합니다.

```sql
checksum table sbtest1;
```

특정 소스 클러스터의 값을 기록해 두십시오. 체크섬 값은 섹션 2에서 위의 소스 클러스터에 대해 계산된 것과 동일해야합니다.

다음을 사용하여 DB 클러스터에서 연결을 끊습니다.

```sql
quit;
```

## 5. 실습 리소스 정리

이 실습을 실행하여 추가 AWS 리소스를 생성했습니다. 이 실습을 완료한 후 이러한 서비스 사용에 대해 원치 않는 요금이 발생하지 않도록 아래 명령을 실행하여 이러한 리소스를 제거하는 것이 좋습니다.

```shell
aws rds delete-db-instance --db-instance-identifier auroralab-mysql-clone-instance

aws rds delete-db-cluster --db-cluster-identifier auroralab-mysql-clone --skip-final-snapshot
```
