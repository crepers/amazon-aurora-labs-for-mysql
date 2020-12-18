# 내결함성 테스트

이 실습에서는 Amazon Aurora에서 제공하는 고가용성 및 내결함성 기능을 테스트합니다. 문서에서 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html" target="_blank">고가용성</a> 기능 및 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html" target="_blank">연결 관리</a> 모범 사례에 대한 자세한 내용을 찾을 수 있습니다. 이러한 테스트는 장애 복구의 가장 중요한 측면을 다루기 위해 설계되었으며 모든 장애 조건에 대해 포괄적인 것은 아닙니다.

??? tip "Amazon Aurora의 고 가용성에 대해 자세히 알아보기"
    Amazon Aurora에서 고가용성 (HA)은 최소 2개의 DB 인스턴스, 하나의 가용 영역에 쓰기, 다른 가용 영역에 읽기 복제본이있는 클러스터를 배포하여 구현됩니다. 이 구성을 **다중 AZ** 라고 합니다. [사전](/modules/prerequisites/) 실습에서 CloudFormation을 사용하여 DB 클러스터를 프로비저닝 했거나, [새 Aurora 클러스터 생성](/modules/create) 실습에 따라 DB 클러스터를 수동으로 생성한 경우 **다중 AZ** Aurora DB 클러스터를 배포한 것입니다.

    장애 발생시 Amazon Aurora는 동일한 DB 인스턴스 내에서 데이터베이스 엔진을 다시 시작하거나 상황에 따라 리더 DB 인스턴스중 하나를 새로운 마서터로 승격하여 작업을 최대한 빨리 복원합니다. 따라서 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html#Aurora.Overview.Endpoints.Types" target="_blank">관련 DB 클러스터 엔드 포인트</a>를 사용하여 Amazon Aurora에 연결하는 것이 좋습니다. '쓰기' 인스턴스의 역할은 오류 발생시한 DB 인스턴스에서 다른 DB 인스턴스로 이동할 수 있으며 클러스터 엔드 포인트는 항상 이러한 변경 사항을 반영하도록 업데이트됩니다. .


    클라이언트 애플리케이션은 지정된 TTL (Time-to-Live) (일반적으로 5 초)을 초과하여 이러한 엔드 포인트에 대해 확인된 DNS 응답을 캐시하지 않아야하며 연결이 끊어진 경우 다시 연결을 시도해야하며 연결된 DB 인스턴스로 예정된 역할이 있는지 항상 확인해야합니다 (쓰기 또는 앍가).


이 실습에는 다음 작업이 포함됩니다.

1. 장애 조치 이벤트 알림 설정
2. 수동 DB 클러스터 장애 조치 테스트
3. 오류 주입 쿼리 테스트
4. 클러스터 인식으로 장애 조치 테스트
5. RDS Proxy를 사용하여 장애 조치 중단 최소화
6. 더 많은 테스트 제안
7. 리소스 정리

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/)
* [Session Manager 워크스테이션에 연결](/prereqs/connect/)
* [새 DB 클러스터 생성](/provisioned/create/) (클러스터를 수동으로 생성하려는 경우)


## 1. 장애 조치 이벤트 알림 설정
DB 클러스터에서 장애 조치 이벤트가 발생할때 알림을 받으려면 Amazon Simple Notification Service (SNS) 주제를 생성하고, SNS 주제에 이메일 주소를 구독하고, SNS 주제에 이벤트를 게시하는 RDS 이벤트 구독을 생성하고, 이벤트 소스로 DB 클러스터를 등록합니다. 

Session Manager 워크스테이션 명령줄에 아직 연결되어 있지 않은 경우 [다음](/prereqs/connect/) 지침에 따라 연결하십시오. 연결되면 다음을 실행하십시오.


```shell
aws sns create-topic \
--name auroralab-cluster-failovers
```

성공하면 명령이 **TopicArn** 를 응답하며, 다음 명령에서 이 값이 필요합니다.


<span class="image">![Create SNS Topic](1-sns-topic.png?raw=true)</span>

다음으로, ==[YourEmail]== 을 변경하여 아래 명령을 사용하여 SNS 주제에 이메일 주소를 구독하십시오.


```shell
aws sns subscribe \
--topic-arn $(aws sns list-topics --query 'Topics[?contains(TopicArn,`auroralab-cluster-failovers`)].TopicArn' --output text) \
--protocol email \
--notification-endpoint '[YourEmail]'
```

해당 주소로 확인 이메일을 받게됩니다. 이메일의 지침에 따라 구독을 확인하십시오.

<span class="image">![Create SNS Topic](1-subscription-verify.png?raw=true)</span>

확인이 완료되거나 확인 이메일이 도착하기를 기다리는 동안, RDS 이벤트 구독을 생성하고 아래 명령을 사용하여 DB 클러스터를 이벤트 소스로 등록합니다.


```shell
aws rds create-event-subscription \
--subscription-name auroralab-cluster-failovers \
--sns-topic-arn $(aws sns list-topics --query 'Topics[?contains(TopicArn,`auroralab-cluster-failovers`)].TopicArn' --output text) \
--source-type db-cluster \
--event-categories '["failover"]' \
--enabled

aws rds add-source-identifier-to-subscription \
--subscription-name auroralab-cluster-failovers \
--source-identifier auroralab-mysql-cluster
```

<span class="image">![RDS Event Subscription](1-rds-event-source.png?raw=true)</span>

이제 이벤트 알림이 구성되었습니다. 다음 섹션으로 진행하기 전에 이메일 주소를 확인했는지 확인하십시오.



## 2. 수동 DB 클러스터 장애 조치 테스트

이 테스트에서는 [간단한 장애 조치 모니터링 스크립트](/scripts/simple_failover.py)를 사용하여 데이터베이스 상태를 확인합니다.


??? tip "간단한 모니터링 스크립트에 대해 자세히 알아보기"
    이 스크립트는 쓰기 DB 인스턴스를 모니터링하도록 설계되었습니다. DB 클러스터의 **클러스터 엔드포인트**에 연결을 시도하고 다음 SQL 쿼리를 실행하여 DB 인스턴스의 상태를 확인합니다.


    ```
    SELECT @@innodb_read_only, @@aurora_server_id, @@aurora_version;
    ```

    변수 | 값 | 설명
    --- | --- | ---
    innodb_read_only | `0` 쓰기용, `1` 읽기용 | 이 전역 시스템 변수는 스토리지 엔진이 읽기전용 모드로 열렸는지 여부를 나타냅니다.
    aurora_server_id | `auroralab-[...]` | 생성시 특정 클러스터 멤버에 대해 구성된 DB 인스턴스 식별자의 값입니다.
    aurora_version | e.g. `1.19.5` | DB 클러스터에서 실행되는 Amazon Aurora MySQL 데이터베이스 엔진의 버전입니다. 이 버전 번호는 MySQL 버전과 다릅니다.

    In the event of a fault, the script will report the number of seconds it takes to reconnect to the intended endpoint and the writer role.
    오류가 발생한 경우 스크립트는 엔드포인트 및 쓰기 역할에 다시 연결하는데 걸리는 수 초가 걸릴 수 있습니다.


Session Manager 워크스테이션에 대한 추가 명령줄 세션을 열어야합니다. 하나에서 명령을 실행하고 다른 세션에서 결과를 볼 수 있습니다. 세션 관리자 명령줄 세션을 만드는 방법은 [Session Manager에 연결](/prereqs/connect/)을 참조 하십시오. 두 브라우저 창이 나란히있는 경우에도 더 효과적입니다.

두개의 명령줄 세션 중 하나에서 다음 명령을 사용하여 모니터링 스크립트를 시작합니다.

```shell
python3 simple_failover.py -e[clusterEndpoint] -u$DBUSER -p"$DBPASS"
```

`Ctrl+C`를 눌러 언제든지 모니터링 스크립트를 종료할 수 있습니다 .


!!! warning "클러스터 엔드포인트"
    이 테스트를 위해 다른 엔드포인트가 아닌 **클러스터 엔드포인트** 를 사용하는지 확인하십시오. 오류가 발생하면 스크립트를 시작하고 엔드포인트가 올바른지 확인하십시오.


<span class="image">![Initialize Sessions](2-initialize-sessions.png?raw=true)</span>

다른 명령줄 세션에서는 클러스터의 수동 장애 조치를 트리거합니다. 이 과정에서 Amazon Aurora는 읽기를 새 쓰기 DB 인스턴스로 승격하고 이전 쓰기를 읽기 역할로 변경합니다. 이 프로세스는 완료하는데 몇 초가 걸리며 모니터링 스크립트와 다른 데이터베이스 연결의 연결을 끊습니다. 클러스터의 모든 DB 인스턴스가 다시 시작됩니다.

모니터링 스크립트를 실행하지 않는 명령줄 세션에 다음 명령을 입력합니다.

```shell
aws rds failover-db-cluster \
--db-cluster-identifier auroralab-mysql-cluster
```

모니터 스크립트의 출력을 기다렸다가 살펴봅니다. Amazon Aurora가 장애 조치를 시작하는데 다소 시간이 걸릴 수 있습니다. 장애 조치가 발생하면 아래 예와 유사한 모니터링 출력이 표시됩니다.

<span class="image">![Trigger DNS Failover](2-dns-failover.png?raw=true)</span>

??? info "관찰"
    처음에 클러스터 DNS 엔드포인트는 클러스터 DB 인스턴스중 하나의 IP 주소로 확인됩니다.(위 예제의 `auroralab-mysql-node-01`) 모니터링 스크립트는 특정 DB 인스턴스에 연결하여 작성자인지 확인합니다.

    실제 장애 조치가 AWS 자동화에 의해 구현되면 쓰기와 읽기기 DB 엔진이 모두 재부팅되고 재구성되므로 모니터링 스크립트가 데이터베이스 엔진에 연결할 수 없게됩니다.

    몇 초 후에 모니터링 스크립트가 DB 엔진에 다시 연결할 수 있지만 DNS가 아직 완전히 업데이트되지 않았으므로 현재는 읽기인 이전 쓰기 DB 인스턴스에 계속 연결됩니다. 이것은 연결을 설정하거나 커넥션 풀에서 사용할 때 엔진의 역할을 확인하는 것의 중요성을 강조합니다. 모니터링 스크립트는 불일치를 올바르게 감지하고 올바른 엔드포인트에 다시 연결을 계속 시도합니다.

    몇 초가 더 지나면 모니터링 스크립트는 클러스터 엔드포인트가 새 쓰기 인스턴스를 가리키도록 자동으로 재구성되고, DNS TTL이 클라이언트를 만료 한 후 올바른 새 쓰기 DB 인스턴스 (위 예제의 `auroralab-mysql-node-02`)에 연결할 수 있습니다. 위의 예에서 클라이언트 측에서 관찰한 총 장애 조치 중단은 ~ 12 초였습니다.

    실제 하드웨어 오류가 발생하면 이전 작성기 인스턴스에 연결할 수 없습니다. 그러나 클라이언트는 시도 시간이 초과 될 때까지 연결을 시도할 수 있습니다. 클라이언트 MySQL 드라이버 구성의 `connect_timeout` 값이 너무 길면 복구가 더 오래 지연될 수 있습니다. 그러나 중단을 최소화하면서 작성기 DB 인스턴스의 컴퓨팅을 확장하는 것과 같이 수동 장애 조치를 시작하려는 다른 사용 사례가 있습니다.

    DNS 확인 및 캐싱의 특성으로 인해 클라이언트가 장애 조치 후 짧은 시간 동안 쓰기와 읽기 사이를 순환 할 수 있으므로 *플립 플롭* 효과를 경험할 수도 있습니다.


장애 조치 절차를 몇 번 반복하여 중요한 차이가 있는지 확인하십시오.

또한 시작한 각 장애 조치에 대해 두 개의 이벤트 알림 이메일을 받게됩니다. 하나는 장애 조치가 **시작** 되었음을 나타내고 다른 하나는 **완료** 되었음을 나타냅니다 .

<span class="image">![SNS Emails](2-notification-emails.png?raw=true)</span>

!!! 주의
    두 이벤트 알림의 알림 타임 스탬프 간의 차이는 모니터링 스크립트를 사용하여 관찰된 실제 중단보다 클 수 있습니다. 모니터링 스크립트는 클라이언트가 관찰한 실제 중단을 측정하는 반면 이벤트 알림은 DB 클러스터가 다시 작동하면 후속 서비스측 검증을 포함하여 종단간 장애 조치 프로세스를 반영합니다.


## 3.  오류 주입 쿼리 테스트

이 테스트에서는 DB 인스턴스에서 데이터베이스 엔진 서비스의 충돌을 시뮬레이션합니다. 이러한 유형의 충돌은 메모리 부족 상태 또는 기타 예기치 않은 상황으로 인해 실제 상황에서 발생할 수 있습니다.


??? tip "오류 주입 쿼리에 대해 자세히 알아보기""
    <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.FaultInjectionQueries.html" target="_blank">오류 주입 쿼리</a> 는 DB 클러스터 운영의 다양한 오류를 시뮬레이션하는 메커니즘을 제공합니다. 이러한 오류에 대한 클라이언트 응용 프로그램의 내구성을 테스트하는데 사용됩니다. 다음 용도로 사용할 수 있습니다.

    * DB 인스턴스에서 실행되는 중요한 서비스의 충돌을 시뮬레이션합니다. 일반적으로 읽기에 대한 장애 조치가 발생하지는 않지만 관련 서비스가 다시 시작됩니다.
    * 본질적으로 일시적이든 더 영구적이든 디스크 하위 시스템 성능 저하 또는 정체를 시뮬레이션합니다.
    * 읽기 전용 복제본 실패 시뮬레이션

모니터링 스크립트를 실행하지 않는 명령줄 세션에서 MySQL 클라이언트를 사용하여 클러스터 엔드포인트에 연결합니다.

```shell
mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab
```

이제 다음 오류 주입 명령을 실행하십시오.

```sql
ALTER SYSTEM CRASH INSTANCE;
```

모니터 스크립트 출력을 기다렸다가 확인하십시오. 트리거되면 아래 예와 유사한 모니터링 출력이 표시됩니다.

<span class="image">![Trigger Fault Injection](3-fault-injection.png?raw=true)</span>

??? info "관찰"
    충돌이 트리거되면 모니터링 스크립트가 데이터베이스 엔진에 연결할 수 없게됩니다.

    몇 초 후 모니터링 스크립트는 DB 엔진에 다시 연결할 수 있습니다.

    DB 인스턴스의 역할은 변경되지 않았으며 작성자는 여전히 동일한 DB 인스턴스입니다.(위의 예에서 `auroralab-mysql-node-02`)

    DNS 변경이 필요하지 않습니다. 결과적으로 복구가 훨씬 더 빨라집니다. 위의 예에서 클라이언트 측에서 관찰된 총 장애 조치 중단은 ~ 2 초였습니다.


다음을 입력하여 연결이 끊어진 경우에도 mysql 명령 콘솔을 종료해야 할 수 있습니다.

```sql
quit;
```


## 4. 클러스터 인식으로 장애 조치 테스트

간단한 DNS 기반 장애 조치는 대부분의 사용 사례에서 잘 작동하며 상대적으로 빠릅니다. 그러나 알고 있듯이 DNS 플립 플로핑 효과를 포함하여 DNS 업데이트 및 만료 지연으로 인해 연결이 여전히 몇 초 동안 중단됩니다. 따라서 장애 조치 시간이 더욱 향상될 수 있습니다. 이 테스트에서는 기본 [클러스터 인식 모니터링 스크립트](/scripts/aware_failover.py)를 사용하고 위에서 사용된 간단한 모니터링 스크립트와 나란히 비교합니다.


??? tip "클러스터 인식 모니터링 스크립트에 대해 자세히 알아보기"
    스크립트는 위의 간단한 모니터링 스크립트와 비슷하지만 몇 가지 중요한 차이점이 있습니다.

    * 처음에는 실패 할 때마다 다시 연결할 때 DB 클러스터 토폴로지를 쿼리하여 '새'쓰기 DB 인스턴스를 결정합니다. 토폴로지는 데이터베이스의 `information_schema.replica_host_status` 테이블에 기록되어 있으며 각 DB 인스턴스에서 액세스 할 수 있습니다.
    * 현재 DB 인스턴스가 쓰기가 아닌 경우 새 쓰기의 DB 인스턴스 엔드포인트에 직접 다시 연결됩니다.
    * 새로운 오류가 발생하면 클러스터 DNS 엔드포인트를 다시 사용하도록 대체됩니다.

두 개의 명령줄 세션이 열려 있고 활성화되어 있다고 가정하고 세 번째 명령줄 세션을 엽니다. Session Manager 명령줄 세션을 만드는 방법은 [Session Manager에 연결](/prereqs/connect/)을 참조 하십시오. 새 브라우저 창을 다른 창과 나란히 놓으면 더 효과적입니다.

새(세 번째) 명령줄 세션에서 다음 명령을 사용하여 클러스터 인식 모니터링 스크립트를 시작합니다.

```shell
python3 aware_failover.py -e[clusterEndpoint] -u$DBUSER -p"$DBPASS"
```

`Ctrl+C` 를 눌러 언제든지 모니터링 스크립트를 종료할 수 있습니다.

!!! 주의
    위의 2단계의 간단한 DNS 모니터링 스크립트와 달리 읽기 엔드포인트를 사용하여 클러스터 인식 모니터링 스크립트를 호출할 수도 있습니다. 그러나 클러스터 엔드 포인트는 여전히 권장됩니다.

모니터링 스크립트를 실행하지 않는 명령줄 세션에 다음 명령을 입력하여 장애 조치를 트리거합니다.

```shell
aws rds failover-db-cluster \
--db-cluster-identifier auroralab-mysql-cluster
```

모니터 스크립트 출력을 기다렸다가 관찰하십시오. Amazon Aurora가 장애 조치를 시작하는데 다소 시간이 걸릴 수 있습니다. 장애 조치가 발생하면 아래 예와 유사한 모니터링 출력이 표시됩니다.


<span class="image">![Trigger Aware Failover](4-aware-failover.png?raw=true)</span>

??? info "관찰""
    충돌이 트리거되면 모니터링 스크립트가 데이터베이스 엔진에 연결할 수 없게됩니다.

    몇 초후 모니터링 스크립트는 DB 엔진에 다시 연결할 수 있으며 DB 인스턴스의 역할이 변경되었음을 감지합니다. 클러스터의 토폴로지를 쿼리하고 새 쓰기 DB 인스턴스 식별자를 검색하고 DB 인스턴스 엔드포인트를 계산합니다.

    연결을 끊었다가 DB 인스턴스 엔드포인트를 사용하여 직접 새 DB 쓰기에 다시 연결합니다.

    위의 예에서 이 프로세스는 클러스터 DNS 엔드 포인트에만 의존할 때 9초에 비해 연결을 복원하는 데 4초가 걸렸습니다.

    초기 클러스터 DNS 엔드포인트는 여전히 권한이 있고 선호됩니다. 클러스터 인식 모니터링 스크립트는 작동하는 동안만 DB 인스턴스 엔드 포인트를 사용하고 장애가 발생하면 클러스터 엔드포인트로 되돌립니다. 이렇게하면 쓰기 DB 인스턴스의 전체 컴퓨팅 오류가 발생하더라도 가능한 빨리 연결이 복원됩니다.


장애 조치 절차를 몇 번 반복하여 중요한 차이가 있는지 확인하십시오.

이전과 마찬가지로 시작한 각 장애 조치에 대해 두 개의 이벤트 알림 이메일을 받게됩니다. 하나는 장애 조치가 **시작** 되었음을 나타내고 다른 하나는 **완료** 되었음을 나타냅니다 .


## 5. RDS Proxy를 사용하여 장애 조치 중단 최소화

이 테스트에서는 DB 클러스터용 <a href="https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy.html" target="_blank">Amazon RDS Proxy</a> 를 생성하고, [간단한 모니터링 스크립트](/scripts/simple_failover.py) 를 사용하여 연결하고, 수동 장애 조치를 호출하고, 결과를 데이터베이스에 직접 연결하는 이전 테스트와 비교합니다.

??? tip "Amazon RDS Proxy에 대해 자세히 알아보기"
    
    Amazon RDS Proxy는 Amazon Relational Database Service (RDS)를 위한 완전 관리형 고 
    가용성 데이터베이스 프록시로, 애플리케이션을보다 확장 가능하고 데이터베이스 장애에 대해보다 탄력적이고 안전하게 만들어줍니다. RDS Proxy는 애플리케이션 연결을 유지하면서 새 데이터베이스 인스턴스에 자동으로 연결하여 데이터베이스 가용성에 영향을 미치는 중단으로 인한 애플리케이션 중단을 최소화합니다. 장애 조치가 발생하면 요청을 다시 라우팅하기 위해 DNS 변경에 의존하는 대신 RDS Proxy가 요청을 새 데이터베이스 인스턴스로 직접 라우팅합니다.


<a href="https://console.aws.amazon.com/rds/home" target="_blank">Amazon RDS 서비스 콘솔</a> 을 엽니다.

!!! warning "리전 확인"
    특히 위의 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 올바른 지역에서 여전히 작업하고 있는지 확인하십시오.

왼쪽 탐색 메뉴에서 **Proxies** 로 이동 합니다. **프록시 생성**을 클릭합니다 .

<span class="image">![Create Proxy](5-create-proxy.png?raw=true)</span>

**프록시 구성**에서 프록시 식별자를 `auroralab-mysql-proxy`로 설정합니다. **대상 그룹 구성** 섹션 데이터베이스에서 `auroralab-mysql-cluster`를 선택합니다. 다른 모든 기본값은 그대로 둡니다.

<span class="image">![Configure Proxy](5-config-proxy-target.png?raw=true)</span>

**연결** 섹션의 **Secrets Manager 보안 암호** 에서 `secretCusterMasterUser`로 시작하는 이름을 가진 암호를 선택합니다. **IAM 역할**에서 **IAM 역할 생성**을 선택합니다. **추가 연결 구성**을 확장하고 기존 VPC 보안 그룹에서 `auroralab-database-sg`을 선택합니다.


<span class="image">![Configure Proxy Connectivity](5-config-connectivity.png?raw=true)</span>

**프록시 생성**을 클릭합니다.

<span class="image">![Agree Create](5-config-proxy-agree.png?raw=true)</span>

Creating a proxy may take several minutes, you may need to refresh your browser page to view up to date status information. Once the status is listed as **Available**, click on the proxy identifier to view details.
프록시를 만드는데 몇 분 정도 걸릴 수 있으며 최신 상태 정보를 보려면 브라우저 페이지를 새로 고쳐야 할 수 있습니다. 상태가 **Available** 로 표시되면 프록시 식별자를 클릭하여 세부 정보를 봅니다.



<span class="image">![Proxy Listing](5-proxy-listing.png?raw=true)</span>

아래 **프록시 엔드 포인트**를 적어둡니다. 나중에 실습에서 사용합니다.

<span class="image">![Proxy Details](5-proxy-details.png?raw=true)</span>

다음으로 두 개의 명령줄 세션이 열려 있고 활성화되어 있어야합니다.(이전 테스트에서 3개가 열려 있는 경우 오른쪽 상단 모서리에있는 **종료** 버튼을 클릭하여 하나를 닫을 수 있습니다). Session Manager 명령 줄 세션을 만드는 방법은 [Session Manager에 연결](/prereqs/connect/)을 참조 하십시오. 또한 두 개의 명령줄 브라우저 창을 나란히 배치하면 더 효과적입니다.

두 명령줄 세션중 하나에서 다음 명령을 사용하여 모니터링 스크립트를 시작합니다.

```shell
python3 simple_failover.py -e[proxy endpoint from above] -u$DBUSER -p"$DBPASS"
```

`Ctrl+C`를 눌러 언제든지 모니터링 스크립트를 종료할 수 있습니다.


!!! warning "프록시 엔드포인트"
    이전 실습에서 사용한 클러스터 엔드포인트가 아니라 이전 단계의 **프록시 엔드포인트**를 사용하고 있는지 확인하십시오. 오류가 발생하면 스크립트를 시작하고 엔드 포인트가 올바른지 확인하십시오.

<span class="image">![Monitor Started](5-monitor-started.png?raw=true)</span>

다른 명령줄 세션에서는 클러스터의 수동 장애 조치를 트리거합니다.

모니터링 스크립트를 실행하지 않는 명령줄 세션에 다음 명령을 입력합니다.

```shell
aws rds failover-db-cluster \
--db-cluster-identifier auroralab-mysql-cluster
```

모니터 스크립트 출력을 기다렸다가 확인하십시오. Amazon Aurora가 장애 조치를 시작하는데 다소 시간이 걸릴 수 있습니다. 장애 조치가 발생하면 아래 예와 유사한 모니터링 출력이 표시됩니다.


<span class="image">![Monitor Failover](5-monitor-failover.png?raw=true)</span>

??? info "관찰
    처음에 프록시는 DB 클러스터의 현재 작성자에게 트래픽을 전송합니다.(위의 예의 `auroralab-mysql-node-01`) 프록시는 모니터링 스크립트 쿼리를 특정 DB 인스턴스로 전달합니다.

    실제 장애 조치가 AWS 자동화에 의해 구현될 때 모니터링 스크립트는 MySQL 오류 1105와 함께 연결 해제를 경험합니다. 1초 후 다시 연결할 수 있습니다. 이번에만 프록시가 새 쓰기(위의 `auroralab-mysql-node-02`)에 쿼리를 전달합니다. 클라이언트는 쿼리 응답에 대해 ~ 2 초의 대기 시간을 경험했습니다.

    클라이언트(이 예에서는 모니터링 스크립트)가 발행한 요청의 타이밍이 중요합니다. 장애 조치가 시작될 때 쿼리가 진행 중이거나 장애 조치가 진행되는 동안 새 연결을 설정하려는 경우 프록시가 오류를 반환하므로 다시 시도 할 수 있습니다. 진행중인 쿼리가없는 기존 클라이언트 연결은 열린 상태로 유지되며, 장애 조치가 시작된후 수신된 모든 쿼리는 장애 조치가 완료될 때까지 프록시에서 대기열에 추가됩니다. 이러한 클라이언트는 장애 조치 중에 발행된 쿼리에 대한 응답 대기 시간이 증가합니다.

장애 조치 절차를 몇 번 반복하여 중요한 차이가 있는지 확인하십시오.

또한 시작한 각 장애 조치에 대해 두 개의 이벤트 알림 이메일을 받게됩니다. 하나는 장애 조치가 **시작** 되었음을 나타내고 다른 하나는 **완료** 되었음을 나타냅니다 .


## 6. 더 많은 테스트 제안

위의 테스트는 비교적 간단한 실패 조건을 나타냅니다. 실패 모드에 따라 고급 테스트 프로세스 또는 클러스터 인식 논리가 필요할 수 있습니다. 고급 테스트 및 오류 복구를 위해 다음 사항을 고려하십시오.

* 대규모 프로덕션로드가 장애 복구 시간에 어떤 영향을 미칩니까? 부하가 걸린 시스템으로 테스트 해보십시오.
* 복구 시간은 당시의 워크로드 조건에 따라 어떤 영향을 받습니까? 다른 작업 부하 조건에서 오류 복구 테스트를 고려하십시오. 예: DDL 작업 중 충돌.


## 7. 리소스 정리

이 실습을 실행하여 추가 AWS 리소스를 생성했습니다. 이 실습을 완료한 후 이러한 서비스 사용에 대해 원치 않는 요금이 발생하지 않도록 아래 명령을 실행하여 이러한 리소스를 제거하는 것이 좋습니다.

```shell
aws rds remove-source-identifier-from-subscription \
--subscription-name auroralab-cluster-failovers \
--source-identifier auroralab-mysql-cluster

aws rds delete-event-subscription \
--subscription-name auroralab-cluster-failovers

aws sns delete-topic \
--topic-arn $(aws sns list-topics --query 'Topics[?contains(TopicArn,`auroralab-cluster-failovers`)].TopicArn' --output text)
```
