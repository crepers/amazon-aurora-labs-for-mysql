# Session Manager Workstation에 연결

Aurora 데이터베이스 클러스터에 실습하려면 워크 스테이션처럼 작동하는  <a href="https://aws.amazon.com/ec2/" target="_blank">Amazon EC2</a> Linux 인스턴스를 사용 하여이 웹 사이트의 실습에서 AWS 리소스와 상호 작용합니다. 필요한 모든 소프트웨어 패키지와 스크립트가이 EC2 인스턴스에 설치 및 구성되었습니다. 통합된 경험을 보장하기 위해 <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html" target="_blank">AWS Systems Manager Session Manager</a>를 사용하여이 워크스테이션과 상호작용 합니다. Session Manager를 사용하면 자신의 장치에 소프트웨어를 설치할 필요없이 관리 콘솔에서 직접 워크 스테이션과 상호 작용할 수 있습니다.

이 실습에는 다음 작업이 포함됩니다.

1. 워크스테이션 인스턴스에 연결
2. 실습 환경 확인

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/)

## 1. 워크스테이션 인스턴스에 연결

<a href="https://console.aws.amazon.com/systems-manager/session-manager" target="_blank">Systems Manager: Session Manager</a> 서비스 콘솔을 엽니다. **세션 시작** 버튼을 클릭 합니다.

!!! warning "리전 확인"
    위의 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 올바른 리전에서 여전히 작업하고 있는지 확인하십시오.

<span class="image">![Start Session](1-start-session.png?raw=true)</span>

세션을 설정할 EC2 인스턴스를 선택하십시오. 워크스테이션의 이름이 `auroralab-mysql-workstation`인 워크스테이션을 선택하고 **세션 시작**을 클릭합니다.


<span class="image">![Connect Instance](1-connect-session.png?raw=true)</span>

터미널 화면 및 프롬프트와 같은 검은색 명령이 표시되면 이제 워크스테이션에 연결된 것입니다. 다음 명령을 입력하여 일관된 경험을 보장하고 성공적으로 연결되었는지 확인하십시오.


```shell
sudo su -l ubuntu
```

!!! warning "Linux 사용자 계정"
    기본적으로 세션 관리자는 사용자 계정 **ssm-user**를 사용하여 연결합니다. 위의 명령을 사용하여 사용자 계정을 실습에 필요한 올바른 설정이있는 **ubuntu** 사용자 계정으로 변경합니다. ```whoami``` 을 입력하여 언제든지 현재 사용자 계정을 확인할 수 있습니다. 그러면 현재 사용자 계정이 인쇄됩니다.

	후속 실습에서 실습 명령에 액세스하는 동안 오류가 발생하면 위 명령을 사용하여 사용자 계정이 변경되지 않았기 때문일 수 있습니다.


<span class="image">![Terminal Connected](1-terminal-sudo.png?raw=true)</span>

## 2. 실습 환경 확인

워크스테이션이 올바르게 구성되었는지 확인하십시오. 세션 관리자 명령줄에 다음 명령을 입력합니다.


```shell
tail -n1 /debug.log
```

`* bootstrap complete, rebooting` 출력이 표시되어야 합니다. 표시되는 출력과 다른 경우 몇 분 더 기다렸다가 다시 시도하십시오.

환경이 올바르게 구성되었는지 확인했으면 다음 실습으로 진행할 수 있습니다.

