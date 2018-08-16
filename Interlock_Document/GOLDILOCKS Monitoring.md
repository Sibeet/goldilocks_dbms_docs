# GOLDILOCKS Monitoring Tool 설치 가이드

## 1. 개요
#### 1.1. 본 문서는 오픈소스 어플리케이션을 이용해서 GOLDILOCKS를 모니터링 할 수 있는 방법을 설명하고자 만들어졌다.

#### 사용되는 어플리케이션은 Telegraf, influxDB, Grafana 세 가지의 어플리케이션이며, GOLDILOCKS plugin을 통해 telegraf와 GOLDILOCKS를 연동하는 시스템이다.




## 2. Monitoring Tool 설명

### 2.1. 구조
![flow_t](https://user-images.githubusercontent.com/35556392/44182018-faf26000-a13f-11e8-8fae-6b35dff514a4.png)

* TELEGRAF는 GOLDILOCKS Plugin을 통하여 설정된 뷰에 있는 DB 데이터를 수집한다. TELEGRAF는 GOLDILOCKS의 TELEGRAF_METRIC_SETTINGS에 등록된 view query를 통해 Monitoring을 원하는 데이터를 받아오게 된다.
* INFLUXDB는 TELEGRAF가 받아온 데이터를 저장하는 용도로 사용된다.
* GRAFANA는 INFLUXDB에 저장된 데이ㅓ를 수집해서 웹 서버를 만들고, dashboard를 이용해서 사용자에게 모니터링 시스템을 제공한다.
* TELEGRAF는 System resource 등의 정보를 수집하기 때문에 GOLDILOCKS(DBMS server)와 같은 시스템 내에 존재해야만 한다.
* INFLUXDB, GRAFANA 등의 모니터링 필요 어플리케이션은 GOLDILOCKS 서버와 분리하는 것을 권장한다.


### 2.2. 설치 절차

* GOLDILOCKS 서버 구동
* TELEGRAF에서 사용할 뷰, TABLE 생성
* TELEGRAF 구동
* INFLUXDB 구동
* GRAFANA 구동
* GRAFANA 접속, DATASOURCE 설정
* DASHBOARD 생성


## 3. 사용 환경 구축

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

#### 3.2.1. telegraf.conf 변경

##### Step1. cd /telegraf_package/conf

##### Step2. telegraf 속성 변경 (inputs.goldilocks)

GOLDILOCKS 서버와 환경이 다를 시에는 library 파일을 telegraf가 있는 장비로 옮겨 줘야 한다.
기본 설정은 $GOLDILOCKS_HOME을 참조하여 실행한다.

~~~
[[inputs.goldilocks]]
goldilocks_odbc_driver_path = "/home/telegraf/telegraf_home/lib/libgoldilockscs-ul64.so"
goldilocks_host = "127.0.0.1"
goldilocks_port = 22581
goldilocks_user = "test"
goldilocks_password = "test"
~~~

##### Step2. telegraf 속성 변경 (outputs.influxdb)

influxDB가 구동될 시스템의 host, port 정보를 설정ㅇ한다.
~~~
[[outputs.influxdb]]
#동일장비 : urls = ["http://127.0.0.1:8086"] / #다른장비 : urls = ["http://192.168.0.21:8086"]
~~~



#### 3.2.2 telegraf 실행

##### Step1. telegraf 실행

telegraf의 실행은 패키지 내에 만들어져 있는 run_telegraf.sh를 통해 실행할 수 있으며,
telegraf_package/lib에 있는 라이브러리 파일이 등록되어 있다면 telegraf 바이너리를 직접 실행해도 무방하다.

~~~
$ cd telegraf_package
$ ./run_telegraf.sh
~~~

##### run_telegraf.sh 상세
~~~
PWD=`pwd`

export LD_LIBRARY_PATH=$PWD/lib:$LD_LIBRARY_PATH
nohup ./bin/telegraf --config conf/telegraf.conf >> log/telegraf.log 2>&1 &
~~~

run_telegraf.sh는 사용자 편의를 위해 생성한 스크립트로, 설정한 telegraf.conf의 설정을 참조하며 세션이 끊어지더라도 Process가 죽지 않도록 nohup을 통해 백그라운드로 실행된다.


##### Step2. 상태 확인

* telegraf는 Cluster 환경에서만 연동이 가능하다. Standalone 환경에서는 다음과 같은 에러가 발생하며, Process가 종료될 수 있다.

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
* ps -ef | grep telegraf
telegraf Process의 구동 여부를 확인한다. telegraf 구동에 문제가 생기면 자동으로 프로세스가 죽게 된다.

* tail -f log/telegraf.log
telegraf의 가동 로그를 확인한다. 아무런 로그가 발생하지 않으면 정상적으로 구동되고 있다고 볼 수 있습니다.

* 기타 에러 로그
~~~
2018-07-03T01:20:28Z I! Database creation failed:
Post http://192.168.0.120:48000/query?q=CREATE+DATABASE+%22telegraf%22
~~~
influxDB 내부에서 telegraf와 관련된 database를 생성할 수 없어서 발생하는 에러이다.
ifnluxDB는 단순히 중계 역할을 하기 때문에 모니터링엔 문제가 생기지 않는다.

~~~
2018-06-21T00:45:00Z E! Error in plugin [inputs.goldilocks]:
SQLPrepare: {42000} [SUNJESOFT][ODBC][GOLDILOCKS]table or view does not exist
~~~
GOLDILOCKS 내부 인스턴스에서 반환하는 에러일 경우 표시되는 로그이다.

~~~
2018-08-02T09:18:40Z E! InfluxDB Output Error:
 Post http://192.168.0.120:48000/write?consistency=any&db=telegraf:
 dial tcp 192.168.0.120:48000: getsockopt: connection refused
2018-08-02T09:18:40Z E! Error writing to output [influxdb]: Could not write to any InfluxDB server in cluster
~~~
influxDB가 구동되어 있지 않아 찾을 수 없거나, host 혹은 port에 접근할 수 없을 때 발생하는 로그이다.


### 3.3. INFLUXDB 설정, 실행

influxDB를 설정 없이 실행하면 기본 포트 8086을 사용하게 된다.
설정된 포트를 변경하고 싶으면 influxdb.conf를 생성, 변경해서 실행 시 설정해주면 된다.

##### Step1. INFLUXDB 실행

~~~
$ wget https://dl.influxdata.com/influxdb/releases/influxdb-1.6.0_linux_amd64.tar.gz
$ tar -xvf influxdb-1.6.0_linux_amd64.tar.gz
$ cd influxdb-1.6.0-1/usr/bin
$ ./influxd config > influxdb.conf

$ ./influxd #default 실행
or
$ ./influxd -config influxdb.conf #포트 변경 등 config 변경시
~~~

##### Step2. INFLUXDB 설정

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

##### Step1. GRAFANA 실행
~~~
$ wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-5.2.2.linux-amd64.tar.gz
$ tar -xvf grafana-5.2.2.linux-amd64.tar.gz
$ ./bin/grafana-server
~~~

grafana-server 프로세스가 정상적으로 실행되었다면 브라우저를 통해 https://(addr):(port) 로 접속하면 된다.

##### Step2. GRAFANA 설정
~~~
[server]
http_addr = 192.168.0.97 // grafana 접속 주소
http_port = 3000 // grafana 접속 포트
~~~


## 4. GRAFANA - datasource, dashboard 설정

### 4.1. 초기 설정, dashboard 설명

지정된 host와 port를 통해 웹으로 접속하면 아래와 같은 로그인 화면이 나타난다.(ex. http://127.0.0.1:3000)

![login](https://user-images.githubusercontent.com/35556392/44182031-fc238d00-a13f-11e8-85a6-e48de7631455.png)

기본 admin 아이디는 admin/admin이며, 필요에 의해 변경할 수 있다.


![grafana - home](https://user-images.githubusercontent.com/35556392/44182030-fc238d00-a13f-11e8-9a21-8234eb7ede0a.png)

로그인이 완료되면 Add data source를 선택한다.

![datasource](https://user-images.githubusercontent.com/35556392/43495685-5b2182e4-9574-11e8-9c38-a33eb64a2e81.png)

표시된 부분을 이미지와 같이 변경, 입력한 후 하단의 Save & Test를 선택한다.

![datasource working](https://user-images.githubusercontent.com/35556392/43447375-5616b7b2-94e6-11e8-928a-12a38534936a.png)

정상적으로 influxdb에 접촉할 수 있다면 상기 이미지와 같이 'Data source is working' 메시지가 나타난다.

![grafana - datasourceconfig](https://user-images.githubusercontent.com/35556392/44182010-f9c13300-a13f-11e8-80e2-be94c1770505.png)

추가된 data source는 좌측 설정 탭의 Data Sources에서 확인, 변경 가능하다.

![datasource error](https://user-images.githubusercontent.com/35556392/44182012-f9c13300-a13f-11e8-92e1-d73b6158acfa.png)

influxDB를 찾을 수 없는 경우 다음과 같은 에러 메시지가 발생한다.

![grafana - homedash](https://user-images.githubusercontent.com/35556392/43447409-6a8d1588-94e6-11e8-86e5-ef0dbb144915.png)

홈으로 돌아가 New dashboard를 선택한다.

![dashboard](https://user-images.githubusercontent.com/35556392/43447449-7eac4778-94e6-11e8-89a4-d2b263af344e.png)

dashboard 창으로 이동하면 새로운 패널을 생성할 수 있다. 우선 Graph를 선택한다.

![dash-edit](https://user-images.githubusercontent.com/35556392/43447585-c026dbfa-94e6-11e8-8d3b-1b696d1e879c.png)

'Panel Title'을 클릭하고 Edit를 선택한다.

![dash-editquery](https://user-images.githubusercontent.com/35556392/43447589-c1e55dae-94e6-11e8-8924-bb0cdcea265d.png)

생성된 패널이 어떤 쿼리를 수행해서 표시될 지 선택하고 편집할 수 있는 Edit창이 나타난다.
Edit창에서 선택할 수 있는 쿼리는 GOLDILOCKS에서 생성된 TELEGRAF_METRIC_SETTINGS 테이블에서 불러온다.

![toggle](https://user-images.githubusercontent.com/35556392/44182011-f9c13300-a13f-11e8-9398-52fa5546b15e.png)

우측 중단의 X를 누르면 패널 선택 & 편집 화면으로 돌아갈 수 있다.

![new panel](https://user-images.githubusercontent.com/35556392/44182015-fa59c980-a13f-11e8-9904-4b3a2604cd9f.png)

패널 선택 화면에서 중앙 상단의 패널 추가 버튼을 누르면 새로운 패널을 만들 수 있다.
드래그 등을 이용하여 원하는 모양과 구성으로 대시보드를 편집할 수 있다.

![grafana - save](https://user-images.githubusercontent.com/35556392/44182007-f9289c80-a13f-11e8-9def-0bfeebb955a9.png)

패널 추가 버튼 오른쪽에 위치한 Save 버튼을 누르면 편집된 대시보드의 상태를 저장할 수 있다.

![grafana - goldilocks](https://user-images.githubusercontent.com/35556392/44182005-f9289c80-a13f-11e8-938b-92eaf3171f11.png)

패널 편집을 통해 대시보드를 구성한 예이다.


### 4.2. 원하는 Monitoring Query 추가 방법

#### 4.2.1. 기존 Monitoring Query 조회

우선 GOLDILOCKS에서 TELEGRAF_METRIC_SETTINGS에 등록된 쿼리를 확인한다.

~~~
gSQL> select * from telegraf_metric_settings;

SERIES_NAME # goldilocks_session_stat
      QUERY # SELECT * FROM MONITOR_SESSION_STAT
       TAGS # GROUP_NAME|MEMBER_NAME
     FIELDS #
TOTAL_SESSION_COUNT|ACTIVE_SESSION_COUNT|TOTAL_STATEMENT_COUNT|LONG_RUNNIN
_STATEMENT_COUNT|TOTAL_TRANSACTION_COUNT|LONG_RUNNING_TRANSACTION_COUNT
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

* Data Source : 값을 불러올 Datasource 이름
* FROM : QUERY를 통해 불러올 테이블
* WHERE : TAGS COLUMN으로 설정되어 있는 COLUMN. |를 통해 구분된다.
* SELECT : 모니터링 패널에 표시될 값. FIELDS COLUMN으로 설정되어 있는 COLUMN을 선택할 수 있다. |를 통해 구분된다.



![dash_toggle](https://user-images.githubusercontent.com/35556392/43497654-36426830-957e-11e8-84b5-36307ac58eca.png)

Toggle Edit Mode를 선택하면 전체 조회 쿼리를 볼 수 있다.


#### 4.2.2. Monitoring Query 추가

dashboard에서 조회할 뷰를 추가하고 싶으면 사용자가 TELEGRAF_METRIC_SETTINGS 테이블에 ROW를 추가해 주어야 한다.

##### 사용 예시

##### Step1. 간단한 구조의 테이블을 하나 만들고 구분이 가능하게 컬럼을 구성했다.

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


~~~
Step2. INSERT 구문을 통해 TELEGRAF_METRIC_SETTINGS에 쿼리를 추가해 줍니다.

~~~
gSQL> insert into telegraf_metric_settings values(
'SELECT * FROM T1',
'C1|C2',
'C3',
null,
0
)

1 row created.



~~~

Step3. 등록된 쿼리를 조회 후 확인한다.

~~~
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

![dash_add](https://user-images.githubusercontent.com/35556392/43504888-efca7e52-959f-11e8-8bc0-eba8f513ea4d.png)

Step4. 데이터 갱신을 기다린 후 쿼리를 선택하면 등록되어 있는 모습을 볼 수 있다.

### 4.2. Grafana Panel 추가 설치

Grafana에 기본적으로 등록된 패널 외에도 추가적인 패널을 원하는 경우, 플러그인 형식으로 추가가 가능하다.

![config-plugin](https://user-images.githubusercontent.com/35556392/44182003-f8900600-a13f-11e8-8402-1341f85522c3.png)

Configuration - Plugins를 선택한다.

![find plugin](https://user-images.githubusercontent.com/35556392/44182002-f7f76f80-a13f-11e8-85a0-37fae4985acd.png)

Plugins 선택창에서 Find more plugins, on Grafana.com'을 선택한다.

접속을 위해 외부망이 연결되어 있어야 한다.

![select panel](https://user-images.githubusercontent.com/35556392/44182001-f7f76f80-a13f-11e8-805e-2e4cd10ea58b.png)

플러그인을 선택하는 창에서 패널을 선택하면 패널별로 조회할 수 있다.

![install plugin](https://user-images.githubusercontent.com/35556392/44182034-fcbc2380-a13f-11e8-935d-c16e9b4a4cc1.png)

원하는 패널을 선택 후 installation 탭을 선택하면 설치법에 관해 볼 수 있다.
grafana 서버에서 cli를 통해 설치를 해도 되지만, zip 파일을 다운받아 직접 grafana/data/plugins에 압축을 풀어줘도 설치가 가능하다.
설치 후에는 Grafana 서버를 리부트해줘야 한다.

![installed panel](https://user-images.githubusercontent.com/35556392/44182033-fcbc2380-a13f-11e8-85c8-84435b1ea447.png)

서버 리부트 후 패널 추가를 선택하면 추가된 패널을 확인할 수 있다.
