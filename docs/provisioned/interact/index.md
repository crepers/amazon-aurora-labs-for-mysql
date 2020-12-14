# DB 연결, 데이터 로드 및 오토 스케일

이 실습에서는 방금 생성한 DB 클러스터에 연결하고 처음으로 클러스터를 사용하는 과정을 안내합니다. 마지막으로 부하 발생 스크립트를 사용하여 Aurora 읽기 전용 복제본 Auto Scaling이 실제로 어떻게 작동하는지 테스트합니다.

이 실습에는 다음 작업이 포함됩니다.

1. DB 클러스터에 연결
2. S3에서 초기 데이터 세트로드
3. 읽기 전용 워크로드 실행

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/)
* [Session Manager 워크스테이션에 연결](/prereqs/connect/)
* [새 DB 클러스터 생성](/provisioned/create/) (클러스터를 수동으로 생성하려는 경우)


## 1. DB 클러스터에 연결

클라이언트 도구를 사용하여 다른 MySQL기반 데이터베이스와 마찬가지로 Aurora 데이터베이스에 연결합니다. 이 실습에서는 `mysql` 명령줄을 사용하여 연결합니다.

이전 실습의 Session Manager 워크스테이션 명령줄(Command line)에 아직 연결하지 않은 경우 [다음](/prereqs/connect/)과 같이 연결하십시오. 연결되면 아래 명령을 실행하여 ==[clusterEndpoint]== 에 DB 클러스터 엔드포인트의 값으로 변경합니다.

!!! tip "클러스터 엔드포인트는 어디에서 찾을 수 있습니까?""
	이전 실습을 완료하고 Aurora DB 클러스터를 수동으로 생성 한 경우 해당 실습의 2단계에서 설명한대로 RDS 콘솔의 DB 클러스터 세부 정보 페이지에서 클러스터 엔드포인트의 값을 찾을 수 있습니다.

	공식 워크숍에 참여하고 있으며 이벤트 엔진을 사용하여 랩 환경이 프로비저닝된 경우 클러스터 엔드포인트의 값은 이벤트 엔진의 팀 대시 보드에서 찾을 수 있습니다.

	그렇지 않으면 시작하기 전제 조건 모듈에 표시된대로 CloudFormation 스택 출력에서 클러스터 엔드포인트를 검색 할 수 있습니다.

```shell
mysql -h[clusterEndpoint] -u$DBUSER -p"$DBPASS" mylab
```
??? tip "이 모든 파라미터는 무엇을 의미합니까?""
	적절한 CloudFormation 템플릿을 사용하여 DB 클러스터가 자동으로 생성되도록 선택한 경우 DB 클러스터의 데이터베이스 자격 증명이 자동으로 설정됩니다. 또한 이름이 mylab으로 지정된 스키마도 생성했습니다. 자격 증명은 AWS SecretsManager 암호에 저장되었습니다.

	다음 명령을 사용하여 저장된 자격 증명을 볼 수 있습니다.

    ```shell
    aws secretsmanager get-secret-value --secret-id [secretArn] | jq -r '.SecretString'
    ```

데이터베이스에 연결되면 아래 코드를 사용하여 나중에 실습에서 사용할 저장 프로시저를 생성하여 DB 클러스터에 부하를 생성합니다. 	다음 SQL 쿼리를 실행합니다.

```sql
DELIMITER $$
DROP PROCEDURE IF EXISTS minute_rollup$$
CREATE PROCEDURE minute_rollup(input_number INT)
BEGIN
 DECLARE counter int;
 DECLARE out_number float;
 set counter=0;
 WHILE counter <= input_number DO
 SET out_number=SQRT(rand());
 SET counter = counter + 1;
END WHILE;
END$$
DELIMITER ;
```


## 2. S3에서 초기 데이터 세트 로드


DB 클러스터에 연결되면 다음 SQL 쿼리를 실행하여 초기 테이블을 생성합니다.


```sql
DROP TABLE IF EXISTS `sbtest1`;
CREATE TABLE `sbtest1` (
 `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
 `k` int(10) unsigned NOT NULL DEFAULT '0',
 `c` char(120) NOT NULL DEFAULT '',
 `pad` char(60) NOT NULL DEFAULT '',
PRIMARY KEY (`id`),
KEY `k_1` (`k`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

다음으로 Amazon S3 버킷에서 데이터를 가져와서 초기 데이터 세트를 로드합니다.


```sql
LOAD DATA FROM S3 MANIFEST
's3-us-east-1://awsauroralabsmy-us-east-1/samples/sbdata/sample.manifest'
REPLACE
INTO TABLE sbtest1
CHARACTER SET 'latin1'
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\r\n';
```

데이터로드는 몇 분 정도 걸릴 수 있으며 완료되면 성공적인 메시지를 받게됩니다. 완료되면 MySQL 명령 줄을 종료합니다.


```sql
quit;
```


## 3. 읽기 전용 워크로드 실행

데이터로드가 성공적으로 완료되면 읽기 전용 워크로드를 실행하여 클러스터에 대한로드를 생성할 수 있습니다. 또한 DB 클러스터 토폴로지에 미치는 영향을 관찰합니다. 이 단계에서는 클러스터의 **리더 엔드포인트**를 사용 합니다. 클러스터를 수동으로 생성한 경우 해당 실습에 표시된대로 엔드 포인트 값을 찾을 수 있습니다. DB 클러스터가 자동으로 생성된 경우 CloudFormation 스택 출력에서 ​​값을 찾을 수 있습니다.

Session Manager 워크 스테이션 명령 줄에서 ==[readerEndpoint]== 를 리더 엔드포인트로 변경하고 부하 발생 스크립트를 실행합니다.

```shell
python3 reader_loadtest.py -e[readerEndpoint] -u$DBUSER -p"$DBPASS" -dmylab
```

이제 다른 브라우저 탭에서 <a href="https://console.aws.amazon.com/rds/home#databases:" target="_blank">Amazon RDS 서비스 콘솔</a>을 엽니다.

!!! warning "리전 확인"
    특히 위의 링크를 따라 서비스 콘솔을 여는 경우 올바른 리전에서 여전히 작업하고 있는지 확인하십시오.



리더 노드가 현재 부하를 받고 있다는 점에 유의하십시오. 메트릭이 들어오는 부하를 완전히 반영하는데는 1분 이상 걸릴 수 있습니다.

<span class="image">![Reader Load](3-read-load.png?raw=true)</span>

몇 분 후 인스턴스 목록으로 돌아가서 새 리더가 클러스터에 프로비저닝되고 있음을 확인합니다.

<span class="image">![Application Auto Scaling Creating Reader](3-aas-create-reader.png?raw=true)</span>


새 복제본을 사용할 수 있게 되면 부하가 분산되고 안정화됩니다(안정화되는데 몇 분 정도 걸릴 수 있음).

<span class="image">![Application Auto Scaling Creating Reader](3-read-load-balanced.png?raw=true)</span>

이제 Session Manager 명령줄로 다시 전환하고 CTRL+C입력하여 부하 발생기를 종료할 수 있습니다. 잠시 후 추가된 리더가 자동으로 제거됩니다.
