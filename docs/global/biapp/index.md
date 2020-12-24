# Aurora 글로벌 데이터베이스에 애플리케이션 연결

Amazon Aurora는 MySQL 및 PostgreSQL 호환 데이터베이스 엔진을 모두 제공합니다. 즉, MySQL 및 PostgreSQL과 함께 작동하는 기존 애플리케이션은 Amazon Aurora와 드롭 인(drop-in) 호환성이 있습니다. 이 실습에서는 쿼리 지연 시간을 줄이기 위해 두 리전 각각에서 작동하는 BI(business intelligence) 애플리케이션을 구성하고 Aurora 글로벌 데이터베이스의 각 로컬 DB 클러스터 읽기 엔드포인트에 연결합니다.


이 실습에서는 <a href="https://superset.incubator.apache.org/" target="_blank">Apache Superset</a> 을 애플리케이션으로 사용합니다. Superset은 시각적이고 직관적이며 상호 작용하도록 설계된 오픈 소스, 비즈니스 인텔리전스 및 데이터 탐색 플랫폼입니다.

이 실습에는 다음 작업이 포함됩니다.

1. 필요한 정보 수집
2. 기본 지역에서 애플리케이션 구성
3. 보조 지역에서 애플리케이션 구성
4. 애플리케이션을 사용하여 데이터에 액세스

이 실습에는 다음 전제 조건이 필요합니다.

* [시작](/prereqs/environment/) (**Deploy Global DB** 옵션 선택)
* [Aurora 글로벌 데이터베이스 배포](/global/deploy/)


## 1. 필요한 정보 수집¶

Apache Superset이 실습 환경에 이미 설치되어 있지만 Superset 웹 애플리케이션의 URL과 사용자 이름 및 비밀번호를 검색해야 합니다. 기본 및 보조 지역 모두에 대해 이러한 세부 정보를 검색해야 합니다. 마찬가지로 기본 및 보조 DB 클러스터에 대한 DB 클러스터 엔드포인트와 데이터베이스 액세스 자격 증명을 검색해야 합니다.

이 표는 아래의 세부 단계와 함께이 정보를 찾을 수있는 위치에 대한 개요를 제공합니다.

파라미터 | 기본 리전 위치 | 보조 리전 위치
----- | ----- | -----
Superset URL | CloudFormation 스택 출력 또는 Event Engine Team Dashboard | CloudFormation 스택 출력
Superset 계정정보 | CloudFormation 스택 출력 또는 Event Engine Team Dashboard | *기본 지역과 동일*
Aurora **클러스터** 엔드포인트 | CloudFormation 스택 출력 또는 Event Engine Team Dashboard | *보조 지역에서는 사용할 수 없습니다.*
Aurora **읽기** 엔드포인트 | CloudFormation 스택 출력 또는 Event Engine Team Dashboard | RDS 서비스 콘솔
Aurora DB 자격증명 | AWS Secrets Manager secret | *기본 지역과 동일*

**기본 리전** 에서 <a href="https://console.aws.amazon.com/cloudformation/home#/stacks" target="_blank">CloudFormation</a> 서비스 콘솔을 엽니다. 이름이 `auroralab`이거나 `mod-`로 시작하는 스택을 클릭합니다.

!!! warning "리전 확인"
    특히 위 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 **기본 리전** 에서 여전히 작업하고 있는지 확인합니다.


<span class="image">![CFN List of Stacks](cfn-stacks-list.png?raw=true)</span>

Change to the **Outputs** tab, and find the values for the parameters, and make a note of them:

**출력** 탭으로 변경하고 파라미터에 대한 값을 찾아서 기록해 둡니다.

* supersetURL
* supersetUsername
* supersetPassword
* clusterEndpoint
* readerEndpoint

<span class="image">![CFN Stack Outputs](cfn-stack-outputs.png?raw=true)</span>

!!! 주의
    이러한 값이 없으면 올바른 리전을 선택하지 않았거나 글로벌 데이터베이스 기능이 활성화된 상태에서 실습 환경이 초기화되지 않았을 수 있습니다. 조직된 이벤트(예 : 워크샵)에 참여하는 경우 실습 진행자에게 도움을 요청하십시오.


다음으로 **기본 리전** 에서 <a href="https://console.aws.amazon.com/secretsmanager/home#/listSecrets" target="_blank">AWS Secrets Manager 서비스 콘솔</a>을 엽니다. `secretClusterMasterUser-`로 시작하는 보안 암호를 클릭합니다.


!!! warning "리전 확인"
    특히 위 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 **기본 리전** 에서 여전히 작업하고 있는지 확인합니다.


<span class="image">![Secrets Manager List of Secrets](sm-secrets-list.png?raw=true)</span>

**보안 암호 값** 섹션까지 아래로 스크롤하고 **보안 암호 값 검색**을 클릭합니다. **보안 암호 키/값** 및 **보안 암호 값**을 기록해 둡니다.


<span class="image">![Secrets Manager Secret Detail](sm-secret-details.png?raw=true)</span>

이제 기본 리전에 필요한 모든 매개 변수를 수집했습니다. 다음으로 **보조 리전**에 필요한 파라미터를 수집합니다.

** 보조 리전** 에서 <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks" target="_blank">Amazon CloudFormation 서비스 콘솔</a>을 엽니다. 스택 이름이 `auroralab`이거나 mod-로 시작하는 스택을 클릭합니다.

!!! warning "리전 확인"
    특히 위 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 **보조 리전** 에서 여전히 작업하고 있는지 확인합니다.

<span class="image">![CFN List of Stacks](cfn-stacks-list.png?raw=true)</span>

**출력** 탭으로 변경하고 파라미터에 대한 값을 찾아 기록해 둡니다.

* supersetURL
* supersetUsername
* supersetPassword

<span class="image">![CFN Stack Outputs Secondary](cfn-stack-outputs-sec.png?raw=true)</span>

!!! 주의
    Apache Superset 애플리케이션의 사용자 이름과 비밀번호는 기본 및 보조 지역 모두에서 동일해야 하지만 애플리케이션 URL 엔드포인트는 다릅니다.


**보조 리전**에서 <a href="https://console.aws.amazon.com/rds/home?region=us-east-1#database:id=auroralab-mysql-secondary;is-cluster=true" target="_blank">Amazon RDS 서비스 콘솔</a> 의 **보조 DB 클러스터** MySQL의 DB 클러스터 세부 정보 페이지를 엽니다.


!!! warning "리전 확인"
    특히 위 링크를 따라 오른쪽 화면에서 서비스 콘솔을 여는 경우 **보조 리전** 에서 여전히 작업하고 있는지 확인합니다.

아직 선택하지 않은경우 **연결 및 보안** 탭을 클릭하고 **읽기 엔드포인트** 값을 기록해 둡니다. `사용가능` 상태인지 확인하십시오.


<span class="image">![RDS Secondary Endpoints](rds-secondary-endpoints.png?raw=true)</span>

!!! 주의
    쓰기(클러스터) 엔드포인트도 표시되지만 `Creating` 또는 `Inactive` 상태로 표시됩니다. 이는 정상적인 현상입니다. 클러스터 엔드포인트는 보조 리전이 독립 실행형(stand-alone) DB 클러스터로 승격된 경우에만 활성화됩니다.


이제 **기본** 및 **보조 지역** 모두에서 필요한 모든 정보를 수집했으며 애플리케이션 구성을 계속할 수 있습니다.


## 2. 기본 리전에서 애플리케이션 구성

새 브라우저 탭 또는 창을 엽니다. Apache Superset은 **기본 리전** 의 EC2 인스턴스에서 실행되는 웹 기반 애플리케이션입니다. ==[supersetURL]== **기본 리전** 의 값을 주소 표시줄에 입력합니다. URL 형식은 다음과 같습니다.

```text
http://ec2-XXX-XXX-XXX-XXX.<xx-region-x>.compute.amazonaws.com/
```

Apache Superset의 로그인 페이지가 표시되어야 합니다. **Username** 과 **Password**에 ==[supersetUsername]== 과 ==[supersetPassword]== 값을 입력하십시오.

<span class="image">![Superset Login](superset-login.png?raw=true)</span>

다음으로 Aurora 글로벌 데이터베이스 클러스터에 연결하기 위해 Apache Superset에 대한 새 데이터 원본을 생성합니다. Apache Superset 메뉴(상단 표시 줄)에서 **Sources** 위로 마우스를 이동한 다음 **Databases** 를 클릭합니다 .


<span class="image">![Superset Source Databases](superset-source-db.png)</span>

오른쪽 상단에서 녹색 **+** 아이콘을 클릭하여 새 데이터베이스 소스를 추가하십시오.

<span class="image">![Superset Source Databases](superset-list-sources.png)</span>

다음 양식에 아래 값을 입력하여 데이터 소스를 추가한 후 **Save** 을 클릭 하십시오 .


필드 | 값 | 설명
----- | ----- | -----
Database | `aurora-mysql-writer` | 이것은 Superset에 있는 Aurora 데이터베이스의 이름입니다.
SQLAlchemy URI | `mysql://[username]:[password]@[cluster_endpoint]/mysql` | ==[username]== and ==[password]== 를 이전에 검색한 Aurora DB 자격 증명으로 변경합니다. 이전에 검색한 Aurora DB 클러스터 엔드포인트(기본 리전에서)를 ==[cluster_endpoint]== 로 변경합니다. **Test Connection**를 클릭하여 설정이 올바른지 확인하십시오.
Expose in SQL Lab | &#9745; | 이 옵션이 **선택**되어 있는지 확인하십시오.
Allow CREATE TABLE AS | &#9745; | 이 옵션이 **선택**되어 있는지 확인하십시오.
Allow DML | &#9745; | 이 옵션이 **선택**되어 있는지 확인하십시오.

<span class="image">![Superset Writer DB Connection Settings](superset-dbconn-writer.png)</span>


## 2. 보조 리전에서 애플리케이션 구성

보조 리전 설정도 매우 유사합니다. 새 브라우저 탭 또는 창을 엽니다. Apache Superset은 **보조 리전** 의 EC2 인스턴스에서도 실행되는 웹 기반 애플리케이션입니다. **보조 리전**의 ==[supersetURL]== 값을 주소 표시줄에 입력합니다. URL 형식은 다음과 같습니다.

```text
http://ec2-XXX-XXX-XXX-XXX.<xx-region-x>.compute.amazonaws.com/
```
Apache Superset의 로그인 페이지가 표시되어야 합니다. **Username** 과 **Password**에 ==[supersetUsername]== 과 ==[supersetPassword]== 값을 입력하십시오.

<span class="image">![Superset Login](superset-login.png?raw=true)</span>

다음으로 Aurora 글로벌 데이터베이스 클러스터에 연결하기 위해 Apache Superset에 대한 새 데이터 원본을 생성합니다. Apache Superset 메뉴(상단 표시 줄)에서 **Sources** 위로 마우스를 이동한 다음 **Databases** 를 클릭합니다 .

<span class="image">![Superset Source Databases](superset-source-db.png)</span>

오른쪽 상단에서 녹색 **+** 아이콘을 클릭하여 새 데이터베이스 소스를 추가하십시오.

<span class="image">![Superset Source Databases](superset-list-sources.png)</span>

다음 양식에 아래 값을 입력하여 데이터 소스를 추가한 후 **Save** 을 클릭 하십시오 .

필드 | 값 | 설명
----- | ----- | -----
Database | `aurora-mysql-secondary` | 이것은 Superset에 있는 Aurora 데이터베이스의 이름입니다.
SQLAlchemy URI | `mysql://[username]:[password]@[reader_endpoint]/mysql` | ==[username]== and ==[password]== 를 이전에 검색한 Aurora DB 자격 증명으로 변경합니다. 이전에 **보조 리전**에서 검색한 Aurora DB **읽기 엔드포인트**를 ==[reader_endpoint]== 로 변경합니다. **Test Connection**를 클릭하여 설정이 올바른지 확인하십시오.
Expose in SQL Lab | &#9745; | 이 옵션이 **선택**되어 있는지 확인하십시오.
Allow CREATE TABLE AS | &#9744; | 이 옵션이 **선택 해제** 되어 있는지 확인하십시오.
Allow DML | &#9744; | 이 옵션이 **선택 해제** 되어 있는지 확인하십시오.

<span class="image">![Superset Secondary DB Connection Settings](superset-dbconn-secondary.png)</span>

## 4. 애플리케이션을 사용하여 데이터에 액세스

**기본 리전** 에서 Apache Superset에 대한 브라우저 창이나 탭이 더 이상 열려있지 않으면 위의 **2. 기본 지역에서 애플리케이션 구성 에서** 설명한 것과 동일한 단계를 사용하여 다시 로그인 합니다.

Apache Superset 메뉴에서 **SQL Lab** 위에 마우스를 놓은다음 **SQL Editor** 를 클릭합니다 .

<span class="image">![Superset SQL Lab](superset-sql-lab.png)</span>

Superset 의 웹 기반 IDE 왼쪽 메뉴에서, `mysql aurora-mysql-writer` **Database** 를 선택한 다음 **Schema** 에서 `mylab` 를 선택합니다.

다음 SQL 쿼리를 복사하여 편집기에 붙여넣은 다음 **Run Query** 를 클릭합니다 .


```sql
DROP TABLE IF EXISTS gdbtest1;
DROP PROCEDURE IF EXISTS InsertRand;

CREATE TABLE gdbtest1 (
  pk INT NOT NULL AUTO_INCREMENT,
  gen_number INT NOT NULL,
  PRIMARY KEY (pk)
  );

CREATE PROCEDURE InsertRand(IN NumRows INT, IN MinVal INT, IN MaxVal INT)
  BEGIN
     DECLARE i INT;
     SET i = 1;
     START TRANSACTION;
     WHILE i <= NumRows DO
           INSERT INTO gdbtest1 (gen_number) VALUES (MinVal + CEIL(RAND() * (MaxVal - MinVal)));
           SET i = i + 1;
     END WHILE;
     COMMIT;
  END;

CALL InsertRand(1000000, 1357, 9753);

SELECT count(pk), sum(gen_number), md5(avg(gen_number)) FROM gdbtest1;
```

<span class="image">![Superset SQL Editor On Writer](superset-sqledit-writer.png?raw=true)</span>


완료하는데 약 30초가 소요될 수 있습니다. 쿼리의 결과를 받기 전에 `Offline` 으로 보여도 괜찮습니다. 이 SQL은 새 테이블을 만들고 백만 개의 레코드를 글로벌 데이터베이스에 임의로 삽입합니다. 결과를 메모장에 기록하거나 브라우저 창을 열어 둡니다.

다음으로, **보조 지역** 에서 Apache Superset 브라우저 창이나 탭이 더 이상 열려있지 않으면 위에 설명된 **3. 보조 리전에서 애플리케이션 구성** 과 동일한 단계를 사용하여 다시 로그인합니다. 

마찬가지로 Apache Superset 메뉴에서 **SQL Lab** 위에 마우스를 놓은다음 **SQL Editor** 를 클릭합니다 .

Superset 의 웹 기반 IDE 왼쪽 메뉴에서 `aurora-mysql-secondary` **Database** 를 선택한 다음 **Schema** 에서 `mylab` 을 선택합니다.

다음 SQL 쿼리를 복사하여 붙여넣은 다음 **Run Query** 를 클릭합니다.


```sql
SELECT count(pk), sum(gen_number), md5(avg(gen_number)) FROM gdbtest1;
```  
    
<span class="image">![Superset SQL Editor On Writer](superset-sqledit-secondary.png?raw=true)</span>

결과에 유의하십시오. 필드는 기본 인스턴스의 이전 결과와 정확히 일치해야합니다. 여기에는 레코드 수, 무작위로 생성된 값의 합계, 생성된 값의 평균에 대한 md5 해시가 포함됩니다.

이는 Aurora 글로벌 데이터베이스의 다양한 지역 구성 요소와 상호 작용할 수있는 방법과 데이터가 기본 DB 클러스터에서 보조 DB 클러스터로 복제되는 방법을 보여줍니다.
