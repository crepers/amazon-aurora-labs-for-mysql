# Aurora 글로벌 데이터베이스 장애 조치

리전내에 복제본과 함께 사용할 경우 Aurora DB 클러스터는 리전내에서 자동 장애조치 기능을 제공합니다. Aurora 글로벌 데이터베이스를 사용하면 보조 리전의 DB 클러스터에 대한 수동 장애조치를 수행 할 수 있으므로 전체 리전의 인프라 또는 서비스를 사용할 수 없게되는 시나리오에서도 데이터베이스가 유지될 수 있습니다.

교차 리전에 배포된(변경 불가능한 인프라 또는 <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html" target="_blank">지역간 AMI 복사</a>) 애플리케이션 계층과 결합하면 데이터 일관성을 유지하면서 애플리케이션의 가용성을 더욱 높일 수 있습니다.

이 실습에는 다음 작업이 포함됩니다.

1. 기본 리전에 오류 삽입
2. 보조 DB 클러스터 승격

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/) (**Deploy Global DB** 옵션 선택)
* [Session Manager 워크스테이션에 연결](/prereqs/connect/)
* [Aurora 글로벌 데이터베이스 배포](/global/deploy/)

## 1. 기본 리전에 오류 삽입

Session Manager 워크스테이션 명령줄에 아직 연결되어 있지 않은 경우, **기본 리전**에서 [다음 지침](/prereqs/connect/) 에 따라 연결하십시오. 연결되면 아래 명령을 실행하여 DB 클러스터의 클러스터 엔드 포인트를 ==[clusterEndpont]== 에 변경합니다.


!!! warning "리전 확인"
특히 이 가이드에서 오른쪽 화면의 링크에서 서비스 콘솔을 여는 경우 여전히 **기본 리전** 에서 작업하고 있는지 확인합니다 .


```shell
mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab
```

!!! tip "클러스터 엔드포인트 (또는 곳의 파라미터들)는 어디에서 찾을 수 있습니까?"
    Aurora 글로벌 데이터베이스로 작업하고 있으므로 기본 및 보조 리전에서 추적해야하는 여러 엔드포인트가 있습니다. 클러스터의 엔드포인트를 찾는 방법에 대한 지침은 [전역 데이터베이스 배포](global/deploy/) 실습의 마지막을 확인하십시오.


데이터베이스에 연결되면 아래 코드를 사용하여 테이블을 만듭니다. 다음 SQL 쿼리를 실행합니다.

```sql
DROP TABLE IF EXISTS failovertest1;
CREATE TABLE failovertest1 (
  pk INT NOT NULL AUTO_INCREMENT,
  gen_number INT NOT NULL,
  some_text VARCHAR(100),
  input_dt DATETIME,
  PRIMARY KEY (pk)
);
INSERT INTO failovertest1 (gen_number, some_text, input_dt)
VALUES (100,"region-1-input",now());
COMMIT;
SELECT * FROM failovertest1;
```

쿼리 결과를 기록해두면 나중에 비교할때 사용할 수 있습니다.

여러 메커니즘을 사용하여 리전내 실패를 시뮬레이션 할 수 있습니다. 이를 테스트 하기위한 실습은 [내결함성 테스트](/provisioned/failover/) 실습을 참조하십시오. 그러나 이 경우 장기적이고 더 큰 규모의 장애를 시뮬레이션 하게되며(빈번하지 않음)이를 수행하는 가장 좋은 방법은 Aurora 글로벌 데이터베이스의 기본 DB 클러스터를 모든 수신/송신 데이터 트래픽을 중지하는 것입니다. 실습 환경에는 연결된 서브넷에서 나가는 모든 수신/발신 트래픽을 차단하는 특정 **DENY ALL**규칙이 있는 <a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html" target="_blank">VPC 네트워크 ACL(NACL)</a>이 포함되어 있습니다. 이는 기본적으로 애플리케이션에서 기본 리전 DB 클러스터를 사용할 수 없게 만드는 더 광범위한 장애를 가정합니다.

<a href="https://console.aws.amazon.com/vpc/home#acls:sort=networkAclId" target="_blank">VPC 서비스 콘솔</a>을 열고 **기본 리전**에서 **네트워크 ACL** 메뉴를 선택합니다. **auroralab-denyall-nacl** 이라는 NACL이 표시되어야 하며, 현재 서브넷과 연결되어 있지 않습니다. 이름 옆에있는 체크박스를 선택하여 선택하고 하단 세부 사항 패널에서 **인바운드 규칙** 및 **아웃바운드 규칙** 탭을 모두 검토하십시오. 모든 트래픽에 대해 **DENY** 로 설정되어 있는지 확인해야 합니다.

!!! warning "리전 확인"
    특히 위 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 **기본 리전**에서 여전히 작업하고 있는지 확인합니다.

<span class="image">![NACLs Rules Review](vpc-nacl-rules.png)</span>

선택한 NACL의 세부 정보 패널에서 **서브넷 연결** 탭으로 전환한 다음, **서브넷 연결 편집** 버튼을 클릭 합니다.

Aurora DB 클러스터는 실습 환경용으로 구성된 **프라이빗 서브넷**을 사용하도록 구성되고 실습 환경용으로 생성된 RDS DB 서브넷 그룹에 의해 관리됩니다. 이러한 모든 서브넷의 이름은 `auroralab-prv-sub-` 접두사로 지정되고, 가용 영역에 해당하는 번호가 있습니다. 이름을 보려면 **서브넷 ID** 열을 더 넓게 끌어야 할 수 있습니다.

이 접두사로 시작하는 모든 서브넷을 선택합니다. 검색 상자를 사용하여 이름이 **prv**인 서브넷을 필터링한 다음 선택할 수도 있습니다. 그런 다음 **편집** 버튼을 클릭하여 연결을 확인합니다.

<span class="image">![NACLs Edit Subnet Association](vpc-subassoc-edit.png)</span>

연결되면 NACL 규칙이 즉시 적용되고 프라이빗 서브넷 내의 리소스에 연결할 수 없게됩니다. Session Manager 명령줄에서 데이터베이스에 대한 연결은 결국 클라이언트가 서버에 대한 연결이 끊어졌다는 오류 메시지와 함께 끊어집니다.

## 2. 보조 DB 클러스터 승격

장기간의 리전 인프라 또는 서비스 수준 장애를 시뮬레이션하는 경우 일반적으로 애플리케이션 및 데이터베이스 스택에 대해 DR 사이트에 대한 지역 장애 조치를 수행하도록 선택합니다. 다음으로 **보조 지역**에 있는 **보조 DB 클러스터**를 독립된 리전의 DB 클러스터로 승격합니다.

**보조 DB 클러스터**의 MySQL DB 클러스터 세부 정보 페이지에서  <a href="https://console.aws.amazon.com/rds/home?region=us-east-1#database:id=auroralab-mysql-secondary;is-cluster=true" target="_blank">Amazon RDS service console</a>Amazon RDS 서비스 콘솔</a> 을 엽니다.

!!! warning "리전 확인"
    특히 위 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 **보조 리전**에서 여전히 작업하고 있는지 확인합니다 .


??? tip "기본 DB 클러스터 및 DB 인스턴스의 상태가 여전히 <i>사용 가능</i>으로 보고되는 이유는 무엇입니까?"
    RDS 콘솔에서 여전히 기본 리전 DB 클러스터 및 DB 인스턴스를 **사용 가능**으로 보고한다는 것을 알 수 있습니다. 이는 NACL을 통해 모든 네트워킹 액세스를 차단하여 장애를 시뮬레이션하고 이러한 시뮬레이션은 AWS 계정, 특히 VPC 및 영향을 받는 서브넷으로만 제한됩니다. RDS/Aurora 서비스 및 내부 상태 확인은 서비스 제어 플레인 자체에서 제공되며 *실제 중단*이 없기 때문에 DB 클러스터와 DB 인스턴스를 정상으로 보고합니다.

`auroralab-mysql-secondary` 보조 DB 클러스터를 선택한 상태에서, **작업** 메뉴를 클릭한 다음 **글로벌에서 제거**를 선택합니다.


<span class="image">![RDS Remove Cluster From Global](rds-cluster-action-remglobal.png)</span>

기본 DB 클러스터에서 복제가 중단될 것을 확인하는 메시지가 나타납니다. **제거 및 승격**을 클릭하여 확인합니다.

<span class="image">![RDS Confirm Remove Cluster From Global](rds-cluster-confirm-remglobal.png)</span>

프로모션 프로세스는 1분 미만이 소요됩니다. 완료되면 이전 보조 DB 클러스터가 이제 **리전**으로 역할이 지정되고 DB 인스턴스가 이제 **쓰기** 노드인것을 볼 수 있습니다.

새로 승격된 DB 클러스터를 클릭합니다. **연결 & 보안** 탭의 **쓰기** 엔드포인트는 다음과 같이 **사용 가능**으로 표시돠야 합니다. 엔드포인트 문자열을 복사하여 메모장에 붙여 넣습니다.

<span class="image">![RDS Cluster Endpoints Promoted](rds-cluster-endpoints-promoted.png)</span>

**보조 리전**의 Session Manager 워크스테이션에 대한 추가 명령줄 세션을 엽니다. Session Manager 명령줄 세션을 만드는 방법은 [Session Manager를 이용하여 워크스테이션에 연결](/prereqs/connect/)을 참조 하십시오. 그러나 **보조 리전**에서 사용해야 합니다.

!!! warning "리전 확인"
    특히 위 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 **보조 리전**에서 여전히 작업하고 있는지 확인합니다 .

연결되면 보조 리전의 EC2 워크스테이션에서 데이터베이스 자격증명을 설정해야 합니다. 원래 기본 DB 클러스터를 수동으로 생성한 경우 비슷한 단계를 수행했습니다. 다음 명령을 실행하여 식별자를 내용을 아래 표에 표시된 값으로 바꿉니다.

식별자 | 찾을 수있는 곳
----- | -----
==[secretArn]== | 공식 워크샵에 참여하고 있고 이벤트 엔진을 사용하여 실습 환경이 프로비저닝된 경우, Secret ARN의 값은 이벤트 엔진의 팀 대시 보드에서 찾을 수 있습니다. 그렇지 않으면 실습 환경을 프로비저닝하는데 사용한 CloudFormation 스택의 출력에서 찾을 수 있습니다. 값은`arn : aws : secretsmanager :`로 시작합니다.
==[primary_region]== | 사용중인 **기본 리전**의 식별자는 콘솔의 오른쪽 상단에있는 이름을 클릭합니다. 지역은 다를 수 있지만 이름 옆에 `ap-northeast-2`와 같이 표시됩니다.


```shell
CREDS=`aws secretsmanager get-secret-value --secret-id [secretArn] --region [primary_region] | jq -r '.SecretString'`
export DBUSER="`echo $CREDS | jq -r '.username'`"
export DBPASS="`echo $CREDS | jq -r '.password'`"
echo "export DBPASS=\"$DBPASS\"" >> /home/ubuntu/.bashrc
echo "export DBUSER=$DBUSER" >> /home/ubuntu/.bashrc
```

다음으로 아래 명령을 실행하여 ==[newClusterEndpont]== 를 새로 승격된 DB 클러스터의 클러스터 엔드포인트로 변경합니다.

```shell
mysql -h[newClusterEndpoint] -u$DBUSER -p"$DBPASS" mylab
```

!!! tip "클러스터 엔드포인트(또는 다른 표시자 파라미터)는 어디에서 찾을 수 있습니까?
    Aurora 글로벌 데이터베이스로 작업하고 있으므로 기본 및 보조 리전에서 추적해야하는 여러 엔드포인트가 있습니다. 클러스터의 엔드포인트를 찾는 방법에 대한 지침은 [글로벌 데이터베이스 배포](global/deploy/) 실습의 끝을 확인하십시오.

데이터베이스에 연결되면 아래 코드를 사용하여 테이블을 만듭니다. 다음 SQL 쿼리를 실행합니다.

```sql
SELECT * FROM failovertest1;
```

쿼리 결과를 확인합니다. 이렇게하면 시뮬레이션된 장애직전에 데이터베이스에 입력한 새 테이블과 레코드가 반환됩니다.

이 테이블에 새 레코드를 삽입하십시오. 다음 쿼리를 복사하여 붙여넣고 실행합니다. 결과는 어떻습니까?

```sql
INSERT INTO failovertest1 (gen_number, some_text, input_dt)
VALUES (200,"region-2-input",now());
SELECT * FROM mylab.failovertest1;
```

장애 조치전의 이전 결과와 새 레코드를 모두 확인해야합니다. 이제 애플리케이션은 **보조 리전** 이었던 리전에서 읽기 및 쓰기 쿼리를 모두 제공하고 리전 전체가 중단되는 동안 사용자에게 정상적으로 서비스를 제공할 수 있습니다.
