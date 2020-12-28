# Amazon Aurora Labs for MySQL

<div class="aurora"><img src="/assets/images/amazon-aurora.svg"></div>

<a href="https://aws.amazon.com/rds/aurora/details/mysql-details/" target="_blank">Amazon Aurora MySQL</a> 데이터베이스에 대한 AWS 워크샵 및 실습 콘텐츠 포털에 오신것을 환영합니다! 여기에서 Amazon Aurora 기능에 대한 이해를 돕기위한 워크샵 및 기타 실습 콘텐츠들을 배우실 수 있습니다.

이 사이트에는 예제와 오로라를 시작하는데 도움이되는 템플릿 및 실습 랩을 지원하는 작업을 자동화하는 스크립트가 포함된 따라하기 쉬운 내용이 포함되어 있습니다. 이러한 리소스는 Amazon Aurora MySQL 데이터베이스의 고급 기능이 작동하는 방식을 발견하는데 중점을 둡니다. AWS 및 MySQL 기반 데이터베이스에 대한 사전 전문 지식이 도움이 되지만 실습을 완료하는데 필요하지는 않습니다.


## 실습 개요

현재 다음 실습을 사용할 수 있습니다. 해당 주제에 대한 실습을 보려면 관련 탭을 클릭하십시오.

=== "필수 구성 요소"
    다른 실습을 실행하기 전에 다음 사전 요구 사항을 완료해야 합니다. **이것을 먼저하십시오!**

    # | 실습 모듈 | 추천 | 개요
    --- | --- | --- | ---
    P1 | [**실습 환경 사용 시작**](/prereqs/environment/) | **필수, 여기부터 시작하세요** | 랩 환경을 설정하고 필수 리소스를 프로비저닝 합니다.
    P2 | [**Session Manager를 이용하여 워크 스테이션에 연결**](/prereqs/connect/) | **필수** | Session Manager를 사용하여 EC2 기반 워크 스테이션에 연결하면 데이터베이스를 사용할 수 있습니다.

=== "프로비저닝"
    # | 실습 모듈 | 추천 | 개요
    --- | --- | --- | ---
    R1 | [**DB 클러스터 생성**](/provisioned/create/) | 선택 | 새로운 Amazon Aurora MySQL DB 클러스터를 수동으로 생성합니다. 자동으로 프로비저닝된 클러스터를 사용하여 환경을 배포할 수도 있으므로 이는 선택 사항입니다.
    R2 | [**DB 연결, 데이터로드 및 오토 스케일**](/provisioned/interact/) | 추천 | DB 클러스터에 연결하고 S3에서 초기 데이터 세트를 로드하고 읽기 전용 복제본 Auto Scaling을 테스트합니다. 초기 데이터 세트는 후속 실습에서 사용될 수 있습니다.
    R3 | [**DB 클러스터 복제**](/provisioned/clone/) | 추천 | Aurora DB 클러스터를 복제하고 데이터 세트의 차이를 확인합니다.
    R4 | [**DB 클러스터 역추적**](/provisioned/backtrack/) | 추천 | Aurora DB 클러스터를 역추적하여 잘못된 DDL 작업을 수정합니다.
    R5 | [**성능 개선 도우미 사용**](/provisioned/perf-insights/) | 추천 | RDS 성능 개선 도우미를 사용하여 DB 인스턴스의 성능을 확인합니다.
    R6 | [**내결함성 테스트**](/provisioned/failover/) | 추천 | Amazon Aurora MySQL의 장애 조치 프로세스와 최적화 방법을 살펴봅니다.
    R7 | [**데이터베이스 활동 스트림 설정**](/provisioned/das/) | 추천 | 데이터베이스 활동 스트림을 사용하여 데이터베이스 활동을 모니터링 합니다.


=== "서버리스"
    # | 실습 모듈 | 추천 | 개요
    --- | --- | --- | ---
    S1 | [**Aurora 서버리스 DB 클러스터 생성**](/serverless/create/) | 필수 | 새로운 Amazon Aurora Serverless MySQL DB 클러스터를 수동으로 생성합니다.
    S2 | [**AWS Lambda 함수와 함께 Aurora 서버리스 사용**](/serverless/dataapi/) | 추천 | RDS Data API 및 Lambda 함수를 사용하여 DB 클러스터에 연결합니다.



=== "글로벌 데이터베이스"
    # | 실습 모듈 | 추천 | 개요
    --- | --- | --- | ---
    G1 | [**글로벌 데이터베이스 배포**](/global/deploy/) | 필수 | 여러 지역에 걸쳐있는 글로벌 데이터베이스를 만듭니다.
    G3 | [**애플리케이션 연동**](/global/biapp/) | 추천 | BI 애플리케이션을 글로벌 데이터베이스에 연결합니다.
    G4 | [**글로벌 데이터베이스 모니터링**](/global/monitor/) | 추천 | Amazon CloudWatch 대시 보드를 생성하여 글로벌 데이터베이스의 지연 시간, 복제된 I/O 및 교차 리전 복제 데이터 전송을 모니터링합니다.
    G5 | [**글로벌 데이터베이스 장애 조치**](/global/failover/) | 추천 | 리전 장애 및 DR 시나리오를 시뮬레이션하고 글로벌 데이터베이스에서 보조 리전을 승격합니다.
    G6 | [**글로벌 데이터베이스 장애 복구**](/global/failback/) | 선택 | 장애 조치 후 글로벌 데이터베이스의 원래 기본 리전에서 전체 작업을 복원합니다.



=== "기계 학습"
    # | 실습 모듈 | 추천 | 개요
    --- | --- | --- | ---
    M1 | [**개요 및 전제 조건**](/ml/overview/) | 필수 | 기계학습 통합을 위한 샘플 스키마 및 데이터를 설정합니다.
    M2 | [**Aurora와 함께 Comprehend 사용**](/ml/comprehend/) | 추천 | Aurora를 Comprehend Sentiment Analysis API와 통합하고 SQL 명령을 통해 감정 분석 추론을 수행합니다.
    M3 | [**Aurora와 함께 SageMaker 사용**](/ml/sagemaker/) | 추천 | Aurora를 SageMaker Endpoints와 통합하여 SQL 명령을 사용하여 데이터 세트에서 고객 이탈을 추론합니다.
    M4 | [**리소스 정리**](/ml/cleanup/) | 추천 | 실습후 불필요한 AWS 리소스를 제거합니다.      


[실습 및 워크샵과 관련된](/related/labs/)페이지에서 Amazon Aurora와 관련된 다른 실습, 실습 및 워크샵을 찾을 수도 있습니다 .

## 한눈에 보는 랩 환경

실습시 경험을 단순화하기 위해 실습 환경에 필요한 리소스를 프로비저닝하는 <a href="https://aws.amazon.com/cloudformation/" target="_blank">AWS CloudFormation</a>용 기본 템플릿을 제공합니다. 이러한 템플릿은 일관된 네트워킹 인프라와 랩에서 사용되는 소프트웨어 패키지 및 구성 요소의 클라이언트측 환경을 배포하도록 설계되었습니다.

<div class="architecture"><img src="/assets/images/generic-architecture.png"></div>

CloudFormation을 사용하여 배포 된 환경에는 여러 구성 요소가 포함됩니다.


*	퍼블릭 및 프라이빗 서브넷이있는 <a href="https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html" target="_blank">Amazon VPC</a> 네트워크 구성
*	클러스터 및 워크 스테이션에 대한 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html#USER_VPC.Subnets" target="_blank">데이터베이스 서브넷 그룹</a> 및 관련 <a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html" target="_blank">보안 그룹</a>
*	실습에 필요한 소프트웨어 컴포넌드들로 구성된 <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Instances.html" target="_blank">Amazon EC2 인스턴스</a>
*	<a href="https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_Monitoring.OS.html" target="_blank">향상된 모니터링</a>, S3 액세스 및 로깅을 위한 워크 스테이션 및 클러스터에 대한 액세스 권한이 있는 <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html" target="_blank">IAM 역할</a>
*	Amazon Aurora 클러스터를 위한 사용자 지정 클러스터 및 DB 인스턴스 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithParamGroups.html" target="_blank">파라미터 그룹</a>으로 로깅 및 성능 스키마 활성화
*	옵션, 2개 노드가 있는 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html" target="_blank">Amazon Aurora</a> DB 클러스터 : 쓰기 및 읽기 전용 복제본
*   클러스터가 생성된 경우 마스터 데이터베이스 자격 증명이 자동으로 생성되어 <A href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html" target="_blank">AWS Secrets Manager</a>에 저장됩니다.
*	옵션, <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Integrating.AutoScaling.html" target="_blank">읽기 전용 복제본 자동 확장</a> 구성
*	옵션, 로드 테스트를 실행하기위한 <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html" target="_blank">AWS Systems Manager</a> command 문서


## 실습에 필요한 추가 소프트웨어

최신 웹 브라우저를 제외하고 이 실습에 사용중인 컴퓨터에는 특별한 소프트웨어가 필요하지 않습니다. 랩 환경을 설정하는 템플릿 및 스크립트는 랩 배포 및 실행을 위해 랩 환경에 다음 소프트웨어를 설치합니다.


* [mysql-client](https://dev.mysql.com/doc/refman/5.6/en/programs-client.html) 패키지. MySQL 오픈 소스 소프트웨어는 GPL 라이선스에 따라 제공됩니다.
* GPL 라이선스를 사용하여 [sysbench](https://github.com/akopytov/sysbench)를 사용할 수 있습니다. 
* Creative Commons Attribution-Share Alike 3.0 Unported License를 사용하여 [test_db](https://github.com/datacharmer/test_db)를 사용할 수 있습니다.
* Apache License 2.0을 사용하여 [Percona sysbench-tpcc](https://github.com/Percona-Lab/sysbench-tpcc)를 사용할 수 있습니다. available using the Apache License 2.0.
* Apache License 2.0을 사용하여 [Apache Superset](https://superset.apache.org/index.html)을 사용할 수 있습니다.
