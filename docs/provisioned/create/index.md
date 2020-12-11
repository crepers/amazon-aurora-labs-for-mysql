#  Aurora 클러스터 생성

이 실습에서는 Amazon Aurora 데이터베이스 클러스터를 수동으로 생성하고 클러스터 구성 요소에 필요한 파라미터를 앱에 구성하는 단계를 안내합니다. 이 실습이 끝나면 다음 실습에서 사용할 수있는 데이터베이스 클러스터가 준비됩니다.

!!! tip "DB 클러스터를 생성하는 방법을 이미 알고 있습니까?"
    Amazon Aurora MySQL의 기본 개념에 익숙하고 과거에 클러스터를 생성한 경우 DB 클러스터를 자동으로 배포하는 옵션을 사용하여 실습 환경을 프로비저닝하여 이 모듈을 건너 뛸 수 있습니다. 랩 환경 프로비저닝에 대한 자세한 내용은 [시작](prereqs/environment/)하기 사전 조건 모듈을 참조하십시오. [DB 연결, 데이터로드 및 오토 스케일](/provisioned/interact/)로 건너 뜁니다.


이 실습에는 다음 작업이 포함됩니다.

1. DB 클러스터 생성
2. DB 클러스터 엔드 포인트 검색
3. DB 클러스터에 IAM 역할 할당
4. 복제본 Auto Scaling 정책 생성
5. AWS Secrets Manager 암호 생성
6. 클라이언트 워크 스테이션 구성
7. DB 클러스터 확인

이 실습에서는 먼저 다음 실습 모듈을 완료해야합니다.

* [시작하기](/prereqs/environment/) (DB 클러스터를 자동으로 프로비저닝 할 필요가 없음)
* [Session Manager Workstation에 연결](/prereqs/connect/) (작업 \#6에 필요)



## 1. DB cluster 만들기

DB 클러스터 생성을 위해 <a href="https://console.aws.amazon.com/rds/home" target="_blank">Amazon RDS 서비스 콘솔</a>을 엽니다.

!!! warning "리전 확인"
    서비스 콘솔의 오른쪽 상단에서 올바른 리전(Region)에서 작업하고 있는지 확인하십시오.

**데이터베이스 만들기**를 클릭합니다.

<span class="image">![Create Database](1-create-database.png?raw=true)</span>

새 DB 클러스터의 구성 화면에서 다음 옵션을 설정합니다.

**데이터베이스 생성 방법 선택** 섹션에서 :

* [ ] **표준 생성** 옵션이 선택 되었는지 확인합니다.


**엔진 옵션** 섹션에서 :

* [ ] `Amazon Aurora` 엔진을 선택하십시오.
* [ ] `MySQL과 호환되는 Amazon Aurora` 에디션을 선택하십시오.
* [ ] `Aurora (MySQL 5.7) 2.08.1`버전을 선택하십시오.
<!-- * [ ] Select the `Regional` database option. -->

<span class="image">![Engine Options](1-engine-options.png?raw=true)</span>

**복제 기능** 섹션에서 :

* [ ] `단일 마스터` 옵션을 선택하십시오.


**템플릿** 섹션에서 :

* [ ] `프로덕션` 옵션을 선택하십시오.


**설정** 섹션에서 :

* [ ] **DB 클러스터 식별자** 를 `auroralab-mysql-cluster`로 설정합니다. 이 이름을 사용하면 후속 실습에서 명령을 편집 할 필요가 없습니다.
* [ ] **마스터 사용자 이름**을 `masteruser`로 설정합니다. 이것은 데이터베이스에서 가장 높은 권한을 가진 사용자 계정이며 다른 이름을 사용할 수 있지만 후속 실습에서 명령을 편집해야 할 수도 있습니다.
* [ ] **마스터 암호**를 원하는 기억할 수있는 값으로 설정하고 암호를 확인하십시오.
* [ ] **암호 자동 생성** 확인란이 **선택되어 있지 않은지** 확인 합니다.

??? info 이러한 선택은 무엇을 의미합니까?
    지금까지 선택한 사항은 클러스터에 하나의 쓰기 및 읽기 전용 데이터베이스 인스턴스가 있는 고가용성 구성에서 지정된 버전의 MySQL 5.7 호환 엔진을 사용하여 Aurora MySQL 데이터베이스 클러스터를 AWS에서 생성합니다. 클러스터를 구동하는 특정 유형의 컴퓨팅 용량을 나타내므로 이를 **Aurora 프로비저닝된 DB 클러스터**라고 합니다. Aurora 서버리스 DB 클러스터를 생성하고 Aurora 서버리스 기능을 테스트하려는 경우 [다른 실습](/serverless/create/)이 있습니다 .

<span class="image">![Database Settings](1-db-settings.png?raw=true)</span>

**DB 인스턴스 크기** 섹션에서 :

* [ ] **Memory Optimized classes**를 선택하고, `r5.large` 크기의 인스턴스를 선택하십시오.

**가용성 및 내구성** 섹션에서 :

* [ ] `다른 AZ에 Aurora 복제본/리더 노드 생성(확장된 가용성에 권장)`를 선택하십시오.

<span class="image">![Database Settings](1-db-size.png?raw=true)</span>

**연결** 섹션에서 :

* [ ] **추가 연결 구성** 이라는 하위 섹션을 확장합니다. 이 섹션에서는 정의된 네트워크 구성 내에서 데이터베이스 클러스터를 배포 할 위치를 지정할 수 있습니다. 실습을 단순화하기 위해 모든 네트워킹 리소스를 구성했습니다.
* [ ] **Virtual Private Cloud (VPC)** 에서 `auroralab-vpc`를 선택하십시오.
* [ ] 올바른 VPC를 선택하면 DB 서브넷 그룹이 자동으로 선택됩니다. 선택이 올바른지 확인하십시오. DB 서브넷 그룹의 이름은 `auroralab-db-subnet-group` 입니다.
* [ ] **퍼블랙 액세스 가능** 옵션이 `아니오`로 설정되어 있는지 확인하십시오.
* [ ] At **VPC security group**, make sure the **Choose existing** security group option is selected. The lab environment already provides a security group that allows your lab workspace EC2 instance to connect to the database.
* [ ] **VPC 보안 그룹** 에서, **기존 항목 선택** 옵션을 선택합니다. 실습 환경은 이미 EC2 인스턴스가 데이터베이스에 연결할 수 있도록하는 보안 그룹을 제공합니다.
* [ ] `auroralab-database-sg` 보안 그룹을 선택합니다.
* [ ] 선택에서 `default`와 같은 다른 보안 그룹을 제거하십시오 .


**데이터베이스 인증** 섹션에서 :
* [ ] `암호 및 IAM 데이터베이스 인증`을 선택하십시오. IAM 인증은 일부 후속 실습에서 사용할 수 있습니다.

<span class="image">![Connectivity](1-connectivity.png?raw=true)</span>

**추가 구성** 섹션을 확장하고 다음과 같이 옵션을 구성합니다.

* [ ] **초기 데이터베이스 이름** 을 `mylab`로 설정합니다. 여기에서도 사용자 지정 이름을 사용할 수 있지만 이후 실습에서 명령과 스크립트를 편집해야합니다.
* [ ] **DB 클러스터 파라미터 그룹** 및 **DB 파라미터 그룹** 에서, `auroralab-[...]` 또는 `mod-[...]` 로 시작이라는 그룹을 선택하십시오.
* [ ] **백업 보존 기간**을 `1 일`로 설정합니다.
* [ ] **암호화 활성화** 확인란을 선택합니다.
* [ ] **마스터 키**를 `[default] aws/rds`로 설정합니다.


<span class="image">![Advanced configuration](1-advanced-1.png?raw=true)</span>

**추가 구성** 섹션에서 계속 하십시오.

* [ ] **역추적 활성화** 확인란을 선택합니다.
* [ ] 대상 역추적 기간을 `24`시간으로 설정합니다.
* [ ] **성능 개선 도우미 활성화** 확인란을 선택합니다.
* [ ] 보존 기간을 `기본값 (7 일)`로 설정합니다.
* [ ] **마스터 키** 를 `[default] aws/rds`로 설정합니다.
* [ ] `Enhanced 모니터링 활성화` 확인란을 선택합니다.
* [ ] 세부 수준은 `1 초`로 설정합니다.

<span class="image">![Advanced configuration](1-advanced-2.png?raw=true)</span>

**추가 구성** 섹션에서:


* [ ] **로그 내보내기**에서 `에러 로그`와 `느린 쿼리 로그`를 선택합니다.
* [ ] **삭제 보호 활성화** 확인란을 선택 취소/해제합니다. 운영 환경에서 사용할때는 해당 옵션을 선택된 상태로 두어야하지만 테스트 목적으로 이 옵션을 해제하면 실습을 완료한 후 리소스를 더 쉽게 정리할 수 있습니다.

??? info 이러한 선택은 무엇을 의미합니까?
    다음과 같은 특성을 가진 데이터베이스 클러스터를 생성합니다.

    * Aurora MySQL 5.7 호환
    * 서로 다른 가용 영역에 있는 쓰기와 읽기 전용 DB 인스턴스로 구성된 클러스터(고가용성)
    * VPC에 배포되고 실습 환경의 네트워크 구성을 사용
    * 느린 쿼리 로그, S3 액세스 및 몇 가지 다른 구성 조정을 활성화하는 사용자 지정 데이터베이스 엔진 파라미터 사용
    * 연속 자동 백업, 7 일 동안 백업 보관
    * 데이터 암호화 사용
    * 역추적 목적으로 24 시간 분량의 변경 데이터 보관
    * Enhanced 모나터링 및 성능 개선 도우미 활성

<a href="#" onclick="VerifyAllCheckboxes(); return false;">확인</a>모든 선택한 내용을 확인합니다.

**데이터베이스 생성**을 클릭하여 DB 클러스터를 프로비저닝합니다.

<span class="image">![Advanced configuration - end](1-advanced-3.png?raw=true)</span>


## 2. DB 클러스터 엔드포인트 검색

클러스터를 구성하는 DB 인스턴스를 포함하여 데이터베이스 클러스터를 프로비저닝하는데 몇 분 정도 걸릴 수 있습니다. DB 클러스터에 연결하고 후속 실습에서 사용을 시작하려면 DB 클러스터 엔드포인트를 검색해야 합니다. 기본적으로 생성되는 두 개의 엔드포인트가 있습니다. **클러스터 엔드포인트**는 항상 클러스터의 **쓰기** DB 인스턴스를 가리키고, 쓰기와 읽기에 사용해야 합니다. **읽기 엔드포인트**는 항상 **읽기** DB 인스턴스 중 하나로 확인되며 읽기 작업을 읽기 전용 복제본으로 사용해야 합니다. RDS 콘솔에서 클러스터 DB 식별자를 클릭하여 DB 클러스터 세부정보 보기로 이동합니다.


<span class="image">![DB Cluster Status](2-db-cluster-status.png?raw=true)</span>

**엔드포인트** 섹션의 **연결 및 보안** 세부 사항 페이지의 엔드포인트 탭을 선택합니다. 나중에 사용할 것이므로이 값을 적어 두십시오.


<span class="image">![DB Cluster Endpoints](2-db-cluster-details.png?raw=true)</span>


## 3. DB 클러스터에 IAM 역할 할당

클러스터가 Amazon S3에 액세스하여 데이터 가져오기 및 내보내기를 할 수 있도록 IAM 역할을 DB 클러스터에 할당해야 합니다. 실습 환경에서 IAM 역할이 이미 생성되었습니다. 이전과 동일한 DB 클러스터 세부정보 페이지의 **IAM 역할 관리** 섹션에서 **이 클러스터에 추가할 IAM 역할 선택**을 선택하고 드롭 다운에서 `auroralab-integrate-[region]`로 시작하는 IAM 역할을 선택합니다. 해당 접두사가있는 두 개 이상의 역할을 사용할 수있는 경우 현재 운영중인 지역에 대한 역할을 선택합니다. 그런 다음 **역할 추가**를 클릭합니다.

<span class="image">![DB Cluster Add Role](3-add-role.png?raw=true)</span>

작업이 완료되면 역할의 **상태** 가 `수정 중`에서 `사용가능`로 변경 됩니다.

## 4. 복제본 Auto Scaling 정책 생성

마지막으로 읽기 전용 복제본 Auto Scaling 구성을 DB 클러스터에 추가합니다. 이렇게하면 DB 클러스터가 로드에 따라 특정 시점에 DB 클러스터에서 작동하는 읽기 DB 인스턴스의 수를 확장 할 수 있습니다.

세부 정보 페이지의 오른쪽 상단에서 **작업**을 클릭하여 **복제본 Auto Scaling 추가**를 클릭합니다 .


<span class="image">![DB Cluster Add Auto Scaling](4-add-as-policy.png?raw=true)</span>

스택 이름을 기반으로 정책 이름을 설정합니다 : `auroralab-autoscale-readers`. **대상 측정치**는 **Aurora 복제본의 평균 CPU 사용률**을 선택합니다. **대상 값**은 `20` 퍼센트를 입력합니다. 운영환경에서 사용에서는 이 값을 훨씬 더 높게 설정해야 할 수 있지만 데모 목적으로 더 낮은 값을 사용하고 있습니다. 다음에, **추가 구성**을 확장하고 **휴지 기간 축소**와 **휴지 기간 확장**을 모두 `180`초로 설정합니다. 그러면 후속 실습에서 확장 작업 사이에 대기해야하는 시간이 줄어듭니다.

**클러스터 용량 세부 정보** 섹션에서 **최소 용량**을 `1`로 하고 **최대 용량**을 `2`로 설정합니다. 운영환경에서 사용시에는 다른 값을 사용해야 할 수 있지만 데모 목적으로 실습과 관련된 비용을 제한하기 위해 읽기 수를 2로 제한합니다. 다음으로 **정책 추가**를 클릭 합니다.

<span class="image">![Auto Scaling Configuration](4-as-policy-config.png?raw=true)</span>


## 5. AWS Secrets Manager 암호 생성

<a href="https://console.aws.amazon.com/secretsmanager/home" target="_blank">AWS Secrets Manager</a> 서비스 콘솔을 엽니다.


!!! warning "리전 확인"
    위의 링크를 따라 서비스 콘솔을 여는 경우 올바른 지역에서 여전히 작업하고 있는지 확인하십시오.

**새 보안 암호 저장**을 클릭 하여 구성 프로세스를 시작합니다.

<span class="image">![Create Secret](5-create-secret.png?raw=true)</span>

**보안 암호 유형 선택** 섹션에서 **RDS 데이터베이스에 대한 자격 증명**을 선택합니다. 다음에 **사용자 이름** ( `masteruser`)을 입력하고 **암호**는 이전에 DB 클러스터를 만들때 입력한 암호를 입력합니다.

다음, **이 보안 비밀이 액세스할 RDS 데이터베이스 선택** 섹션에서 클러스터에 할당한 DB 클러스터 식별자를 선택합니다 (예 : `auroralab-mysql-cluster`). **다음**을 클릭합니다.


<span class="image">![Configure Secret](5-config-secret.png?raw=true)</span>

보안 암호 이름을 `secretCusterMasterUser`로 입력하고 암호에 대한 관련 설명을 입력합니다. **다음**을 클릭합니다.


<span class="image">![Name Secret](5-name-secret.png?raw=true)</span>

마지막으로 **자동 교체 구성** 섹션에서 **자동 교체 비활성화** 옵션을 선택된 상태로 둡니다. 운영 환경에서는 추가 보안을 위해 자동으로 교체되는 데이터베이스 자격 증명을 사용할 수 있습니다. **다음**을 클릭 합니다.


<span class="image">![Rotate Secret](5-rotate-secret.png?raw=true)</span>

**검토** 섹션에서는 생성하기 전에 입력한 암호 구성을 확인합니다. 또한 널리 사용되는 프로그래밍 언어로된 샘플 코드를 검색 할 수 있으므로 애플리케이션에 대한 암호를 쉽게 사용 할 수 있습니다. 화면 하단의 **저장**을 클릭 합니다.

새로 생성된 암호의 **ARN**을 식별합니다. 이 값은 후속 실습에서 필요합니다. 콘솔의 **보안 암호** 목록에서 새로 생성된 암호의 이름을 클릭합니다.

<span class="image">![List Secrets](5-list-secrets.png?raw=true)</span>

암호의 세부정보 보기에서 **보안 암호 ARN**의 값을 기록해 둡니다. 이후 실습에서 필요합니다.

<span class="image">![Secret ARN](5-arn-secret.png?raw=true)</span>


## 6. EC2 워크 스테이션 구성

이 실습에서는 DB 클러스터가 수동 또는 자동으로 생성이 되었는지에 관계없이 일관된 방법으로 DB 자격 증명에 액세스합니다. 상호 작용을 단순화하기 위해 자격 증명은 데이터베이스에 명령을 내리는데 사용하는 EC2 워크스테이션의 환경 변수에 저장됩니다. 클러스터가 자동으로 생성되면 자격 증명도 자동으로 설정됩니다. 클러스터를 수동으로 생성 할 때 몇 가지 추가 명령을 실행해야 합니다.

Session Manager 워크스테이션 CLI에 아직 연결되어 있지 않은 경우 다음 지침에 따라 연결하십시오.

워크스테이션이 연결되면 아래 명령을 실행하여 ==[secretArn]== 에 생성한 보안 암호 ARN을 입력합니다.

```shell
CREDS=`aws secretsmanager get-secret-value --secret-id [secretArn] | jq -r '.SecretString'`
export DBUSER="`echo $CREDS | jq -r '.username'`"
export DBPASS="`echo $CREDS | jq -r '.password'`"
echo "export DBPASS=\"$DBPASS\"" >> /home/ubuntu/.bashrc
echo "export DBUSER=$DBUSER" >> /home/ubuntu/.bashrc
```

## 7. DB 클러스터 확인

DB 클러스터가 제대로 생성되었는지 확인하겠습니다. 먼저 자격 증명이 환경 변수에 올바르게 저장되었는지 확인하고 다음을 실행합니다.


```shell
echo $DBUSER
```

`masteruser`이 응답 문자열로 표시되어야 합니다. 다음으로 생성된 데이터베이스 엔진의 버전을 확인합니다. 아래 명령을 실행하여 이전 단계에서 만든 클러스터 엔드포인트의 값을 ==[clusterEndpoint]== 있는 내용으로 변경합니다.


```shell
mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" -e"SELECT @@aurora_version;"
```

`2.08.1`버전 번호가 포함된 응답이 표시되어야 합니다.
