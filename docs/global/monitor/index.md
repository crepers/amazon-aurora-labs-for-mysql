# 글로벌 데이터베이스 모니터링

Amazon Aurora는 Aurora 글로벌 데이터베이스의 상태와 성능을 모니터링하고 결정하는데 사용할 수 있는 다양한  <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Monitoring.html" target="_blank">Amazon CloudWatch 지표</a> 를 제공합니다. 이 실습에서는 Aurora 글로벌 데이터베이스에 대한 지연 시간, 복제된 IO 및 교차 리전 복제 데이터 전송을 모니터링하기 위해 Amazon CloudWatch 대시 보드를 생성합니다.

이 실습에는 다음 작업이 포함됩니다.

1. 기본 DB 클러스터에 부하 생성
3. 클러스터로드 및 복제 지연 모니터링

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/) (**Deploy Global DB** 옵션 선택)
* [Session Manager 워크스테이션에 연결](/prereqs/connect/)
* [Aurora 글로벌 데이터베이스 배포](/global/deploy/)


## 1. 기본 DB 클러스터에 부하 생성

부하를 생성하기 위해 sysbench를 기반으로하는 Percona의 TPCC와 유사한 벤치 마크 스크립트를 사용합니다. 단순화를 위해 AWS Systems Manager 부하를 생성하기 위해 sysbench를 기반으로하는 Percona의 TPCC와 유사한 벤치 마크 스크립트를 사용합니다. 단순화를 위해 <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-ssm-docs.html" target="_blank">AWS Systems Manager Command 문서</a> 에 올바른 명령 세트를 패키징했습니다.<a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html" target="_blank">AWS Systems Manager Run Command</a> 를 사용하여 테스트합니다.


Session Manager 워크스테이션 명령줄에 아직 연결되어 있지 않은 경우 **기본 리전** 에서 [다음](/prereqs/connect/) 지침에 따라 연결하십시오. 연결되면 다음 명령중 하나를 입력하여 필요한 내용을 적절하게 바꿉니다.

!!! warning "리전 확인"
    특히 이 가이드에서 오른쪽 화면에서 서비스 콘솔을 열 수있는 링크인 경우 여전히 **기본 리전** 에서 작업하고 있는지 확인합니다.

[새 DB 클러스터 생성 실습](/provisioned/create/)을 완료하고 Aurora DB 클러스터를 생성한 경우 다음 명령을 수동으로 실행합니다.


```shell
aws ssm send-command \
--document-name [loadTestRunDoc] \
--instance-ids [ec2Instance] \
--parameters \
clusterEndpoint=[clusterEndpoint],\
dbUser=$DBUSER,\
dbPassword="$DBPASS"
```

AWS CloudFormation을 사용하여 DB 클러스터를 프로비저닝하고 **새 DB 클러스터 생성 실습**을 건너뛴 경우 다음과 같은 간단한 명령을 실행할 수 있습니다.


```shell
aws ssm send-command \
--document-name [loadTestRunDoc] \
--instance-ids [ec2Instance]
```

??? tip "이 파라미터들은 무엇을 의미합니까?"
    파라미터 | 설명
    --- | ---
    --document-name | 사용자를 대신하여 실행할 Command 문서의 이름입니다.
    --instance-ids | 이 명령을 실행할 EC2 인스턴스입니다.
    --parameters | 추가 명령 파라미터.

이 명령은 테스트 데이터 세트를 준비하고 부하 테스트를 실행할 워크스테이션 EC2 인스턴스로 전송됩니다. CloudWatch가 지표에 추가로드를 반영하는데 최대 1 분이 걸릴 수 있습니다. 명령이 시작되었다는 확인 메시지가 표시됩니다.

<span class="image">![SSM Command](ssm-cmd-sysbench.png?raw=true)</span>


## 2. 클러스터로드 및 복제 지연 모니터링

**기본 리전**의 RDS 서비스 콘솔에서 아직 선택하지 않은 경우, `auroralab-mysql-cluster`(기본)를 선택하고 **모니터링** 탭으로 전환합니다. 해당 클러스터에있는 쓰기 및 읽기 DB 인스턴스의 합쳐진 내용이 표시됩니다. 현재 읽기를 사용하고 있지 않으며 부하는 쓰기 인스턴스에만 전달됩니다. 메트릭을 탐색하고 구체적으로 **CPU 사용률**, **커밋 처리량**, **DML 처리량**,**처리량 선택** 메트릭을 검토하고 초기 데이터 세트를 채우는 sysbench 도구로 인한 초기 급증을 넘어서 상당히 안정적임을 확인하십시오.

<span class="image">![RDS Cluster Primary Metrics](rds-cluster-primary-metrics.png?raw=true)</span>

다음으로 새로 생성된 **보조 DB 클러스터**로 이동합니다. CloudWatch 대시 보드를 생성하여 글로벌 클러스터보다 구체적으로 보조 DB 클러스터와 관련된 세 가지 주요 지표를 모니터링합니다.


CloudWatch 지표 이름 | 설명
----- | -----
`AuroraGlobalDBReplicatedWriteIO` | 보조 지역에 복제된 쓰기 IO 수.
`AuroraGlobalDBDataTransferBytes` | 보조 영역으로 전송된 리두 로그의 양(바이트)
`AuroraGlobalDBReplicationLag` | 밀리 초 단위로 측정한 보조 영역이 기본 영역의 작성자보다 뒤처지는 정도

보조 지역에서  <a href="https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:" target="_blank">Amazon CloudWatch 서비스 콘솔</a> 을 열고 **대시보드**로 이동합니다.

!!! warning "리전 확인"
    이후 단계에서는 다른 리전(버지니아 북부, us-east-1)에서 작업하게 됩니다. 여러 브라우저 탭과 명령줄 세션이 열려 있으므로 항상 의도한 리전에서 작동하는지 확인하십시오.

<span class="image">![CloudWatch Dashboards Listing](cw-dash-listing.png?raw=true)</span>

**대시보드 생성**을 클릭합니다. 새 대시 보드의 이름을 `auroralab-gdb-dashboard`로 지정하고 **대시보드 생성** 버튼을 다시 클릭합니다.

<span class="image">![CloudWatch Dashboard Creation](cw-dash-create.png)</span>

보조 및 기본 Aurora 클러스터간의 복제 지연 시간을 보여주는 첫 번째 위젯을 대시보드에 추가합니다. **번호(즉각적으로 지표 최신 값 확인)**를 다음을 클릭합니다.

<span class="image">![CloudWatch Dashboard Add Number Widget](cw-dash-add-number.png)</span>

**지표 그래프 추가**화면에서, **모든 지표** 탭에서 **RDS**를 선택한 다음, **DBClusterIdentifier, SourceRegion**이라는 메트릭 그룹을 선택합니다.

이제 필터링된 지표 이름 `AuroraGlobalDBReplicationLag` 이 표시되고 SourceRegion 열이 기본 클러스터의 기본 지역 이름으로 표시됩니다. 체크 박스를 사용하여이 측정 항목을 선택합니다.

이제 위젯 미리보기가 밀리 초 단위의 지연 시간 샘플과 함께 맨 위에 있어야합니다. 위젯을 추가로 업데이트하겠습니다. 편집 아이콘 (연필 아이콘)을 클릭하여 친숙한 이름을 지정하고 위젯 이름을`제목 없음`에서`Global DB Replication Lag (평균, 1분)`로 변경한 다음 체크/확인 아이콘을 눌러 변경 사항을 적용합니다.

하단에서 **그래프로 표시된 지표** 탭을 클릭하여 수정합니다. **통계** 열에서 이를 '평균'으로 변경하고, **기간**을 '1분'으로 변경합니다.

설정이 아래 예와 유사한지 확인한 다음 **위젯 생성**을 클릭합니다.

<span class="image">![CloudWatch Widget Configuration](cw-widget-setup.png)</span>

이제 첫 번째 위젯을 만들었습니다. 오른쪽 상단 새로 고침 메뉴에서 설정된 간격으로 자동 새로 고침으로 설정할 수 있습니다.

**대시보드 저장**을 클릭하여 변경 사항을 저장하십시오.

<span class="image">![CloudWatch Dashboard Single Widget](cw-dash-single-widget.png)</span>

보다 완벽한 모니터링 대시보드를 구축하기 위해 대시 보드에 위젯을 개별적으로 추가할 수 있습니다. 그러나 시간을 절약하기 위해 아래 JSON 사양으로 대시 보드의 소스를 업데이트하기 만하면됩니다.

먼저, 대시보드에서 **작업** 드롭 다운을 클릭하고 **소스 보기/편집**을 선택합니다.

<span class="image">![CloudWatch Dashboard Actions](cw-dash-actions.png)</span>

화면에 나타나는 텍스트 상자에 다음 JSON 코드를 붙여 넣습니다. 아래 코드에서 필요한 경우 **보조 리전**과 일치하도록 AWS 지역을 업데이트해야 합니다. 또한 이 가이드에 표시된 것과 다른 DB 클러스터 식별자(이름)를 DB 클러스터에 사용한 경우 해당 식별자도 업데이트해야 합니다.


```json
{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 3,
            "width": 24,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/RDS", "AuroraGlobalDBReplicationLag", "DBClusterIdentifier", "auroralab-mysql-secondary" ],
                    [ "...", { "stat": "Maximum" } ]
                ],
                "view": "timeSeries",
                "stacked": false,
                "region": "us-east-1",
                "title": "Global DB Replication Lag (max vs. avg, 1min)",
                "stat": "Average",
                "period": 60
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 9,
            "height": 3,
            "properties": {
                "metrics": [
                    [ "AWS/RDS", "AuroraGlobalDBReplicationLag", "DBClusterIdentifier", "auroralab-mysql-secondary" ]
                ],
                "view": "singleValue",
                "region": "us-east-1",
                "title": "Global DB Replication Lag (avg, 1min)",
                "stat": "Average",
                "period": 60
            }
        },
        {
            "type": "metric",
            "x": 9,
            "y": 0,
            "width": 15,
            "height": 3,
            "properties": {
                "metrics": [
                    [ "AWS/RDS", "AuroraGlobalDBReplicatedWriteIO", "DBClusterIdentifier", "auroralab-mysql-secondary", { "label": "Global DB Replicated Write IOs" } ],
                    [ ".", "AuroraGlobalDBDataTransferBytes", ".", ".", { "label": "Global DB DataTransfer Bytes" } ]
                ],
                "view": "singleValue",
                "region": "us-east-1",
                "stat": "Sum",
                "period": 86400,
                "title": "Billable Replication Metrics (aggregate, last 24 hr)"
            }
        }
    ]
}
```

대시보드를 변경하려면 **업데이트**를 클릭하십시오.

<span class="image">![CloudWatch Dashboard Source](cw-dash-source.png)</span>

**대시보드 저장**을 클릭하여 대시보드가 새 변경 사항으로 저장되었는지 확인하십시오.

<span class="image">![CloudWatch Dashboard Multiple Widgets](cw-dash-multi-widget.png)</span>

!!! 주의
    **Aurora Global DB Replicated I/O** 및 **Aurora Global DB Data Transfer Bytes** 지표는 1 시간에 한 번만 보고되므로 시간에 따라 해당 지표에 대한 데이터를 볼 수 없을 수도 있습니다. 다른 실습을 수행하는 경우 해당 실습을 계속하고 나중에 다시 방문하여 이러한 측정 항목을 확인하십시오.
