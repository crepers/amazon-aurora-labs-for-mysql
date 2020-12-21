# 데이터베이스 활동 스트림(Database Activity Streams - DAS) 설정

이 실습에서는 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/DBActivityStreams.html" target="_blank">Aurora DAS (Database Activity Streams)</a>를 설정하고 활용하는 방법을 보여줍니다. 데이터베이스 활동 스트림은 관계형 데이터베이스에서 데이터베이스 활동의 거의 실시간 데이터 스트림을 제공합니다. 데이터베이스 활동 스트림을 타사 모니터링 도구와 통합하면 데이터베이스 활동을 모니터링하고 감사 할 수 있습니다.

??? tip "데이터베이스 활동 스트림 (DAS)에 대해 자세히 알아보기"
    데이터베이스 활동 스트림을 사용하면 데이터베이스 활동을 모니터링하고 감사하여 데이터베이스를 보호하고 규정 준수 및 규제 요구 사항을 충족할 수 있습니다. 데이터베이스 활동 스트림 위에 구축된 솔루션은 내부 및 외부 위협으로부터 데이터베이스를 보호할 수 있습니다. 데이터베이스 활동의 수집, 전송, 저장 및 처리는 데이터베이스 외부에서 관리되므로 데이터베이스 사용자 및 관리자와는 독립적인 액세스 제어를 제공합니다. 데이터베이스 활동은 Aurora DB 클러스터를 대신하여 암호화된 <a href="https://docs.aws.amazon.com/streams/latest/dev/introduction.html" target="_blank">Amazon Kinesis 데이터 스트림</a>에 비동기로 게시됩니다 .

    데이터베이스 활동 스트림에는 다음과 같은 제한 및 요구 사항이 있습니다.

    1. 현재 DAS는 MySQL 버전 5.7과 호환되는 Aurora MySQL 버전 2.08.0 이상에서만 지원됩니다.
    2. DAS는 활동 스트림이 항상 고객 관리형 키(CMK)로 암호화되기 때문에 AWS Key Management Service(AWS KMS)를 사용해야합니다.


이 실습에는 다음 작업이 포함됩니다.

1. AWS KMS 고객 관리형 키(CMK) 생성
2. 데이터베이스 활동 스트림 구성
3. 데이터베이스로드 생성
4. 스트림에서 활동 읽기
5. 데이터베이스 활동 스트림 비활성화

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/)
* [Session Manager 워크스테이션에 연결](/prereqs/connect/)
* [새 DB 클러스터 생성](/provisioned/create/) (클러스터를 수동으로 생성하려는 경우)
* [DB 연결, 데이터 로드 및 오토 스케일](/provisioned/interact/) (연결 및 데이터로드 섹션만 해당)


## 1. AWS KMS 고객 관리형 키(CMK) 생성

DAS는 데이터 키를 암호화하기 위해 마스터 키가 필요하며, 이는 기록된 데이터베이스 활동을 암호화 합니다 (자세한 내용 은 (see <a href="https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#enveloping" target="_blank">봉투 암호화</a> 참조). 기본 Amazon RDS 마스터 키는 DAS의 마스터 키로 사용할 수 없습니다. 따라서 DAS를 구성하려면 새 AWS KMS 고객 관리형 키(CMK)를 생성해야 합니다.

<a href="https://console.aws.amazon.com/kms/home#/kms/home" target="_blank">AWS Key Management Service (KMS) 콘솔</a>을 엽니다. **키 생성**을 클릭합니다 .

<span class="image">![KMS Home](kms-home.png?raw=true)</span>

On the next screen under **Configure key** choose `Symmetric` for **Key type** and click **Next**.
아래 다음 화면에서 **키 구성**에서 **대칭**을 선택하고 **다음**을 클릭합니다.


<span class="image">![KMS Configure](kms-configure.png?raw=true)</span>

In the **Create alias and description** section:
**별청 및 설명 생성** 섹션 에서:


* [ ] **별칭** 을 `auroralab-mysql-das` 로 설정합니다.
* [ ] 다음 설명에 `Amazon Aurora lab, CMK for Aurora MySQL Database Activity Streaming (DAS)`로 입력합니다.

그런 **다음**을 클릭합니다.

<span class="image">![KMS Labels](kms-labels.png?raw=true)</span>

다음 단계는이 실습을 실행하는 상황에 따라 다릅니다. 아래에서 상황에 가장 적합한 탭을 선택하십시오.

=== "Event Engine을 사용하는 워크샵에 있습니다."
    **키 관리자** 섹션에서(이름으로 검색할 수 있습니다)

    * [ ] `TeamRole` 과 `OpsRole` 을 선택하고 확인란을 체크합니다. 
    
    **다음**을 클릭합니다.

    In the **This account** section (you can search for the names to find them quicker):

    **이 계정** 섹션에서(이름으로 검색할 수 있습니다)

    * [ ] `TeamRole` 과 `OpsRole` 을 선택하고 확인란을 체크합니다. 
    * [ ] `auroralab-wkstation-[region]`(둘 이상일 수 있음)옆에있는 확인란 체합니다.

=== "내 계정을 사용하고 있습니다"
    **키 관리자** 섹션에서

    * [ ] 로그인한 IAM 역할 또는 사용자 또는 키를 관리할 다른 관리 계정을 선택합니다.

    **다음**을 클릭합니다.

    *이 계정** 섹션에서

    * [ ] 로그인한 IAM 역할 또는 사용자 또는 키를 관리할 다른 관리 계정을 선택합니다.
    * [ ] `auroralab-wkstation-[region]`(둘 이상일 수 있음)옆에있는 확인란 체합니다.

계속 하려면 **다음** 을 클릭하십시오


<span class="image">![KMS Administrators](kms-admins.png?raw=true)</span>

정책을 검토하고 **완료** 버튼을 클릭합니다 .

<span class="image">![KMS Review](kms-review.png?raw=true)</span>

KMS 대시 보드에서 새로 생성된 KMS 키를 확인합니다.

<span class="image">![KMS Listing](kms-listing.png?raw=true)</span>


## 2. 데이터베이스 활동 스트림 구성
클러스터 세부 정보 페이지에서  <a href="https://console.aws.amazon.com/rds/home#database:id=auroralab-mysql-cluster;is-cluster=true" target="_blank">Amazon RDS 서비스 콘솔</a>을 엽니다. 다른 방법으로 RDS 콘솔로 이동한 경우 콘솔의 **데이터베이스** 섹션에서 `auroralab-mysql-cluster`을 클릭합니다.

**작업** 버튼 메뉴에서 **활동 스트림 시작**을 선택합니다. **데이터베이스 활동 스트림 설정** 창이 나타납니다.

<span class="image">![RDS Cluster Details](rds-cluster-detail-action.png?raw=true)</span>

**마스터 키** 를 이전 단계에서 만든 대칭 키의 별칭으로 설정합니다. **즉시 적용**을 선택하고 **계속**을 클릭합니다.

<span class="image">![RDS Enable DAS](rds-das-confirm.png?raw=true)</span>

DB 클러스터의 **상태** 열에 **configuring-activity-stream**이 표시되기 시작합니다. 클러스터가 다시 **사용 가능** 해질 때까지 기다리십시오. 최신 상태를 얻으려면 브라우저 페이지를 새로 고쳐야 할 수 있습니다.

<span class="image">![RDS Cluster Configuring](rds-cluster-configuring.png?raw=true)</span>

`auroralab-mysql-cluster` 클러스터를 클릭하고 **구성** 탭으로 전환하여 DAS가 활성화되었는지 확인합니다.

<span class="image">![RDS Cluster Configuration Details](rds-cluster-config-pane.png?raw=true)</span>

**리소스 ID** 및 **Kinesis 스트림** 값을 적어둡니다. 이후 실습에서 이 값들이 필요합니다.


## 3. 데이터베이스로드 생성

읽기 전용 워크로드를 사용하여 DB 클러스터에 부하를 생성합니다. 이 [부하 생성 스크립트](/scripts/reader_loadtest.py)는 동시 스레드를 사용하여 다양한 읽기 쿼리를 생성합니다.

Session Manager 워크 스테이션 명령줄에 아직 연결되어 있지 않은 경우 [다음](/prereqs/connect/) 지침에 따라 연결하십시오. 연결되면 아래 명령을 실행하여 ==[clusterEndpoint]== 를 DB 클러스터의 클러스터 엔드포인트로 입력합니다. CloudFormation 스택 출력 또는 워크숍에 참여하는 경우 Event Engine 팀 대시 보드애서도 엔드포인트를 찾을 수 있습니다.

```shell
python3 reader_loadtest.py -e[clusterEndpoint] -u$DBUSER -p"$DBPASS" -dmylab -t2
```

<span class="image">![SSM Command](ssm-command-loadtest.png?raw=true)</span>

`Ctrl+C`를 눌러 언제든지로드 생성기 스크립트를 종료할 수 있습니다.


## 4. 스트림에서 활동 읽기

[활동 스트림 소비자 스크립트](/scripts/das_reader.py) 를 사용하여 활동 스트림에서 이벤트를 읽고 명령줄에 표시합니다.

다른 세션에서 실행중인 부하 생성기에 의해 생성된 활동 이벤트를 보려면 세션 관리자 워크 스테이션에 대한 추가 명령줄 세션을 열어야합니다. 새 세션 관리자 명령줄 세션을 만드는 방법은 [Session Manager에 연결](/prereqs/connect/)을 참조하세요.(이전 실습에서 아직 활성 세션이 없는 경우) 활동 스트림이 활성화 된 후 위에서 확인한 **리소스 ID** 및 **스트림 이름** 값을 ==[resourceId]== 와 ==[streamName]== 에 입력하고 새 세션에서 아래 명령을 실행합니다.

```shell
python3 das_reader.py -i [resourceId] -s [streamName]
```

`Ctrl+C`를 눌러 언제든지 모니터링 스크립트를 종료할 수 있습니다.


이벤트를 더 잘 보려면 <a href="https://jsonformatter.org/" target="_blank">jsonformatter.org</a> 와 같은 사이트를 이용하여 JSON 구조를 더 읽기 쉽게 확인 할 수 있습니다.

<span class="image">![Formatted Output](das-formatted-output.png?raw=true)</span>

결과는 다음 예제와 유사해야 합니다.

```json
{
  "logTime": "2020-08-05 20:15:14.055973+00",
  "type": "record",
  "clientApplication": null,
  "pid": 21971,
  "dbUserName": "masteruser",
  "databaseName": "mylab",
  "remoteHost": "172.31.0.211",
  "remotePort": "10935",
  "command": "QUERY",
  "commandText": "SELECT SQL_NO_CACHE *, SHA2(c, 512), SQRT(k) FROM sbtest1 WHERE id BETWEEN 1953750 AND 1954012 ORDER BY id DESC LIMIT 10",
  "paramList": null,
  "objectType": "TABLE",
  "objectName": "sbtest1",
  "statementId": 5070228,
  "substatementId": 1,
  "exitCode": "0",
  "sessionId": "851",
  "rowCount": 10,
  "serverHost": "auroralab-mysql-node-1",
  "serverType": "MySQL",
  "serviceName": "Amazon Aurora MySQL",
  "serverVersion": "MySQL 5.7.12",
  "startTime": "2020-08-05 20:15:14.055697+00",
  "endTime": "2020-08-05 20:15:14.055973+00",
  "transactionId": "0",
  "dbProtocol": "MySQL",
  "netProtocol": "TCP",
  "errorMessage": "",
  "class": "MAIN"
}
```


## 5. 데이터베이스 활동 스트림 비활성화

Amazon RDS 서비스 콘솔을 아직 열려있지 않은경우 <a href="https://console.aws.amazon.com/rds/home?#database:id=auroralab-mysql-cluster;is-cluster=true" target="_blank">Amazon RDS 서비스 콘솔</a>을 엽니다. 클러스터가 아직 선택되지 않은경우 데이터베이스를 선택하고 `auroralab-mysql-cluster`클러스터가 있는 DB를 클릭합니다.

**작업**의 드롭 다운 메뉴에서 **활동 스트림 중지**를 선택합니다.

<span class="image">![RDS Cluster Stop](rds-cluster-action-stop.png?raw=true)</span>

설정 화면에서 **즉시 적용**을 선택하고 **계속**을 클릭합니다.

<span class="image">![RDS DAS Stop](rds-das-stop.png?raw=true)</span>

클러스터에 대한 RDS 데이터베이스 홈 페이지의 상태 열이 `configuring-activity-stream` 로 표시되기 시작합니다. DB 클러스터 및 DB 인스턴스가 상태가 `사용 가능` 으로 표시되면 작업이 완료된 것입니다.
