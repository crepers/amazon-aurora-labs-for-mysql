# Aurora 글로벌 데이터베이스 배포

<a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html" target="_blank">Amazon Aurora 글로벌 데이터베이스</a> 는 전 세계적으로 분 된 애플리케이션용으로 설계되어 단일 Amazon Aurora 데이터베이스가 여러 AWS 리전에 걸쳐있을 수 있습니다. 데이터베이스 성능에 영향을 주지않고 데이터를 복제하고, 각 지역에서 짧은 지연 시간으로 빠른 로컬 읽기를 가능하게하며, 지역 전체 중단으로부터 재해 복구를 제공합니다.

이 실습에는 다음 작업이 포함됩니다.

1. 다른 지역에 실습 환경 만들기
2. Aurora 글로벌 클러스터 생성

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/)(**Deploy Global DB** 옵션 선택)
* [새 DB 클러스터 생성](/provisioned/create/) (클러스터를 수동으로 생성하려는 경우에만)


## 1. 다른 리전에 실습 환경 만들기

!!! warning "여러 리전 사용"
    글로벌 데이터베이스의 **다중 리전** 특성으로 인해 이 실습의 작업을 수행하기 위해 두 지역간에 전환하는 경우가 많습니다. 생성된 리소스 중 일부는 두 리전간에 매우 유사하므로 적절한 리전에서 작업을 수행하고 있다는 점에 **유의하십시오.**

    현재 DB 클러스터가 배포되고 지금까지 작업한 리전을 **기본 리전**이라고 합니다.

    보조, 읽기 전용 DB 클러스터를 배포할 리전을 **보조 리전**이라고 합니다.

실습 시작을 단순화하기 위해 실습 환경에 필요한 리소스를 프로비저닝하는  <a href="https://aws.amazon.com/cloudformation/" target="_blank">AWS CloudFormation</a>용 기본 템플릿을 만들었습니다. 이러한 템플릿은 일관된 네트워킹 인프라와 실습에서 사용되는 소프트웨어 패키지 및 구성 요소의 클라이언트측 환경을 배포하도록 설계되었습니다.

Aurora 글로벌 데이터베이스를 지원하기 위해 **미국 동부(버지니아 북부, us-east-1)** 리전에서 실습 환경을 프로비저닝 하려면 아래 **스택 생성**을 클릭하십시오 .

<a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=auroralab&templateURL=https://s3.amazonaws.com/[[bucket]]/templates/lab_template.yml&param_deployCluster=No&param_deployML=No&param_deployGDB=Yes&param_isSecondary=Yes" target="_blank"><img src="/assets/images/cloudformation-launch-stack.png" alt="Launch Stack"></a>

**스택 이름** 필드에서 값이 `auroralab`으로 설정되어 있는지 확인하십시오. 나머지 매개 변수에 대해 모든 기본값을 사용하십시오.

페이지 하단으로 스크롤하여 다음 확인란을 선택합니다. `AWS CloudFormation에서 사용자 지정 이름으로 IAM 리소스를 생성할 수 있음을 승인합니다`를 확인한 다음 **스택 생성** 을 클릭 합니다.

<span class="image">![Create Stack](cfn-create-stack-confirm.png?raw=true)</span>

스택을 프로비저닝하는데 약 10분이 소요되며 **스택 세부 정보** 페이지에서 상태를 모니터링 할 수 있습니다. **이벤트 탭** 을 새로고쳐 스택 생성 프로세스의 진행 상황을 모니터링 할 수 있습니다. 목록의 최신 이벤트 `CREATE_COMPLETE`는 스택 리소스를 나타냅니다.


<span class="image">![Stack Status](cfn-stack-status.png?raw=true)</span>

스택 상태가 `CREATE_COMPLETE`이면 **출력** 탭을 클릭합니다. 여기에있는 값은 나머지 실습을 완료하는데 중요합니다. 나머지 실습 동안 쉽게 액세스할 수 있도록 이 값들을 저장하십시오. **키** 열에 나타나는 이름은 매개 변수 형식을 사용하여 후속 단계의 지침에서 직접 참조됩니다. ==[outputKey]==


<span class="image">![Stack Outputs](cfn-stack-outputs.png?raw=true)</span>


## 2. Aurora 글로벌 클러스터 생성

자동으로 프로비저닝된 실습 환경에는 부하 생성기를 실행중인 Aurora MySQL DB 클러스터가 이미 있습니다. 이 기존 DB 클러스터를 **기본** 으로 사용하여 글로벌 데이터베이스 클러스터를 생성합니다.


???+ info "'글로벌 클러스터' vs. 'DB 클러스터': 차이점은 무엇입니까?"
    **DB 클러스터**는 한 리전에서만 존재합니다. 동일한 스토리지 볼륨을 공유하는 최대 16개의 **DB 인스턴스**를 위한 컨테이너입니다. 이것은 Aurora 클러스터의 기존 구성입니다. 프로비저닝된 클러스터, 서버리스 또는 다중 마스터 클러스터 배포 여부에 관계없이 단일 리전내에 **DB 클러스터**를 배포합니다 .

    **글로벌 [데이터베이스] 클러스터**가 여러 가지에 대한 컨테이너이고 **DB 클러스터** 의 다른 영역에있는 각각의 응집력 데이터베이스와 해당 법. **글로벌 클러스터**는 구성되어 한 개의 특정 리전에서 **기본 [DB] 클러스터**를 최대 5개까지 쓰기를 사용할 수 있고, 그리고 **보조 [DB] 클러스터**의 다른 지역에서 각 읽기 전용으로 사용할 수 있습니다. 특정 **글로벌 클러스터** 의 각 DB 클러스터에는 자체 스토리지 볼륨이 있지만 데이터는 특수 제작된 낮은 지연 및 높은 처리량의 복제 시스템을 사용하여 **기본 클러스터** 에서 각 **보조 클러스터**로 비동기식으로 복제됩니다.


위의 **1단계에서 다른 리전에 실습 환경만들기** 가 완료되면 계속 진행할 수 있습니다.


**기본 리전** 의 MySQL DB 클러스터 세부 정보 페이지에서 <a href="https://console.aws.amazon.com/rds/home#database:id=auroralab-mysql-cluster;is-cluster=true" target="_blank">Amazon RDS 서비스 콘솔</a> 을 엽니다. 위 링크가 아닌 다른 방법으로 RDS 콘솔로 이동한 경우 RDS 서비스 콘솔의 **데이터베이스** 메뉴에서 `auroralab-mysql-cluster`를 클릭하고 **기본 리전**으로 돌아왔는지 확인합니다.

!!! warning "리전 확인"
    특히 위 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 **기본 지역** 에서 여전히 작업하고 있는지 확인합니다 .


첫째, **역추적** 기능을 **해제**해야 합니다. 현재 데이터베이스 역추적은 Aurora 글로벌 데이터베이스와 호환되지 않으며 해당 기능이 활성화된 클러스터는 글로벌 데이터베이스로 변환할 수 없습니다. `auroralab-mysql-cluster`를 선택하고 **수정** 버튼을 클릭합니다.



!!! 주의
    [DB Cluster 역추적](/provisioned/backtrack/) 실습을 아직 완료 하지 않았거나 실습이 필요한 경우는 이 실습을 진행하기전에 지금 해당 실습을 완료하십시오. 비활성화되면 기존 DB 클러스터에서 역추적 기능을 다시 활성화 할 수 없습니다.



<span class="image">![RDS Cluster Modify](rds-cluster-action-modify.png?raw=true)</span>

추가 구성 섹션(필요한 경우 확장)까지 아래로 스크롤하고 **역추적 활성화** 옵션을 선택 취소한 다음, 페이지 하단에서 **계속** 을 클릭합니다.


<span class="image">![RDS Cluster Disable Backtrack](rds-cluster-disable-backtrack.png?raw=true)</span>

**수정 예약** 섹션에서 **즉시 적용** 옵션을 선택한 다음, **클러스터 수정**을 클릭합니다.


<span class="image">![RDS Cluster Confirm Changes](rds-cluster-modify-confirm.png?raw=true)</span>


수정이 완료되고 DB 클러스터가 다시 `사용 가능` 상태가 되면 **작업** 드롭다운 버튼에서 **리전 추가**를 선택합니다.


!!! 주의
    리전 추가를 하기 전에 역추적을 비활성화한 후, 웹 브라우저 페이지를 새로 고쳐야 할 수 있습니다. **리전 추가**가 회색으로 표시되면 웹 브라우저 페이지를 새로 고침하거나 역추적이 올바르게 비활성화되었는지 확인하십시오.


<span class="image">![RDS Cluster Add Region](rds-cluster-action-add.png?raw=true)</span>

보조 DB 클러스터의 구성 화면에서 다음 옵션을 설정합니다.

1. **글로벌 데이터베이스 설정** 섹션에서 :

    * [ ] **글로벌 데이터베이스 식별자** 를 `auroralab-mysql-global`으로 설정

2. **AWS 리전** 섹션에서 :

    * [ ] **보조 리전**을 `US East (N. Virginia)`로 선택합니다.

3. **연결** 섹션에서 **추가 연결 구성**이라는 하위 섹션을 확장합니다. 이 섹션에서는 위에서 만든 정의된 네트워크 구성 내에서 데이터베이스 클러스터를 배포할 위치를 지정할 수 있습니다.

    * [ ] **Virtual Private Cloud (VPC)** 을 `auroralab-vpc`로 설정합니다.
    * [ ] **서브넷 그룹**은 자동으로 `auroralab-db-subnet-group`이 선택되었는지 확인합니다.
    * [ ] **퍼블릭 액세스 가능** 옵션이 `No`로 설정되어 있는지 확인하십시오.
    * [ ] **VPC 보안 그룹** 은 `auroralab-database-sg`를 선택하고, `default`와 같은 다른 보안 그룹은 제거합니다.

4. **추가 구성** 섹션을 확장하고, 다음 옵션을 구성합니다.

    * [ ] **DB 인스턴스 식별자**에 `auroralab-mysql-node-3`을 설정합니다.
    * [ ] **DB 클러스터 식별자**로 `auroralab-mysql-secondary`를 설정합니다.
    * [ ] **DB 클러스터 파라미터 그룹**에서 스택 이름으로 시작하는 그룹을 선택합니다.  (예 : auroralab-[...])
    * [ ] **DB 파라미터 그룹**에서 스택 이름으로 시작하는 그룹을 선택합니다. (예 : auroralab-[...])
    * [ ] **백업 보존 기간** 에 1일을 선택합니다.
    * [ ] **성능 개선 도우미 활성화**을 체크합니다.
    * [ ] **보존 기간** 을 `기본값 (7 일)`로 설정합니다.
    * [ ] **마스터 키**는 `[default] aws/rds`를 설정합니다.
    * [ ] **Enhanced 모니터링을 활성화를**를 체크합니다.
    * [ ] **세부 수준**은 `1 초`로 설정합니다.
    * [ ] **역할 모니터링** 에 `auroralab-monitor-[secondary-region]`를 설정합니다.

!!! 주의
    목록에는 **두 개**의 역할 모니터링이 있습니다. 하나는 기본 리전(웹 페이지의 오른쪽 상단 모서리에 있음)에 대한 역할이고 다른 하나는 보조 리전에 대한 역할 (일반적으로 `us-east-1`)입니다. 이 단계에서는 **보조 리전**이 필요합니다.



<span class="image">![RDS Cluster Add Region](rds-cluster-add-region.png?raw=true)</span>

??? tip "이러한 선택은 무엇을 의미합니까?""
    한 단계로 연결된 구성을 사용하여 해당 보조 클러스터에 글로벌 클러스터, 보조 DB 클러스터 및 DB 인스턴스를 생성합니다. 기존 DB 클러스터는 새 글로벌 클러스터에서 기본 DB 클러스터가 됩니다. AWS CLI, SDK 또는 기타 도구를 사용하여 글로벌 클러스터를 생성하는 경우 RDS 서비스에 대한 별개의 API 호출입니다. RDS 서비스 콘솔은 이러한 개별 단계를 단일 작업으로 결합합니다.


**리전 추가** 를 클릭 하여 글로벌 클러스터를 프로비저닝합니다.


!!! 주의
    기존 DB 클러스터를 기반으로 글로벌 클러스터를 생성하는 것은 원활한 작업이며 워크로드가 중단되지 않습니다. 위에서 시작된 부하 생성기의 성능 메트릭을 모니터링하여 유효성을 검사 할 수 있습니다.



보조 DB 클러스터 및 인스턴스를 포함한 글로벌 클러스터를 프로비저닝하는데 최대 30분이 걸릴 수 있습니다.

DB 클러스터에 연결하고 사용을 시작하려면 DB 클러스터 엔드포인트를 검색해야합니다. 일반 DB 클러스터와 달리 **읽기 엔드포인트** 만 프로비저닝됩니다. **클러스터 엔드포인트**는 보조 DB 클러스터는 읽기를 포함하기 때문에, 프로비저닝되지 않으며, 쓰기를 받아 들일 수 없습니다. **읽기 엔드포인트**는 항상 읽기 DB 인스턴스중 하나에 해결할 수와 낮은 지연 시간은 그 리전내에서 읽기 작업을 위해 사용되어야한다. RDS 콘솔에서  `auroralab-mysql-secondary`라는 보조 DB 클러스터의 클러스터 DB 식별자를 클릭하여 DB 클러스터 세부 정보보기로 이동합니다.

**엔드포인트** 섹션 **연결 및 보안**에서 세부 사항 페이지가 표시 엔드포인트 탭을 선택합니다. 나중에 사용할 것이므로이 값을 적어 두십시오.


<span class="image">![RDS Cluster Secondary Endpoints](rds-cluster-secondary-endpoints.png?raw=true)</span>


