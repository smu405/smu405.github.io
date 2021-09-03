
# Spark Cloud

Last updated 20200907MON1000 20180430MON1230

## 목적
Databricks Spark 사용할 수 있다.

## 목차

* C.1 Databricks
    * 1. 노트북: 노트북 생성, 열기, 사용
    * 2. 파일시스템: fs, dbfs dbutils 명령어 (put, ls, ...), 라이브러리 설치
    * 3. 파일: 메뉴사용해서 올리기, dbutils를 사용하여 읽기, 쓰기, 삭제
* C.2 Google Colab

## C.1 Databricks

Spark는 클라우드 플랫폼으로 제공되고 있어서, 자신의 컴퓨터에 직접 설치하지 않고 사용할 수 있다. Databricks, Google Dataproc, 마이크로소프트 Azure에서 제공하고 있다. 여기서는 데이터브릭스에서 온라인으로 제공하는 https://community.cloud.databricks.com 를 설명한다.
Databricks Community Edition은 **Amazon Web Services**에서 실행되지만, 사용자에게 비용이 부과되지는 않는다. 용량이 더 필요하거나, 실제 제품을 만드려면 community판에서 업그레이드해야 한다.

### C.1.1 노트북

#### Workspace
로그인하면 **Workspace**가 주어진다. Workspace는 자신의 작업공간으로 **루트 폴더**와 같다. Notebook을 저장하거나 라이브러리를 설치하거나 데이터를 저장할 수 있다. 또한 일반적으로 자신의 컴퓨터에서 할 수 있는 복사, 이동 등을 할 수 있다.

#### 클러스터 생성
Notebook을 사용하려면 클러스터를 가동해야 한다.
맨 좌측 메뉴에서 **Clusters**를 선택하고, 생성된 클러스터가 있는지 확인하자. 처음 사용하는 것이라면 당연히 없겠다.
**+ Create Cluster** 버튼을 누르고:
- Cluster Name에는 적당한 이름, 자신의 영문 이니셜을 입력해도 된다.
- Databricks Runtime Version은 원하는 버전을 골라서 선택하면 되지만, 현재 보여지는 것을 그냥 두어도 된다.

* 주의: 무료 사용 (Community Edition)은 2시간 아무 작업도 하지 않으면 클러스터가 종료된다 (your cluster will automatically terminate after an idle period of two hours.)

#### 노트북 생성
노트북을 사용하려면, 맨 좌측 메뉴에서 **Workspace** 아이콘을 클릭하면 우측으로 열리는 메뉴 **아래 화살표**를 누른다.
처음 노트북을 생성하려면, **Workspace** 아이콘을 클릭하면 열리는 메뉴판에서 자신의 로그인 이메일 옆의 **화살표 'v'**를 누르면 드롭다운 메뉴가 나타난다. Create > Notebook를 선택하면 생성할 수 있다.
- Name은 적당한 이름, 가급적 영어와 숫자로 구성하여 입력한다. 확장자 ipynb는 입력하지 않는다 (예: s1_rdd)
- Default Language는 Python으로 하자.
- Cluster는 방금 생성한 클러스터를 고른다.

#### 노트북 열기
생성할 때와 같이 **Workspace** 아이콘을 클릭하면 열리는 메뉴판에서 자신의 로그인 이메일, 이번에는 아래에 생성했던 노트북 파일을 클릭한다.
열리면 맨 위에 클러스터를 Detached되어 있다면, 생성해 놓았던 클러스터를 선택하면 된다.

#### 노트북 사용

##### Spark 사용
보통 Spark 인스턴스를 만드는 것과 같이 생성해서 사용할 수 있다.
또는 Databricks에서 미리 친절하게 생성해서 제공하는 아래와 같은 Spark 변수가 있으니, 그냥 사용해도 된다.

미리 생성된 변수들 | 클래스
-----|-----
spark | SparkSession, 모든 컨텍스트의 통합 세션
sc | SparkContext
sqlContext | SQLContext

##### shell 명령어
노트북 shell 명령어는 우리가 보통 Notebook에서 사용하는 것과 크게 다르지 않다.
```%```를 붙여서 사용하면 된다.

셀 명령어 | 설명
-----|-----
%md | markdown을 쓸 경우
%sh | shell
%fs | DBFS Databricks file system
%scala, %python, %r, and %sql | 다른 언어 지원

### 노트북 가져나가기

상단 메뉴에서 File > Export > IPython Notebook을 클릭하면, 현재 작성중인 파일이 로컬에 저장된다.


### C.1.2 File System

* 1) 내 계정의 로컬파일은 file:/로 시작한다. 보통 리눅스와 같아서, 사용자 계정 /home/ubuntu/를 제공한다.
* 2) 분산파일 (dbfs:/)이 있다. dbfs 약어는 Databricks File System의 줄임말로 자체적으로 제공하는 파일 시스템이다.

파일관련 명령어는 리눅스와 거의 동일하다. 도움말은 dbutils.fs.help() 명령으로 볼 수 있다.

#### C.1.2.1 cli
github 오픈소스 Databricks command-line interface (CLI) 설치해서 사용한다.
리눅스에서 사용하는 명령어와 비슷하다.

```
dbfs ls
dbfs cp ./apple.txt dbfs:/apple.txt
dbfs cp -r ./banana dbfs:/banana
```

#### C.1.2.2 %fs

셀명령어 ```%fs```를 사용해서 해당 셀에 다음과 같이 파일을 읽거나, 복사하거나 할 수 있다.


```python
%fs ls /

%fs ls /FileStore/tables
```

#### C.1.2.3 dbutils

DBFS Utils 사용 예: 디렉토리 생성, put, head, rm 등 명령어를 사용할 수 있다.
* dbfs:/는 dbfs경로를 접근할 경우 사용한다.
* file:/는 자신의 databricks 계정으로 할당이 된 로컬 파일을 접근할 경우 사용한다.
* %fs로 줄여쓸 수 있다.
* 도움말 dbutils.fs.help()

```
dbutils.fs.mkdirs("/foobar/")
```

dbutils.fs.ls('dbfs:/FileStore/tables')  # list b'dbfs:/FileStore/tables'

dbutils.fs.ls("/data/")

dbutils.fs.ls("/") 또는 'dbfs:' 넣어서 dbutils.fs.ls("dbfs:/")

dbutils.fs.ls("file:/home/ubuntu/databricks") # 'file:' lists local files (not mine!)

dbfs cp ./apple.txt dbfs:/apple.txt # Put local file ./apple.txt to dbfs:/apple.txt

dbfs cp /apple.txt ./apple.txt # Get dbfs:/apple.txt and save to local file ./apple.txt

dbfs cp -r ./banana /banana # Recursively put local dir ./banana to dbfs:/banana

#### C.1.3 라이브러리 설치

웬만한 라이브러리는 설치되어 있다.
다음이 설치되어 있는지 라이브러를 불러와 보자.

```
import matplotlib.pyplot as plt
import seaborn as sns
```

##### 메뉴에서 라이브러리 설치

Databriks Runtime version 5.1이상에서 library설치 기능이 보완되었다.
**Workspace** 아이콘을 누른 후, Create, Library를 클릭한다.
Python 라이브러리를 설치하려면 PyPI를 누르고, 라이브러리 명을 타이핑하고 Create를 누르면 설치가 된다.
PyPI에서 simplejson을 설치하고 테스트 해보자.

```
import simplejson
```

##### dbutils 명령어로 라이브러리 설치

또는 그냥 노트북 셀에서 dbutils 명령어로 설치할 수 있다.

* 1) 처음에 'scipy'이 설치되었는지 확인하자. 버전을 확인하면 제공하는 버전은 '1.4.1'

```
%python
import pkg_resources
pkg_resources.get_distribution('scipy').version

Out[1]: '1.4.1'
```

* 2) 위 '1.4.1'버전은 기본으로 제공되는 버전이고, 내가 사용하는 노트북에서 설치된 라이브러리를 확인해보자.
지금은 처음이니 아무 것도 없을 것이다.

```
dbutils.library.list()

Out[2]: []
```

* 3) 설치하려는 버전 'scipy' '1.5.0'을 설치하고, 재시작 restartPython()하면 1.5.0이 설치된 것을 확인할 수 있다.

```
dbutils.library.installPyPI('scipy','1.5.0')
dbutils.library.restartPython()
dbutils.library.list()

Out[3]: ['scipy==1.5.0']
```

* 4) 설치된 scipy의 버전이 맞는지 import scipy해서 확인해보자.

```
%python
import scipy
import pkg_resources
pkg_resources.get_distribution('scipy').version

Out[1]: '1.5.0'
```

##### 해당 노트북으로 제한된 라이브러리

pip 명령어를 사용한다.
```python
%pip install matplotlib
%pip uninstall -y matplotlib
```

### C.1.3 데이터

데이터베이스는 테이블로 구성되며, 테이블은 구조적 데이터로 구성된다.
테이블은 크게 나누면:
* 전역 테이블 Global tables은 모든 클러스터에서 사용할 수 있는 파일이고
* 지역 테이블 Local tables은 한 클러스터에서만 사용할 수 있는 파일을 말한다.
테이블은 파일을 올리는 메뉴를 이용하거나, 프로그램으로 파일을 올리면 만들어진다.

#### C.1.3.1 메뉴를 사용해서 파일 올리기

좌측 메뉴에서 **Data** 아이콘을 클릭하면 메뉴가 열린다. 우측 상단에 ```Add Data```를 하면 ```Create New Table```이 뜬다.

* 1) 아래 Upload File, S3, DBFS, Other Data Sources메뉴가 있다.
DBFS에 파일을 올리거나, 현재 올려진 파일 목록을 볼 수 있다.

* 2) 또는 /FileStore/tables/ 아래에 'Select' 버튼으로 선택하거나, 마우스 Drop으로 파일을 올릴 수 있다.
그러면 ```/FileStore/tables/<filename>-<random-number>.<file-type>``` 명칭으로 생성된다.

#### C.1.3.2 dbutils를 사용하여 읽고, 쓰거나, 삭제

1) fs.put() 또는 2) 보통 파일 쓰는 방식으로 쓴다.

```
jstxt="""Wikipedia
Apache Spark is an open source cluster computing framework.
아파치 스파크는 오픈 소스 클러스터 컴퓨팅 프레임워크이다.
Apache Spark Apache Spark Apache Spark Apache Spark
아파치 스파크 아파치 스파크 아파치 스파크 아파치 스파크
Originally developed at the University of California, Berkeley's AMPLab,
the Spark codebase was later donated to the Apache Software Foundation,
which has maintained it since.
Spark provides an interface for programming entire clusters with
implicit data parallelism and fault-tolerance."""

dbutils.fs.put("/data/ds_spark_wiki.txt", jstxt)
```

방금 쓰인 파일을 확인할 수 있다.
```
dbutils.fs.ls("/data/")
```

파일을 삭제할 때는 rm 명령어를 사용한다.

```
dbutils.fs.rm('/data/ds_spark_wiki.txt')
```

파일을 읽고 쓸 때는 로컬파일과 분별하기 위해 앞에 '/dbfs'를 붙여주어야 한다.
```
dbutils.fs.ls("/FileStore/tables/")
```

#### C.1.3.3 프로그램으로 파일 생성

spark 또는 SQL 명령어로 테이블을 생성할 수 있다.
아래 명령어는 DataFrame을 테이블로 생성하고 있다.
주의할 점은 파일이 생성되는 폴더가 /usr/hive/warehouse 디렉토리 안에 default 데이터베이스의 테이블로 생성된다.
그 디렉토리로 가서 찾아보자.
그 디렉토리는 'Create New Table' > DBFS > user > hive > warehouse를 클릭하면 거기서 찾을 수 있다.
```python
df=spark.read.csv('/FileStore/tables/tmp.txt')
df.write.saveAsTable("tmpDf")
dbutils.fs.ls('/user/hive/warehouse')
```

데이터브릭스는 클라우드 환경에서 실행됩니다.

```python
import os
_url = 'http://kdd.ics.uci.edu/databases/kddcup99/kddcup.data_10_percent.gz'
_fname = os.path.join(os.getcwd(),'/data','kddcup.data_10_percent.gz')
```

위 명령어를 실행하고, 해당 디렉토리 /data를 찾아보세요. /data아니고, /FileStore/tables라고 적어도 됩니다.

```
%fs ls /data
```

위 명령어를 실행하면 size가 2144903으로 파일이 이미 다운로드 되어있어요. 그렇다면 urlretrieve 명령을 또 실행할 필요가 없어요.
그 다음은 Rdd, DataFrame 생성할 때 그 경로명을 넣어주면 됩니다.

```python
_rdd = spark.sparkContext.textFile(_fname)
_rdd.count()
```

##### local file

자신이 만든 영역, 즉 로컬파일을 읽을 수 있다.
/dbfs/라고 한다. dbfs:/라고 하지 않는다.
아래 명령어는 파일을 찾을 수 없다고 ```FileNotFoundError``` 오류가 발생하고 있다. 현재 버전에서는 Python에서 파일을 읽을 때 경로에 문제가 있다.


```python
with open("/dbfs/data/ds_spark_wiki.txt", "r") as f_read:
  for line in f_read:
    print line
```


```python

```


```python

```

## C.2 Google Colab

클라우드에서 GPU

Colab에서 Apache Spark 2.3.2 with hadoop 2.7, Java 8를 설치해야 한다.

!apt-get install openjdk-8-jdk-headless -qq > /dev/null


!wget -q https://www-us.apache.org/dist/spark/spark-2.4.1/spark-2.4.1-bin-hadoop2.7.tgz

wget https://downloads.apache.org/spark/spark-3.0.1/spark-3.0.1-bin-hadoop2.7.tgz

!tar xf spark-2.4.1-bin-hadoop2.7.tgz

!pip install -q findspark

### jdk 설치

뒤 qq는 아마 조용히 아무런 출력없이 설치?


```python
!apt-get install openjdk-8-jdk-headless -qq > /dev/null
```

### Spark 설치


```python
!wget https://downloads.apache.org/spark/spark-3.0.3/spark-3.0.3-bin-hadoop2.7.tgz

!tar -xvf spark-3.0.3-bin-hadoop2.7.tgz
```

### findspark 설치


```python
!pip install findspark
```

### JAVA_HOME, SPARK_HOME 설정


```python
import os
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-8-openjdk-amd64"
os.environ["SPARK_HOME"] = "/content/spark-3.0.3-bin-hadoop2.7"
```


```python
import os
os.listdir('/content')
```


```python
import findspark
findspark.init()
```


```python
import pyspark
```


```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.master("local[*]").getOrCreate()

spark.version
```

### 파일올리기

https://towardsdatascience.com/3-ways-to-load-csv-files-into-colab-7c14fcbdcb92

1) Github에서 읽기

Github의 파일을 들어가서, Raw 버튼을 클릭하면 브라우저 주소창에 url이 나타난다.
이 url을 복사해서 붙여넣고 읽어오면 된다.
requests로 읽으면 txt, json 형식을 잘 읽어올 수 있다.
csv 파일은 pandas로 읽을 수 있다.

필드의 값이 해당 파일을 다운로드 받을 수 있는 uri입니다. 이 URI 구조를 보면,

https://raw.githubusercontent.com/:owner/:repo/:branch/:file_path


```python
import requests

url = "https://raw.githubusercontent.com/smu405/s/master/PITCHME.md"
req = requests.get(url)
req = req.text
print(req[:100])
```

    # Spark ML
    
    온라인 슬라이드 https://gitpitch.com/smu405/s
    
    ## M.1 학습내용
    
    ### M.1.1 목표
    
    * 사례별로 데이터를 분석하여 분류, 


2) local file

colab에서 제공하는 files.upload() 함수를 사용하면, 바로 파일을 선택하고 업로드할 수 있다.

```
from google.colab import files
files.upload()
```

3) Google Drive에서의 파일

PyDrive 라이브러리를 사용하여, Google Drive의 파일을 가져올 수 있다.
이 방법은 인증을 거쳐야 하며 위 두 방법에 비해 복잡한 절차가 필요하다.
```
!pip install -U -q PyDrive
from pydrive.auth import GoogleAuth
from pydrive.drive import GoogleDrive
from google.colab import auth
from oauth2client.client import GoogleCredentials

auth.authenticate_user()
gauth = GoogleAuth()
gauth.credentials = GoogleCredentials.get_application_default()
drive = GoogleDrive(gauth)
```


```python

```
