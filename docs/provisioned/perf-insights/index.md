# 성능 개선 도우미 사용

이 실습에서는 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html" target="_blank">Amazon RDS 성능 개선 도우미</a> 의 사용 방법을 보여줍니다. Amazon RDS 성능 개선 도우미는 데이터베이스 성능을 분석하고 문제를 해결할 수 있도록 Amazon RDS DB 인스턴스로드를 모니터링합니다.

이 실습에는 다음 작업이 포함됩니다.

1. DB 클러스터에 부하 생성
2. 성능 개선 도우미 인터페이스 이해
3. DB 인스턴스의 성능 검사

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/)
* [Session Manager 워크스테이션에 연결](/prereqs/connect/)
* [새 DB 클러스터 생성](/provisioned/create/) (클러스터를 수동으로 생성하려는 경우)


## 1. DB 클러스터에 부하 생성

부하를 생성하기 위해 sysbench를 기반으로하는 Percona의 TPCC와 유사한 벤치 마크 스크립트를 사용합니다. 단순화를 위해 <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-ssm-docs.html" target="_blank">AWS Systems Manager Command</a> 에 올바른 명령 세트를 패키징했습니다. <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html" target="_blank">AWS System Manager Run Command</a> 를 실행합니다.

Session Manager 워크 스테이션 명령 줄에 아직 연결되어 있지 않은 경우 [다음](/prereqs/connect/) 지침에 따라 연결하십시오. 연결되면 아래에서 상황에 가장 적합한 탭을 선택하고 표시된 명령을 실행하십시오.


=== "DB 클러스터가 미리 생성되었습니다."
    AWS CloudFormation이 사용자를 대신하여 DB 클러스터를 프로비저닝하고 **새 DB 클러스터 생성** 실습을 건너 뛴 경우 아래의 간단한 명령을 실행하여 ==[ec2Instance]== 에 값을 입력하세요. CloudFormation 스택 출력에도 ​값이 있으며, 공식 워크숍에 참여하는 경우 Event Engine 팀 대시 보드에서도 확인이 가능합니다.

        aws ssm send-command \
        --document-name auroralab-sysbench-test \
        --instance-ids [ec2Instance]




=== "직접 DB 클러스터를 생성했습니다."
    [새 DB 클러스터 생성](/provisioned/create/) 실습을 완료하고 Aurora DB 클러스터를 수동으로 생성한 경우 아래 명령을 수동으로 실행하여 ==[ec2Instance]== 에 값을 입력하세요. CloudFormation 스택 출력에도 값이 있으며, 공식 워크숍에 참여하는 경우 Event Engine 팀 대시 보드에서도 확인이 가능하니다. 또한 ==[clusterEndpoint]== 를 DB 클러스터의 클러스터 엔드 포인트로 입력합니다.


        aws ssm send-command \
        --document-name auroralab-sysbench-test \
        --instance-ids [ec2Instance] \
        --parameters \
        clusterEndpoint=[clusterEndpoint],\
        dbUser=$DBUSER,\
        dbPassword="$DBPASS"

??? tip "이 파라미터들은 무엇을 의미합니까??"
    Parameter | Description
    --- | ---
    --document-name | 실행할 명령 문서의 이름입니다.
    --instance-ids | 명령을 실행할 EC2 인스턴스입니다.
    --parameters | 추가적인 파라미터들 입니다.

    
이 명령은 테스트 데이터 세트를 준비하고 부하 테스트를 실행할 워크 스테이션 EC2 인스턴스로 전송됩니다. CloudWatch가 지표에 추가로드를 반영하는데 몇 분이 걸릴 수 있습니다. 명령이 시작되었다는 메시지를 확인할 수 있습니다.


<span class="image">![SSM Command](ssm-command-sysbench.png?raw=true)</span>

## 2. 성능 개선 도우미 인터페이스 이해

명령이 실행되는 동안 아직 열려있지 않은경우 새 탭의 DB 클러스터 세부 정보에서 <a href="https://console.aws.amazon.com/rds/home" target="_blank">Amazon RDS 서비스 콘솔</a>을 엽니다.


!!! warning "리전 확인"
    위의 링크를 따라 서비스 콘솔을 여는 경우 올바른 지역에서 여전히 작업하고 있는지 확인하십시오.

왼쪽 메뉴에서 **성능 개선 도우미** 메뉴를 클릭 합니다.

<span class="image">![RDS Dashboard](2-menu-perf-ins.png?raw=true)</span>

다음으로 성능 지표를 로드할 원하는 **DB 인스턴스** 를 선택합니다. Aurora DB 클러스터의 경우 성능 지표는 개별 DB 인스턴스 기반으로 노출됩니다. 클러스터를 구성하는 서로 다른 Db 인스턴스는 서로 다른 워크로드 패턴을 실행할 수 있으며 모두 성능 개선 도우미를 사용하지 않을 수 있습니다. 이 실습에서는 **쓰기** (마스터) DB 인스턴스에 대해서만 로드를 생성 합니다. 이름이 `-node-01` 또는 `-instance-1` 로 끝나는 DB 인스턴스를 선택합니다.

<span class="image">![Select DB Instance](2-select-instance.png?raw=true)</span>

DB 인스턴스를 선택하면 RDS 성능 개선 도우미의 기본 대시보드 보기가 표시됩니다. 대시 보드는 3개의 섹션으로 나뉘어져있어 높은 수준의 성능 지표 메트릭에서 개별 쿼리, 대기, 사용자 및 로드를 생성하는 호스트까지 확인할 수 있습니다.

<span class="image">![Performance Insights Dashboard](2-pi-dashboard.png?raw=true)</span>

대시 보드에 표시되는 성능 메트릭은 시간 단위입니다. 당신은 인터페이스의 상단 오른쪽에 걸쳐 버튼을 클릭하여 시간을 조정할 수 있습니다.( 5m, 1h, 5h, 24h, 1w, all) 그래프를 드래그하여 특정 기간을 확대할 수도 있습니다.

!!! 주의
    모든 대시 보드는 시간이 동기화됩니다. 확대하면 하단의 상세 내용을 포함하여 모든 보기가 조정됩니다.



분류 | 필터 | 설명
--- | --- | ---
**Counter Metrics** | 추가 카운터를 선택하려면 오른쪽 상단 모서리에있는 톱니 바퀴 아이콘을 클릭하세요. | T이 섹션은 읽거나 쓰기 행의 수, 버퍼풀 적중률 등과 같은 시간 경과에 따른 내부 데이터베이스 카운터 메트릭을 표시합니다. 이러한 카운터는 비정상 동작의 원인을 식별하기 위해 데이터베이스 로드 메트릭을 포함한 다른 메트릭과 상호 연관시키는 데 유용합니다.
**Database load** | 로드는 대기(기본값), SQL 명령, 사용자 및 호스트별로 분할될 수 있습니다. | 이 지표는 총로드(선택한 차원으로 분할)를 해당 DB 인스턴스에서 사용 가능한 컴퓨팅 용량(vCPU 수)과 연관 시키도록 설계되었습니다. 로드는 **Average Active Session** (AAS) 메트릭을 사용하여 집계되고 정규화됩니다. DB 인스턴스의 컴퓨팅 용량을 초과하는 많은 AAS는 성능 문제의 주요 지표입니다.
Granular Session Activity | **대기**, **SQL** (기본값), **사용자** 및 **호스트** 별로 정렬 | 개별 명령에 대한 자세한 성능 데이터를 얻을 수 있습니다.


## 3. DB 인스턴스의 성능 검사

위의 부하 생성기 워크 로드를 실행하면 성능 개선 도우미 대시 보드에서 아래 예와 유사한 성능 프로필이 표시됩니다. 부하 생성기 명령은 먼저 `sysbench prepare`를 사용하여 초기 데이터 세트를 만듭니다. 그런 다음 4개의 병렬 스레드를 사용한 동시 트랜잭션 읽기 및 쓰기로 구성된 OLTP 워크로드를 5분 동안 실행합니다.

<span class="image">![Load Test Profile](3-load-profile.png?raw=true)</span>


Amazon Aurora MySQL 특정 대기 이벤트는 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.html#AuroraMySQL.Reference.Waitevents" target="_blank">Amazon Aurora MySQL 참조 안내서</a>에  있습니다. Performance Insights 대시 보드 및 참조 가이드 문서를 사용하여 부하 테스트의 워크로드 프로필을 평가하고 다음 질문에 답하십시오.

1. 로드 테스트 중에 데이터베이스 서버가 과부하 상태입니까?
2. 부하 테스트 중에 리소스 병목 현상을 식별할 수 있습니까? 그렇다면 어떻게 완화할 수 있습니까?
3. 부하 테스트 중 가장 일반적인 대기 이벤트는 무엇입니까?
4. 부하 테스트의 첫 번째 단계와 두 번째 단계에서 부하 패턴이 다른 이유는 무엇입니까?