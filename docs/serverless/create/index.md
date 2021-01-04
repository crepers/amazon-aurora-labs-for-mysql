# Aurora 서버리스 DB 클러스터 생성

이 실습에서는 Amazon Aurora Serverless 데이터베이스 클러스터를 수동으로 생성하고 클러스터의 스케일링 파라미터를 구성하는 단계를 안내합니다.

이 실습에는 다음 작업이 포함됩니다.

1. 서버리스 DB 클러스터 생성
2. 자격 증명을 저장할 암호 만들기

이 실습에서는 먼저 다음 실습 모듈을 완료해야합니다.

* [시작](/prereqs/environment/)(DB 클러스터를 자동으로 프로비저닝 할 필요가 없음)


## 1. 서버리스 DB 클러스터 생성

아직 서비스 콘솔이 열려있지 않은 경우, <a href="https://console.aws.amazon.com/rds/home" target="_blank">Amazon RDS 서비스 콘솔 </a>을 엽니다.

**데이터베이스 생성** 버튼을 클릭하여 생성 프로세스를 시작합니다.

<span class="image">![Create Database](1-create-database.png?raw=true)</span>

**데이터베이스 생성** 페이지의 첫 번째 구성 섹션 데이터베이스 생성 방식 선택이 **표준 생성**으로 선택되어 있는지 확인합니다.

다음으로 **엔진 옵션** 섹션에서 **Amazon Aurora** 엔진 옵션, 에디션은 **MySQL과 호환되는 Amazon Aurora**, 용량 유형은 **서버리스**, 버전은 **Aurora (MySQL)-5.6.10a**를 선택합니다. 지금까지 선택한 사항은 AWS가 서버리스 구성에서 최신 버전의 MySQL 5.6 호환 엔진을 사용하여 Aurora MySQL 데이터베이스 클러스터를 생성하도록 하게합니다.

<span class="image">![Engine Options](1-engine-options.png?raw=true)</span>

**설정** 섹션에서 DB 클러스터 식별자를 `auroralab-mysql-serverless`로 설정합니다. 데이터베이스에서 가장 높은 권한으로 데이터베이스 마스터 사용자의 이름과 암호를 구성합니다. 후속 실습과의 일관성을 위해 사용자 이름 'masteruser'를 사용하고 선택한 비밀번호를 사용하는 것이 좋습니다. 편의상 * 암호 자동 생성** 확인란이 **체크되지 않음**을 확인합니다.

<span class="image">![Database Settings](1-serverless-settings.png?raw=true)</span>

**용량 설정** 섹션에서 **최소 Aurora 용량 단위**를 `1 (2GB RAM)` 으로 선택하고 **최대 Aurora 용량 단위**를 `16 (32GB RAM)`으로 선택합니다. 그런 다음 **추가 조정 구성** 섹션을 확장하고 **다음 시간(분) 동안 비활성 상태 지속 후 컴퓨팅 용량 일시 정지** 옆의 확인란을 **선택**합니다. 이 구성을 통해 Aurora 서버리스는 1개의 용량 단위에서 32개의 용량 단위 사이에서 DB 클러스터의 용량을 확장하고 5분 동안 활동이 없으면 용량을 완전히 중지 할 수 있습니다.

**연결** 섹션에서 **추가 연결 구성**이라는 하위 섹션을 확장합니다. 이 섹션에서는 정의된 네트워크 구성 내에서 데이터베이스 클러스터를 배포 할 위치를 지정할 수 있습니다. Aurora 데이터베이스 클러스터에 필요한 모든 리소스를 포함하는 VPC로 환경이 배포되었습니다. 여기에는 VPC 자체, 서브넷, DB 서브넷 그룹, 보안 그룹 및 기타 여러 네트워킹 구성이 포함됩니다. 이 섹션에서 적절한 기존 연결 제어를 선택하기만 하면 됩니다.

`auroralab-vpc` 라는 **Virtual Private Cloud (VPC)**를 선택합니다. 실습 환경에서는 실습 작업 공간 EC2 인스턴스가 데이터베이스에 연결할 수 있도록 **VPC 보안 그룹**도 구성했습니다. **기존 항목 선택** 보안 그룹 옵션이 선택되어 있는지 확인하고 드롭 다운에서`auroralab-database-sg`라는 보안 그룹을 선택합니다. 선택에서 `default` 와 같은 다른 보안 그룹을 제거하십시오.

<span class="image">![Capacity and Connectivity](1-serverless-capacity.png?raw=true)</span>

다음으로 **추가 구성** 섹션을 확장합니다. **초기 데이터베이스 이름** 텍스트 상자에 `mylab` 을 입력합니다. **백업 보관 기간**을 `1 일` 로 선택합니다. **삭제 방지 활성화** 확인란을 **선택 취소**합니다. 프로덕션 사용 사례에서는 해당 옵션을 선택된 상태로 두어야하지만 테스트 목적으로 이 옵션을 선택 취소하면 실습을 완료한 후 리소스를 더 쉽게 정리할 수 있습니다.

<span class="image">![Advanced configuration](1-serverless-advconfig.png?raw=true)</span>

??? tip "이러한 선택은 무엇을 의미합니까?"
     다음과 같은 특성을 가진 데이터베이스 클러스터를 생성합니다.

     * Aurora MySQL 5.6 호환 (최신 엔진 버전)
     * 서버리스 db 클러스터는 1 ~ 16 용량 단위로 확장되고, 5분 동안 활동이 없으면 컴퓨팅 용량 일시 중지
     * VPC에 배포되고 랩 환경의 네트워크 구성 사용
     * 연속 자동 백업, 7일 동안 백업 유지
     * 저장 데이터 암호화 사용

**데이터베이스 생성**을 클릭하여 DB 클러스터를 프로비저닝합니다.

데이터베이스 목록으로 돌아가면, 목록에서 새 데이터베이스의 이름을 클릭합니다.

<span class="image">![Select Cluster](1-serverless-selection.png?raw=true)</span>

클러스터의 세부정보 보기에서 **구성** 탭을 클릭합니다. **ARN**의 값을 확인합니다. 이것을 적어 두십시오. 나중에 필요합니다.

<span class="image">![CLuster ARN](1-serverless-arn.png?raw=true)</span>


## 2. 자격 증명을 저장할 암호를 만듭니다.

<a href="https://console.aws.amazon.com/secretsmanager/home" target="_blank"> AWS Secrets Manager 서비스 콘솔 </a>을 엽니다.

**새 보안 암호 저장**을 클릭하여 구성 프로세스를 시작합니다.

<span class="image">![Create Secret](2-create-secret.png?raw=true)</span>

**보안 암호 유형 선택** 섹션에서 **RDS 데이터베이스에 대한 자격 증명**을 선택한 다음 서버리스 DB 클러스터를 만들때 제공한 **사용자 이름** (예 :`masteruser`) 및 **비밀번호**를 입력합니다.

다음으로 **이 보안 암호로 액세스할 RDS 데이터베이스 선택** 섹션에서 클러스터에 할당한 DB 클러스터 식별자 (예 :`auroralab-mysql-serverless`)를 선택합니다. **다음**을 클릭합니다.

<span class="image">![Configure Secret](2-config-secret.png?raw=true)</span>

보안 암호의 이름을 `auroralab-mysql-serverless-secret` 으로 지정하고 암호에 대한 관련 설명을 입력한 후 **다음**을 클릭합니다.

<span class="image">![Name Secret](2-name-secret.png?raw=true)</span>

마지막으로 **자동 교체 구성** 섹션에서 **자동 교체 비활성화** 옵션을 선택된 상태로 둡니다. 프로덕션 환경에서는 추가 보안을 위해 자동으로 교체되는 데이터베이스 자격 증명을 사용하는것이 좋습니다. **다음**을 클릭합니다.

<span class="image">![Rotate Secret](2-rotate-secret.png?raw=true)</span>

**검토** 섹션에서 새 보안 암호가 생성되기 전에 구성 정보를 확인할 수 있습니다. 또한 널리 사용되는 프로그래밍 언어로 된 샘플 코드를 검색할 수 있으므로 애플리케이션에 대한 보안 암호를 쉽게 개발할 수 있습니다. 화면 하단의 **저장**을 클릭합니다.

생성되면 새로 생성된 보안 암호의 **ARN**을 확인니다. 이 값은 후속 실습에서 필요합니다. 콘솔의 **보안 암호** 목록에서 새로 생성된 보안 암호의 이름을 클릭합니다.

<span class="image">![List Secrets](2-list-secrets.png?raw=true)</span>

보안 암호 세부 정보보기에서 **보안 암호 ARN**의 값을 확인합니다. 이것을 적어 두십시오. 나중에 필요합니다.

<span class="image">![Secret ARN](2-arn-secret.png?raw=true)</span>
