# GOLDILOCKS Monitoring Tool 설치 가이드

## 1. 개요
#### 1.1. 오픈소스 툴인 Grafana를 이용하여 GOLDILOCKS와 연동하는 방법을 설명한다.

#### 1.2. 환경 구축시 Port나 Configure를 동일하게 맞출 것을 권장한다.

#### 1.3. GOLDILOCKS와 Monitoring 설치 환경은 분리할 것을 권장한다.

#### 1.4. 상기 모니터링 툴은 GOLDILOCKS CLUSTER에서만 동작한다.

#### 1.5. 설치-연동 매뉴얼에서 고려되지 않은 환경 등에 대해서는 사용자가 직접 구성해야 한다.


## 2. Monitoring Tool 설명

### 2.1. 구조
![grafana](https://user-images.githubusercontent.com/35556392/43434262-e4845562-94b5-11e8-9bff-18efa373394a.png)

telegraf를 실행하고 GOLDILOCKS에 뷰를 만들어 주면 telegraf는 생성된 뷰에 맞는 데이터를 GOLDILOCKS에서 가져오게 되고, 그 후에 influxDB로 전달한다.
grafana를 실행시키고 data source를 지정하면, influxDB에 있는 데이터를 grafana가 읽어들인 후 dashboard에 기록하는 식으로 진행된다.

### 2.2. 설치 절차

* GOLDILOCKS 서버 구동
* TELEGRAF에서 사용할 뷰, TABLE 생성
* TELEGRAF 구동
* INFLUXDB 구동
* GRAFANA 구동
* GRAFANA 접속, DATASOURCE 설정
* DASHBOARD 생성


## 3. 사용 환경 구축

telegraf 패키지는 sunjesoft에서 제공된 것을 사용하여야 한다.
GOLDILOCKS는 동일한 장비에 설치되어 있다고 가정하며, 리스너 기본 포트인 22581을 기준으로 설명한다.

### 3.1. GOLDILOCKS-TELEGRAF용 VIEW 생성

제공된 telegraf 패키지 내부의 SQL파일을 실행하면 된다.
단, telegraf에서 설정된 유저와 동일하거나 설정된 유저가 view에 대한 권한을 가지고 있어야 한다.

~~~
$ tar -xvf telegraf_package.tar.gz
$ gsql(net) <username> <password> --import telegraf_package/sql/MonitoringView_CLUSTER.sql //모니터링을 위한 정보를 조회 가능한 뷰
$ gsql(net) <username> <password> --import telegraf_package/sql/InitData_CLUSTER.sql //telegraf가 가져올 데이터를 위한 쿼리를 저장하는 테이블(TELEGRAF_METRIC_SETTINGS)
~~~

telegraf에서 사용하는 테이블의 구조는 다음과 같다.

~~~
CREATE TABLE TELEGRAF_METRIC_SETTINGS
(
    SERIES_NAME  VARCHAR (100 ) PRIMARY KEY,
    QUERY        VARCHAR (4000 ) NOT NULL,
    TAGS         VARCHAR (1024 ) NULL,
    FIELDS       VARCHAR (1024 ) NULL,
    PIVOT_KEY    VARCHAR (100 ) NULL,
    PIVOT        INT NOT NULL DEFAULT 0
);
~~~

테이블의 상세한 내용은 다음과 같다.

* SERIES_NAME : influxdb 에 저장될 series 이름이다.
* QUERY : Goldilocks 에서 수행할 Query String 이다.
* TAGS : Query 를 수행한 결과 중 TAGS 로 사용할 Field 를 기술한다. 각각의 Tags 는 | 로 구분한다.
* FIELDS : Query 를 수행한 결과 중 FIELDS 로 사용할 Fields를 기술한다. 각각의 Fields 는 | 로 구분한다.
* PIVOT : PIVOT 기능을 사용할지 여부를 지정한다. 1이면 사용, 0이면 미사용이다.
* PIVOT_KEY : PIVOT 이 0 이 아닌 경우 쿼리를 수행한 결과집합의 PIVOT_KEY 컬럼의 내용을 Field 로 변환한다. Row 를 Column 으로 바꾸고 싶을때 사용한다. 역은 지원하지 않는다.


테이블의 row를 추가하여 수행할 쿼리를 입력하는 예제는 dashboard 생성 파트에서 설명한다.


### 3.2. telegraf 설정, 실행

#### telegraf.conf 변경

##### inputs.goldilocks

GOLDILOCKS 서버와 환경이 다를 시에는 library 파일을 telegraf가 있는 장비로 옮겨 줘야 한다.
기본 설정은 $GOLDILOCKS_HOME을 참조한다.

~~~
[[inputs.goldilocks]]
goldilocks_odbc_driver_path = "/home/telegraf/telegraf_home/lib/libgoldilockscs-ul64.so"
goldilocks_host = "127.0.0.1"
goldilocks_port = 22581
goldilocks_user = "test"
goldilocks_password = "test"
~~~

##### outputs.influxdb

influxdb는 동일한 장비에 설치한다고 가정한다.

~~~
[[outputs.influxdb]]
urls = ["http://127.0.0.1:8086"]
~~~



#### telegraf 실행
~~~
$ cd telegraf_package
$ ./run_telegraf.sh
~~~

#### run_telegraf.sh 상세
~~~
PWD=`pwd`

export LD_LIBRARY_PATH=$PWD/lib:$LD_LIBRARY_PATH
nohup ./bin/telegraf --config conf/telegraf.conf >> log/telegraf.log 2>&1 &
~~~

run_telegraf.sh는 telegraf를 background로 실행하고, log를 기록하는 역할을 하는 스크립트이다.
정상적으로 실행시엔 Process가 살아있게 되고, log를 조회시 아무런 내용이 갱신되지 않아 정상 실행 구분이 어려울 수 있다.
GOLDILOCKS가 클러스터 환경이 아닐 시엔 다음과 같은 에러를 반환하고 telegraf Process가 죽게 된다.

* 에러 예시
~~~
goroutine 100 [running]:
github.com/influxdata/telegraf/plugins/inputs/goldilocks_cluster.(*Goldilocks).runSQL(0xc4201c60e0, 0x20ae460, 0xc4201980a0, 0xc42024c000, 0x0, 0x0, 0x0)
        /home/ckh0618/workspace/go/src/github.com/influxdata/telegraf/plugins/inputs/goldilocks_cluster/goldilocks_cluster.go:158 +0xc61
github.com/influxdata/telegraf/plugins/inputs/goldilocks_cluster.(*Goldilocks).gatherServer(0xc4201c60e0, 0xc4207a4000, 0x6c, 0x20ae460, 0xc4201980a0, 0x0, 0x0)
        /home/ckh0618/workspace/go/src/github.com/influxdata/telegraf/plugins/inputs/goldilocks_cluster/goldilocks_cluster.go:351 +0x12b
github.com/influxdata/telegraf/plugins/inputs/goldilocks_cluster.(*Goldilocks).Gather.func1(0xc42047e0b0, 0x20ae460, 0xc4201980a0, 0xc4201c60e0, 0xc4207a4000, 0x6c)
        /home/ckh0618/workspace/go/src/github.com/influxdata/telegraf/plugins/inputs/goldilocks_cluster/goldilocks_cluster.go:82 +0x7d
created by github.com/influxdata/telegraf/plugins/inputs/goldilocks_cluster.(*Goldilocks).Gather
        /home/ckh0618/workspace/go/src/github.com/influxdata/telegraf/plugins/inputs/goldilocks_cluster/goldilocks_cluster.go:80 +0xf8
~~~




### 3.3. INFLUXDB 설정, 실행

influxDB를 설정 없이 실행하면 기본 포트 8086을 사용하게 된다.
설정된 포트를 변경하고 싶으면 influxdb.conf를 생성, 변경해서 실행 시 설정해주면 된다.

#### INFLUXDB 실행

~~~
$ wget https://dl.influxdata.com/influxdb/releases/influxdb-1.6.0_linux_amd64.tar.gz
$ tar -xvf influxdb-1.6.0_linux_amd64.tar.gz
$ cd influxdb-1.6.0-1/usr/bin
$ ./influxd config > influxdb.conf

$ ./influxd #default 실행
or
$ ./influxd -config influxdb.conf #포트 변경 등 config 변경시
~~~

#### INFLUXDB 설정 예시

~~~
[meta]
dir = "/home/telegraf/.influxdb/meta"
[data]
dir = "/home/telegraf/.influxdb/data"
wal-dir = "/home/telegraf/.influxdb/wal"
[http]
bind-address = ":8086" //grafana, telegraf에서 influxdb 접촉시 사용할 포트
~~~


### 3.4. GRAFANA 설정, 실행

grafana의 기본 port는 3000이다.

#### GRAFANA 실행
~~~
$ wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-5.2.2.linux-amd64.tar.gz
$ tar -xvf grafana-5.2.2.linux-amd64.tar.gz
$ ./bin/grafana-server
~~~

grafana-server 프로세스가 정상적으로 실행되었다면 브라우저를 통해 https://(addr):(port) 로 접속하면 된다.

#### GRAFANA 설정
~~~
[server]
http_addr = 192.168.0.97 // grafana 접속 주소
http_port = 3000 // grafana 접속 포트
~~~


## 4. GRAFANA - datasource, dashboard 설정

### 4.1. 초기 설정, dashboard 설명

지정된 host와 port를 통해 웹으로 접속하면 아래와 같은 로그인 화면이 나타난다.(ex. http://127.0.0.1:3000)
![login](https://user-images.githubusercontent.com/35556392/43447362-4e3316f8-94e6-11e8-9c02-772595227daa.png)

기본 admin 아이디는 admin/admin이며, 필요에 의해 변경할 수 있다.

![grafana - home](https://user-images.githubusercontent.com/35556392/43447370-5283cf0e-94e6-11e8-8d19-296100a471db.png)

로그인이 완료되면 Add data source를 선택한다.


![datasource](https://user-images.githubusercontent.com/35556392/43495685-5b2182e4-9574-11e8-9c38-a33eb64a2e81.png)

표시된 부분을 이미지와 같이 변경, 입력한 후 하단의 Save & Test를 선택한다.

![datasource working](https://user-images.githubusercontent.com/35556392/43447375-5616b7b2-94e6-11e8-928a-12a38534936a.png)

정상적으로 influxdb에 접촉할 수 있다면 상기 이미지와 같이 'Data source is working' 메시지가 나타난다.

![grafana - homedash](https://user-images.githubusercontent.com/35556392/43447409-6a8d1588-94e6-11e8-86e5-ef0dbb144915.png)

New dashboard를 선택한다.

![dashboard](https://user-images.githubusercontent.com/35556392/43447449-7eac4778-94e6-11e8-89a4-d2b263af344e.png)

dashboard 창으로 이동하면 새로운 패널을 생성할 수 있다. 우선 Graph를 선택한다.

![dash-edit](https://user-images.githubusercontent.com/35556392/43447585-c026dbfa-94e6-11e8-8d3b-1b696d1e879c.png)

'Panel Title'을 클릭하고 Edit를 선택한다.

![dash-editquery](https://user-images.githubusercontent.com/35556392/43447589-c1e55dae-94e6-11e8-8924-bb0cdcea265d.png)

생성된 패널이 어떤 쿼리를 수행해서 표시될 지 선택하고 편집할 수 있는 Edit창이 나타난다.
Edit창에서 선택할 수 있는 쿼리는 GOLDILOCKS에서 생성된 TELEGRAF_METRIC_SETTINGS 테이블에서 불러온다.


### 4.2. 원하는 Monitoring Query 추가 방법

제공된 패키지로 telegraf를 설치했을 시, 기본적으로 사용자 편의를 위해 제공되는 뷰와 쿼리 이외에도 사용자가 모니터링을 원하는 쿼리가 존재할 수 있다.
아래는 그 방법을 서술한다.

#### 4.2.1. 기존 Monitoring Query 조회

~~~
gSQL> set vertical on; //vertical on으로 조회시 가독성이 좋아진다.
gSQL> select * from telegraf_metric_settings;

SERIES_NAME # goldilocks_session_stat
      QUERY # SELECT * FROM MONITOR_SESSION_STAT
       TAGS # GROUP_NAME|MEMBER_NAME
     FIELDS # TOTAL_SESSION_COUNT|ACTIVE_SESSION_COUNT|TOTAL_STATEMENT_COUNT|LONG_RUNNING_STATEMENT_COUNT|TOTAL_TRANSACTION_COUNT|LONG_RUNNING_TRANSACTION_COUNT
  PIVOT_KEY # null
      PIVOT # 0

...

SERIES_NAME # goldilocks_shard_index_distibution
      QUERY # SELECT * FROM MONITOR_SHARD_IND_DISTRIBUTION
       TAGS # OWNER|TABLE_SCHEMA|TABLE_NAME|INDEX_NAME|GROUP_NAME
     FIELDS # ALLOC_BYTES
  PIVOT_KEY # null
      PIVOT # 0

12 rows selected.
~~~

![view](https://user-images.githubusercontent.com/35556392/43497570-cb22fd8a-957d-11e8-9990-4cfa4b57a1fd.png)

* FROM : QUERY를 통해 불러올 테이블을 선택할 수 있다.
* WHERE : TAGS COLUMN으로 설정되어 있는 COLUMN을 선택할 수 있다. |를 통해 구분된다.
* SELECT : 모니터링 패널에 표시될 값을 지정한다. FIELDS COLUMN으로 설정되어 있는 COLUMN을 선택할 수 있다. |를 통해 구분된다.

![dash_toggle](https://user-images.githubusercontent.com/35556392/43497654-36426830-957e-11e8-84b5-36307ac58eca.png)

Toggle Edit Mode를 선택하면 전체 조회 쿼리를 볼 수 있다.


#### 4.2.2. Monitoring Query 추가

dashboard에서 조회할 뷰를 추가하고 싶으면 사용자가 TELEGRAF_METRIC_SETTINGS 테이블에 ROW를 추가해 주어야 한다.

##### 사용 예시

간단한 구조의 테이블을 하나 만들고 구분이 가능하게 컬럼을 구성했다.

~~~
gSQL> desc t1;

COLUMN_NAME TYPE                  IS_NULLABLE
----------- --------------------- -----------
C1          CHARACTER VARYING(10) TRUE
C2          CHARACTER VARYING(10) TRUE
C3          NUMBER(10,0)          TRUE


select * from t1;

C1 C2 C3
-- -- --
A  a   1
A  a   2
B  b   3
B  b   4
A  a  10

gSQL> insert into telegraf_metric_settings values(
'SELECT * FROM T1',
'C1|C2',
'C3',
null,
0
)

1 row created.


gSQL> select * from telegraf_metric_settings;

             SERIES_NAME # goldilocks_session_stat
                   QUERY # SELECT * FROM MONITOR_SESSION_STAT
                    TAGS # GROUP_NAME|MEMBER_NAME
                  FIELDS # TOTAL_SESSION_COUNT|ACTIVE_SESSION_COUNT|TOTAL_STATEMENT_COUNT|LONG_RUNNING_STATEMENT_COUNT|TOTAL_TRANSACTION_COUNT|LONG_RUNNING_TRANSACTION_COUNT
               PIVOT_KEY # null
                   PIVOT # 0
...

             SERIES_NAME # goldilocks_t1
                   QUERY # SELECT * FROM T1
                    TAGS # C1|C2
                  FIELDS # C3
               PIVOT_KEY # null
                   PIVOT # 0

~~~

TELEGRAF_METRIC_SETTINGS 테이블에 원하는 쿼리를 추가 후 GRAFANA를 확인하면 항목이 갱신됨을 확인할 수 있다.

![dash_add](https://user-images.githubusercontent.com/35556392/43504888-efca7e52-959f-11e8-8bc0-eba8f513ea4d.png)
