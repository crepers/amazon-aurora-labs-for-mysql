# 실습 환경을 사용하여 시작하기


실습 환경에서 리소스를 사용하려면 몇 단계를 완료해야합니다.

워크샵이나 기타 교육과 같은 공식 행사에 참석하는 경우 강사가 시작 방법에 대한 구체적인 지침을 제공합니다. 일반적으로 **Event Engine** 이라는 플랫폼을 통해 AWS 계정이 제공되며 실습 환경은 이미 해당 계정에 배포되어 있습니다. **Event Engine 사용 워크샵**인 경우 아래 탭을 선택합니다.

이 실습을 자신의 계정으로 직접 시도하거나 공식 이벤트에 참석하고 강사가 실습 환경을 수동으로 배포하도록 특별히 지시한 경우 아래에서 **실습 환경을 수동으로 배포해야 함** 탭을 선택하십시오.

상황에 가장 적합한 옵션을 선택하십시오.


=== "Event Engine을 사용하는 워크샵에 있습니다."
    <h4>이벤트 엔진에 로그인</h4>

    워크숍 시작시 **12자리 액세스 코드**가 제공되었습니다. 이 액세스 코드는 이 워크샵의 목적으로 전용 AWS 계정을 사용할 수 있는 권한을 부여합니다.

    <a href="https://dashboard.eventengine.run/" target="_blank">**https://dashboard.eventengine.run/**</a>로 이동하여 액세스 코드를 입력하고 **진행**을 클릭합니다.


    <span class="image">![EventEngine Login](1-ee-login.png?raw=true)</span>

    **Team Dashboard** 에서 **AWS Console**을 클릭하십시오. AWS 관리 콘솔에 로그인 할 수 있습니다.

    <span class="image">![EventEngine Dashboard](1-ee-dashboard.png?raw=true)</span>

    **Open Console**을 클릭합니다. 이 워크샵에서는 명령줄 및 API 액세스 자격 증명을 사용할 필요가 없습니다.

    <span class="image">![EventEngine Open Console](1-ee-open-console.png?raw=true)</span>

    <h4>실습환경 Parameters 가져오기</h4>

    Back on the **Team Dashboard** web page (browser tab), close the **AWS Console Login** modal window (shown above) using the `x` in the top right corner, or the **OK** button, and scroll down.
    **Team Dashboard** 웹 페이지(브라우저 탭)로 돌아가서 **AWS Console Login** 오른쪽 상단 모서리에 있는 `x`를 사용하여 AWS 콘솔 로그인 모달 창 (위에 표시됨)을 닫거나 **OK** 버튼을 누르고 아래로 스크롤 합니다.

    실습중에 필요한 매개 변수 세트가 표시됩니다. 여기서 **Parameter**에 나타나는 이름은 매개 변수 형식을 사용하여 후속 실습에서 직접 참조됩니다. Parameter를 후속 실습에 표시된 ==[parameters]== 에서 해당 값으로 바꿉니다.

    <span class="image">![Stack Outputs](1-ee-outputs.png?raw=true)</span>

    이 단계를 완료하면 다음 실습을 계속할 수 있습니다. [**Session Manager 워크스테이션에 연결**](/prereqs/connect/)


=== "실습 환경을 수동으로 배포해야 합니다."
    <h4>AWS Management Console에 연결</h4>

    공식적인 교육 설정에서 이러한 실습을 실행하는 경우 콘솔 URL과 제공된 자격 증명을 사용하여 AWS Management Console에 연결하고 로그인하십시오. 그렇지 않으면 자신의 자격 증명을 사용하십시오. <a href="https://console.aws.amazon.com/" target="_blank">https://console.aws.amazon.com/</a>에서 또는 회사에서 제공하는 SSO(Single Sign-On) 메커니즘을 통해 콘솔에 액세스할 수 있습니다.



    <span class="image">![AWS Management Console Login](2-login.png?raw=true)</span>

    공식적인 교육에서 이러한 실습을 실행하는 경우 **제공된 AWS 리전을 사용하십시오.** 올바른 리전을 선택하지 않는 경우 오른쪽 상단 모서리에서 올바른 AWS 리전이 선택되었는지 확인합니다. 실습은 Amazon Aurora MySQL과 호환되는 모든 리전에서 작동하도록 설계되었습니다. 그러나 현재 Amazon Aurora의 모든 기능을 지원되는 모든 리전에서 사용할 수 있는 것은 아닙니다.



    !!! warning "글로벌 데이터베이스 실습을 위한 지역"
        **Aurora 글로벌 데이터베이스** 실습을 실행하려는 경우 **미국 동부(버지니아 북부)/us-east-1**과 다른 리전을 선택하십시오. 이 지역은 해당 실습의 보조 지역으로 사용되며 글로벌 데이터베이스의 기본 및 보조 지역은 달라야합니다.


    <span class="image">![AWS Management Console Region Selection](2-region-select.png?raw=true)</span>

    <h4>AWS CloudFormation을 사용하여 실습 환경 배포</h4>

    실습 시작 경험을 단순화하기 위해 실습 환경에 필요한 리소스를 프로비저닝하는 <a href="https://aws.amazon.com/cloudformation/" target="_blank">AWS CloudFormation</a>용 기본 템플릿을 만들었습니다. 이러한 템플릿은 일관된 네트워킹 인프라와 실습에서 사용되는 소프트웨어 패키지 및 구성 요소의 클라이언트측 환경을 배포하도록 설계되었습니다.

    실행하려는 실습에 따라 가장 적합한 CloudFormation 템플릿을 선택하고 **스택 생성**을 클릭하십시오.

    Option | One-Click 시작
    --- | ---
    **DB 클러스터를 수동으로 생성하겠습니다.** | <a href="https://console.aws.amazon.com/cloudformation/home#/stacks/create/review?stackName=auroralab&templateURL=https://s3.amazonaws.com/ams-stack-prod-content-us-east-1/templates/lab_template.yml&param_deployCluster=No" target="_blank"><img src="/assets/images/cloudformation-launch-stack.png" alt="Launch Stack"></a>
    **자동으로 Aurora 프로비저닝된 DB 클러스터 생성** | <a href="https://console.aws.amazon.com/cloudformation/home#/stacks/create/review?stackName=auroralab&templateURL=https://s3.amazonaws.com/ams-stack-prod-content-us-east-1/templates/lab_template.yml&param_deployCluster=Yes" target="_blank"><img src="/assets/images/cloudformation-launch-stack.png" alt="Launch Stack"></a>

    ??? tip "CloudFormation 템플릿을 볼 수 있습니까?""
        예, CloudFormation 템플릿을 다운로드하고 생성중인 리소스 및 구성 방법을 검토하는 것이 좋습니다.

        [CloudFormation 템플릿 다운로드](https://[[website]]/templates/lab_template.yml)



    !!! warning "리전 확인"
        특히 위의 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 올바른 지역에서 여전히 작업하고 있는지 확인하십시오. 또한 Aurora Global Databsase 실습을 실행하려는 경우 리전은 미국 동부 (버지니아 북부)/us-east-1이 아니어야 합니다.


    **스택 이름** 필드에서 값 `auroralab`이 설정되어 있는지 확인하십시오. 

    Aurora 글로벌 데이터베이스 실습을 실행할 계획이라면 **Enable Aurora Global Database Labs?**에서 **Yes** 를 선택하십시오.

    Aurora 기계학습 통합 실습을 실행할 계획이라면 **Enable Aurora ML Labs?** 에서 **Yes** 를 선택하십시오.

    <span class="image">![Create Stack](2-create-stack-params.png?raw=true)</span>

    페이지 하단으로 스크롤하여 다음 확인란을 선택합니다. **AWS CloudFormation에서 사용자 지정 이름으로 IAM 리소스를 생성할 수 있음을 승인합니다.** 체크하고 **스택 생성**을 클릭합니다.


    <span class="image">![Create Stack](2-create-stack-confirm.png?raw=true)</span>

    스택을 프로비저닝하는데 약 20분이 소요되며 **스택 세부정보** 페이지에서 상태를 모니터링 할 수 있습니다. **이벤트** 탭을 새로고쳐 스택 생성 프로세스의 진행 상황을 모니터링할 수 있습니다. 목록의 최신 이벤트는 스택 리소스에 대한`CREATE_COMPLETE`를 나타냅니다.



    <span class="image">![Stack Status](2-stack-status.png?raw=true)</span>


    스택 상태가 `CREATE_COMPLETE`이면 **출력** 탭을 클릭합니다. 여기에있는 값은 나머지 실습을 완료하는데 중요합니다. 잠시 시간을내어 이 값을 나머지 실습 중에 쉽게 액세스 할 수있는 위치에 저장하십시오. **키** 열에 나타나는 이름은 매개 변수 형식을 사용하여 후속 단계의 지침에서 직접 참조됩니다. ==[outputKey]==


    <span class="image">![Stack Outputs](2-stack-outputs.png?raw=true)</span>

    <h4>실습환경 확인</h4>


    워크스테이션이 올바르게 구성되었는지 확인하십시오.

    * bastionInstance CloudFormation 스택 출력 키에 `i-0123456789abcdef0`와 비슷한 값(값은 다를 수 있음)이 표시 됩니까?

    그렇다면 다음 실습으로 진행할 수 있습니다. [**Session Manager 워크스테이션에 연결**](/prereqs/connect/). 그렇지 않으면 위의 지침을 다시 확인하십시오. 단계를 놓쳤을 수 있습니다.

