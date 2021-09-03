
# Spark DataFrame

* Last updated 20201004SUN1400 20191029_20170606_20170221_20161125

## S.1 학습내용

### S.1.1 목표

* Spark DataFrame을 생성하고, API를 사용할 수 있다.
* Spark SQL을 사용하여 데이터를 조회할 수 있다.
* Spark 데이터를 MongoDB에 쓰고, 읽을 수 있다.

### S.1.2 목차
* S.2 Jupyter Notebook에서 SparkSession 생성
* S.3 DataFrame: 특징, Schema, DataFrame의 API
* S.4 DataFrame 생성
* S.4.1 schema에서 생성하기
* S.4.2 RDD에서 생성하기
* S.4.3 Pandas
* S.4.4 csv 파일 읽기
* S.4.5 tsv 파일 읽기
* S.4.6 JSON 파일 읽기
* 문제: 월드컵 데이터 JSON
* S.4.7 Parquet 파일 읽기, 쓰기
* S.5 DataFrame API 사용해 보기
    * 빈 DataFrame 생성
    * range
    * 컬럼 추가 withColumn, 컬럼 삭제 Drop
    * 사용자정의 함수 udf
    * 컬럼명 변경 withColumnRenamed
    * 그래프
    * 컬럼 조회 select
    * filter
    * regexp_replace 컬럼의 내용 변경
    * groupBy
    * F 함수
    * 행 추가
    * partition
    * 통계 요약 describe
    * 결측값
* 문제: 년별 분기별 대여건수
* S.6 Spark SQL
* 문제 S-1: 네트워크에 불법적으로 침입하는 사용자의 분석
* 문제 S-2: Twitter JSON 데이터 읽기
* 문제 S-3: 뉴욕에서 출생한 신생아 분석
* 문제 S-4: 우버 택시의 운행기록 분석
* 문제 S-5: JDBC를 사용해서 데이터 읽기
* S.7 MongoDB Spark connector
* S.7.1 설정
* S.7.2 uri 
* S.7.3 MongoDB Python API 
* S.7.4 연습으로 쓰기, 읽기
* S.8 spark-submit
* S.8.1 간단한 작업
* S.8.2 MongoDB

## S.2 Jupyter Notebook에서 SparkSession 생성

### Spark를 설치한 경우

Spark를 설치했다면, Spark가 설치된 디렉토리를 SPARK_HOME으로 설정하고, 실행에 필요한 'py4j-0.10.1-src.zip', 'pyspark.zip' 라이브러리를 추가해야 한다.


```python
import os
import sys
os.environ["SPARK_HOME"]=os.path.join(os.environ['HOME'],'Downloads','spark-2.0.0-bin-hadoop2.7')
os.environ["PYLIB"]=os.path.join(os.environ["SPARK_HOME"],'python','lib')
sys.path.insert(0,os.path.join(os.environ["PYLIB"],'py4j-0.10.1-src.zip'))
sys.path.insert(0,os.path.join(os.environ["PYLIB"],'pyspark.zip'))
```

그러나 pyspark만을 설치한 경우에는, 위와 같이 별도로 경로와 라이브러리를 추가하지 않아도 된다.
다만 Python 2, 3 여러 버전이 설치된 경우에는 아래와 같이 그 경로를 설정해 주어야 한다.


```python
import os
os.environ["PYSPARK_PYTHON"]="/usr/bin/python3"
os.environ["PYSPARK_DRIVER_PYTHON"]="/usr/bin/python3"
```


```python
import pyspark
myConf=pyspark.SparkConf()
spark = pyspark.sql.SparkSession\
    .builder\
    .master("local")\
    .appName("myApp")\
    .config(conf=myConf)\
    .getOrCreate()
```


```python
import pyspark
myConf=pyspark.SparkConf()
spark = pyspark.sql.SparkSession\
    .builder\
    .master("local")\
    .appName("myApp")\
    .enableHiveSupport() \
    .config(conf=myConf)\
    .getOrCreate()
```


```python
print (spark.version)
```

    3.0.0


## S.3 DataFrame

### 특징

DataFrame은 **행, 열로 구조화**된 데이터구조이다.
관계형데이터베이스 RDB의 테이블이나 엑셀 쉬트sheet와 비슷하다.
또는 Pandas 또는 R을 사용해 보았다면 거기서 제공되는 DataFrame과 유사하다.
Apache Spark 1.0에서는 **SchemaRDD**라는 명칭으로 시험적으로 제공되었다. 이름에서 보듯이 RDD에 스키마를 얹어서 만든 개념이다.
그러나 Spark의 DataFrame은 **대용량 데이터를 처리하기 위해 만들어진 프레임워크로 분산**해서 사용할 수 있게 고안되었다.

앞서 사용했던 **RDD가 schema를 정하지 않는** 것과 달리, **DataFrame은 모델 schema**를 설정해서 사용을 한다. '열'에 대해 명칭 및 **데이터 타잎**을 가지고 있고, 이를 지켜서 데이터를 저장하게 된다. 

### Schema

**Row**는 DataFrame의 행으로, 데이터 요소항목을 묶어서 구성한다. **Python list나 dict**를 사용하여 Row를 구성할 수 있다.
**Column**은 DataFrame의 **열**이고, 다음과 같은 **데이터 타잎**을 가진다.

```python
Spark | Python
- NullType
- StringType | string
- BinaryType | bytearray

- BooleanType | bool
- DateType | datetime.date
- TimestampType | datetime.datetime

- DecimalType
- ByteType | int
- ShortType | int
- IntegerType | int
- LongType | long
- FloatType | float
- DoubleType | float

- ArrayType | list, tuple, array
- MapType | dict
- StructType | list or tuple
```

### DataFrame의 API

RDD와 마찬가지로 DataFrame을 구성하여 **머신러닝**의 입력데이터로 사용할 수 있다.
현재 버전 2.0부터 RDD에 대한 지원은 줄여나가고, **버전 3.0 이후에는 DataFrame API를 공식적으로 지원**한다고 발표한 바 있다. RDD보다 우선적으로 사용하는 것이 좋겠다. 아래는 DataFrame에서 제공하는 일부 API이다.

기능 | 설명 | 예제
-------|-------|-------
```json``` | json 파일에서 읽기 | ```spark.read.json("employee.json")```
```show``` | DataFrame에 로딩된 데이터 읽기 | ```df.show()```
```schema``` | 데이터 schema 보기 | ```df.printSchema()```
```select``` | 열을 선택 | ```df.select("name")```
```filter``` | 조건으로 선택하여 읽기 | ```df.filter(df["age"] > 23).show()```
```groupBy``` | 그룹으로 나누기 | ```df.groupBy("age").count().show()```
```dropna``` | na를 삭제 | ```df.dropna()``` ```df.na.drop()```
```fillna``` | na를 값으로 채우기 | ```fillna()```
```count``` | 행 세기 | ```df.count()```
```drop``` | 삭제 | ```df.drop("name")```


## S.4 DataFrame 생성

DataFrame은 관계형데이터베이스 RDB 테이블과 같이, schema를 정해서 생성한다.
**schema를 정해주지 않으면, Spark가 자동으로 유추**하게 된다.
Python 리스트, RDD, Pandas DataFrame, Hive, csv, JSON, RDB, XML, Parquet, Apache Cassandra 등 다양한 채널에서 읽어서 DataFrame을 만들 수 있다.

* **```spark.createDataFrame()```** 함수는 Python List, RDD, Pandas DataFrame 등에서 읽어서 DataFrame을 만들 수 있다.
* 또는 **```spark.read.text, spark.read.json, spark.read.parquet, spark.read.load```** 함수로 옵션을 설정해서 외부 파일 등에서 읽을 수 있다.


### S.4.1 schema 생성하기

DataFrame은 데이터 모델 schema를 정의하고, 각 컬럼의 명칭과 데이터타입을 정해야 한다.

#### 자동으로 인식하는 schema

우선 단순하게 Python 자료구조를 사용해서 생성해 본다.
아래 열이 3개인 데이터를 **```createDataFrame()```** 함수를 사용하여 넣어 보자.


```python
myList=[('1','kim, js', 170),
        ('1','lee, sm', 175),
        ('2','lim, yg',180),
        ('2','lee', 170)]
```


```python
myDf=spark.createDataFrame(myList)
```

이 경우 Spark가 **자동으로 schema를 설정**한다.
```printSchema()``` 함수로 schema를 출력해 보기로 하자.
* **컬럼**명은 **일련번호**를 가지고 생성된다. schema를 정하지 않았으므로, 열은 '_1', '_2'와 같이 명명된다.
* **데이터 타잎**도 유추해서 생성한다. 올바르게 되지 않을 경우도 있다는 점에 유의한다. 아래에서 보듯이 컬럼 1 학년은 ```String```, 컬럼 2 이름은 ```String```, 컬럼 3 키는 ```long```으로 인식하고 있다.
* nullable은 결측값이 허용되는지를 말하는 것이다. true이면 허용된다는 의미이다.


```python
myDf.columns
```




    ['_1', '_2', '_3']




```python
myDf.printSchema()
```

    root
     |-- _1: string (nullable = true)
     |-- _2: string (nullable = true)
     |-- _3: long (nullable = true)
    


한 줄을 출력해 보자. 출력은 Row() 타입으로 앞서 정의한 schema와 일치하는 값을 포함하여 출력한다.
즉 첫째, 둘째는 문자열로 세째는 long 타입이다.


```python
print (myDf.take(1))
```

    [Row(_1='1', _2='kim, js', _3=170)]


#### 컬럼명 설정

앞서 컬럼 Column을 정의하지 않고 DataFrame을 생성하였는데, 이번에는 **컬럼을 정해서** 생성하자.
**```createDataFrame()```** 함수에 **인자로 컬럼명을 리스트 ```['year','name','height']```로** 정해준다.


```python
cols = ['year','name','height']
_myDf = spark.createDataFrame(myList, cols)
```


```python
_myDf.columns
```




    ['year', 'name', 'height']



그리고 한 줄 출력해 보면, 컬럼명이 변경되어 있다.


```python
print (_myDf.take(1))
```

    [Row(year='1', name='kim, js', height=170)]


데이터 100개를 생성해 보자.
우선 데이터의 원천이 되는 names, items를 정의하자.
여기서 하나씩 선택하여 데이터를 생성하게 된다.


```python
names = ["kim","lee","lee","lim"]
items = ["espresso","latte","americano","affocato","long black","macciato"]
```

이 names, items 배열에서 **modulus**를 활용하여 하나씩 선택하여 데이터를 생성한다.
**names**는 4개이므로 4로 나눈 나머지와 **items**는 6개이므로 6으로 나눈 나머지를 하나씩 선택하고 있다.
그리고 컬럼명을 ```["name","coffee"]```으로 정한다.


```python
coffeeDf = spark.createDataFrame([(names[i%4], items[i%6]) for i in range(100)],\
                           ["name","coffee"])
```

자동으로 생성된 schema가 데이터타입을 잘 집어내고 있다.
name, coffee 모두 string이다.


```python
coffeeDf.printSchema()
```

    root
     |-- name: string (nullable = true)
     |-- coffee: string (nullable = true)
    



```python
coffeeDf.show(10)
```

    +----+----------+
    |name|    coffee|
    +----+----------+
    | kim|  espresso|
    | lee|     latte|
    | lee| americano|
    | lim|  affocato|
    | kim|long black|
    | lee|  macciato|
    | lee|  espresso|
    | lim|     latte|
    | kim| americano|
    | lee|  affocato|
    +----+----------+
    only showing top 10 rows
    


#### Row 객체를 사용해서 생성

* Row 생성
**Row**를 사용해 보자.
Row는 **이름(Column)이 붙여진 행**으로 **관계형데이터베이스 레코드 Record**와 같다. 
속성 명은 'year', 'name', 'height'로 명명한다.


```python
from pyspark.sql import Row
Person = Row('year','name', 'height')
row1=Person('1','kim, js',170)
```

* Row 속성 읽기

속성명을 읽을 때에는 **```row.key```** 또는 Python dict형식으로 **```row[key]```**와 같이 속성을 읽을 수 있다.


```python
print ("row1: ", row1.year, row1.name, row1.height)
```

    row1:  1 kim, js 170


* Row를 Dictionary로 저장

Row는 속성명과 값을 가지고 있기 때문에 Dictionary로 쉽게 변환할 수 있다.


```python
row1.asDict()
```




    {'year': '1', 'name': 'kim, js', 'height': 170}



Dictionary에서 키 또는 값을 읽을 수 있다.


```python
row1.asDict().keys()
```




    dict_keys(['year', 'name', 'height'])




```python
row1.asDict().values()
```




    dict_values(['1', 'kim, js', 170])



* Row에서 DataFrame 생성

위에서 설정한 Row를 사용하여 DataFrame을 만들어 보자.
**Python list에 Row를 넣어** 구성한다.
첫번째는 앞서 만든 row1 객체를 넣을 수 있다.


```python
myRows = [row1,
          Person('1','lee, sm', 175),
          Person('2','lim, yg',180),
          Person('2','lee',170)]
```


```python
myDf=spark.createDataFrame(myRows)
```

printSchema()를 해보면, **데이터 타잎**은 string, long으로 Spark에서 **자동** 인식되었다는 것을 알 수 있다.


```python
print (myDf.printSchema())
myDf.show()
```

    root
     |-- year: string (nullable = true)
     |-- name: string (nullable = true)
     |-- height: long (nullable = true)
    
    None
    +----+-------+------+
    |year|   name|height|
    +----+-------+------+
    |   1|kim, js|   170|
    |   1|lee, sm|   175|
    |   2|lim, yg|   180|
    |   2|    lee|   170|
    +----+-------+------+
    


#### schema를 정의하고 생성

모델 schema를 정하고, **데이터 타잎**을 정의해 DataFrame을 생성해 본다.
**```StructType```**으로 구조체를 선언하고, 컬럼에 대해 **```StructField```**를 설정한다.
* **컬럼**의 명칭
* 앞서 소개했던 **데이터 타잎**
* 마지막은 **NULL**이 허용되는지 여부

```python
StructType([
    StructField(컬럼명, StringType(), True),
    ...
])
```


```python
from pyspark.sql.types import StructType, StructField
from pyspark.sql.types import StringType, IntegerType
mySchema=StructType([
    StructField("year", StringType(), True),
    StructField("name", StringType(), True),
    StructField("height", IntegerType(), True)
])
```

앞서 **myRows**를 데이터로, **mySchema**에서는 컬럼 명과 데이터타잎을 정의하여 ```createDataFrame()```함수의 인자로 넘겨주고 있다.


```python
myDf=spark.createDataFrame(myRows, mySchema)
```

앞서 설정한 데이터타입으로 설정되어있다.


```python
myDf.printSchema()
```

    root
     |-- year: string (nullable = true)
     |-- name: string (nullable = true)
     |-- height: integer (nullable = true)
    



```python
myDf.take(1)
```




    [Row(year='1', name='kim, js', height=170)]



### S.4.2 RDD에서 생성하기

RDD는 schema가 정해지지 않은 비구조적 데이터이다.
이와 같이 **schema를 정의하지 않으면, Spark는 schema를 유추**하게 된다.


#### schema 자동 인식

RDD로부터 DataFrame을 생성할 수 있다. 이 경우 schema를 설정하지 않으면 자동으로 인식된다.


```python
myList=[('1','kim, js',170), ('1','lee, sm', 175), ('2','lim, yg',180), ('2','lee',170)]
```


```python
myRdd = spark.sparkContext.parallelize(myList)
```

```toDF()```로 변환하거나 직접 ```createDataFrame()``` 함수를 사용하여 DataFrame을 생성할 수 있다.


```python
rddDf=myRdd.toDF()
```


```python
rddDf.printSchema()
```

    root
     |-- _1: string (nullable = true)
     |-- _2: string (nullable = true)
     |-- _3: long (nullable = true)
    


또는 RDD에서 DataFrame을 생성할 수 있다.


```python
rddDf=spark.createDataFrame(myRdd)
```


```python
rddDf.printSchema()
```

    root
     |-- _1: string (nullable = true)
     |-- _2: string (nullable = true)
     |-- _3: long (nullable = true)
    


#### Row를 사용

학년year는 앞에서는 **string**으로 인식되었다. 이번 예제에서는 **형변환**을 해 본다.
RDD의 ```map()``` 함수를 사용하여 각 속성을 읽고 ```int()``` 함수로 형변환을 한다.
각 속성에 명칭, year, name, height를 설정한다.


```python
from pyspark.sql import Row
_myRdd=myRdd.map(lambda x:Row(year=int(x[0]), name=x[1], height=int(x[2])))
```


```python
_myDf=spark.createDataFrame(_myRdd)
```


```python
_myDf.printSchema()
```

    root
     |-- year: long (nullable = true)
     |-- name: string (nullable = true)
     |-- height: long (nullable = true)
    



```python
_myDf.take(1)
```




    [Row(year=1, name='kim, js', height=170)]



```Row()```를 사용하여 RDD를 생성할 수도 있다.


```python
from pyspark.sql import Row

r1=Row(name="js1", age=10)
r2=Row(name="js2", age=20)
_myRdd=spark.sparkContext.parallelize([r1,r2])
```


```python
_myRdd.collect()
```




    [Row(name='js1', age=10), Row(name='js2', age=20)]



####  schema를 정의하고 생성

앞서 보았듯이, schema를 정의하고 RDD에서 DataFrame을 생성할 수 있다.
**```StructType```**을 선언하고,
컬럼에 대해 **```StructField```**를 **컬럼명**, **데이터 타잎**, **NULL**이 허용되는지 여부를 설정한다.
과거 버전에서는 컬럼명이 정렬되면서 age, name 순서가 변경되었다.
현재는 **컬럼명을 정렬하지 않으므로, 순서대로** 아래와 같이 생성하면 된다.


```python
schema=StructType([
    StructField("name", StringType(), True),
    StructField("age", IntegerType(), True),
    #StructField("created", TimestampType(), True)
])
_myDf=spark.createDataFrame(_myRdd, schema)
```


```python
_myDf.printSchema()
```

    root
     |-- name: string (nullable = true)
     |-- age: integer (nullable = true)
    



```python
_myDf.show()
```

    +----+---+
    |name|age|
    +----+---+
    | js1| 10|
    | js2| 20|
    +----+---+
    


schema를 정해서 RDD로부터 DataFrame을 생성할 수 있다.


```python
from pyspark.sql.types import *

myRdd=spark.sparkContext.parallelize([(1, 'kim', 50.0), (2, 'lee', 60.0), (3, 'park', 70.0)])
schema = StructType([
    StructField("id", IntegerType(), True),
    StructField("name", StringType(), True),
    StructField("height", DoubleType(), True)
])
_myDf = spark.createDataFrame(myRdd, schema)
```


```python
_myDf.printSchema()
```

    root
     |-- id: integer (nullable = true)
     |-- name: string (nullable = true)
     |-- height: double (nullable = true)
    



```python
_myDf.show()
```

    +---+----+------+
    | id|name|height|
    +---+----+------+
    |  1| kim|  50.0|
    |  2| lee|  60.0|
    |  3|park|  70.0|
    +---+----+------+
    


## S.4.3 Pandas

Spark Dataframe은 다른 프로그래밍 언어에서도 분석도구로 많이 사용되는 형식이다.
엑셀 스프레드쉬트와 비슷하다. 또한 최근 많이 사용되는 **R**의 Dataframe이나 **Python Pandas**를 예로 들 수 있다.
Spark와 Pandas의 Dataframe을 비교하면,
**Pandas**는 데이터 양이 적은 경우, Spark는 분산처리할 수 있으므로 빅데이터에 보다 적합하다.
API를 사용하게 되면 Spark Dataframe과 Pandas 간에는 차이가 있다.

구분 | DataFrame | Pandas
-------|-------|-------
csv 파일 읽기 | read.json() | read_csv()
데이터타입 | inferschema=True 설정하면 추정 | 모두 strings

### Dataframe을 Pandas로 변환

Spark Dataframe을 ```toPandas()``` 함수를 사용하여 **Pandas로 변환**할 수 있다.


```python
myDf.toPandas()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>_1</th>
      <th>_2</th>
      <th>_3</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>kim, js</td>
      <td>170</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>lee, sm</td>
      <td>175</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>lim, yg</td>
      <td>180</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2</td>
      <td>lee</td>
      <td>170</td>
    </tr>
  </tbody>
</table>
</div>



### Pandas에서 csv 쓰기

Dataframe을 **csv**파일로 내보내고 Pandas로 읽어보자.

Dataframe을 csv로 쓰려면 라이브러리 ```com.databricks.spark.csv```를 사용해야 한다. **파일이 아니라 디렉토리가 생성**되고 그 안에 파일로 쓰여지게 된다. Pandas를 사용하면 우리가 보통 사용하는 하나의 파일로 쓰여진다.

한 번 생성이 되면 덮어쓰기를 하지 않기 때문에, 다시 write() 할 경우에는 이전 디렉토리를 삭제하도록 하자.


```python
myDf.write.format('com.databricks.spark.csv').save(os.path.join('data','_myDf.csv'))
```


```python
!ls -l data/_myDf.csv/
```

    total 4
    -rw-r--r-- 1 jsl jsl 58  9월 16 15:56 part-00000-2627fdd4-7668-4ee9-840c-9e84fdadb86a-c000.csv
    -rw-r--r-- 1 jsl jsl  0  9월 16 15:56 _SUCCESS


Pandas를 이용하여 Dataframe을 csv파일로 내보낼 수 있다.

```python
,year,name,height
0,1,"kim, js",170
1,1,"lee, sm",175
2,2,"lim, yg",180
3,2,lee,170
```


```python
myDf.toPandas().to_csv(os.path.join('data','myDf.csv'))
```

Pandas에서 컬럼을 생성,삭제 해보자.
recode - 현재 변수 값을 다시 줄 경우

나라변 국제전화코드는 Japan: 81, South Korea: 82, Hong Kong: 852, Australia: 61을 사용한다


```python
import pandas as pd
icc = pd.DataFrame( { 'country': ['South Korea','Japan','Hong Kong'],'codes': [81, 82, 852] })
```


```python
icc
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>country</th>
      <th>codes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>South Korea</td>
      <td>81</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Japan</td>
      <td>82</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Hong Kong</td>
      <td>852</td>
    </tr>
  </tbody>
</table>
</div>



전화코드가 81인 경우 출력해보자.


```python
icc[icc['codes']==81]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>country</th>
      <th>codes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>South Korea</td>
      <td>81</td>
    </tr>
  </tbody>
</table>
</div>



### S.4.4 csv 파일에서 생성

#### RDD에서 DataFrame

앞서 RDD에서 읽었던 csv파일을 다시 읽어보자.

```sparkContext.textFile()``` 함수로 읽은 파일은 RDD이다.


```python
from pyspark.sql import Row
cfile= os.path.join("data", "ds_spark_2cols.csv")
lines = spark.sparkContext.textFile(cfile)
```

RDD에서 ```Row()```를 사용하여 Dataframe으로 변환한다.


```python
_col12 = lines.map(lambda l: l.split(","))
col12 = _col12.map(lambda p: Row(col1=int(p[0].strip()), col2=int(p[1].strip())))

_myDf = spark.createDataFrame(col12)
```


```python
_myDf.printSchema()
_myDf.collect()
```

    root
     |-- col1: long (nullable = true)
     |-- col2: long (nullable = true)
    





    [Row(col1=35, col2=2),
     Row(col1=40, col2=27),
     Row(col1=12, col2=38),
     Row(col1=15, col2=31),
     Row(col1=21, col2=1),
     Row(col1=14, col2=19),
     Row(col1=46, col2=1),
     Row(col1=10, col2=34),
     Row(col1=28, col2=3),
     Row(col1=48, col2=1),
     Row(col1=16, col2=2),
     Row(col1=30, col2=3),
     Row(col1=32, col2=2),
     Row(col1=48, col2=1),
     Row(col1=31, col2=2),
     Row(col1=22, col2=1),
     Row(col1=12, col2=3),
     Row(col1=39, col2=29),
     Row(col1=19, col2=37),
     Row(col1=25, col2=2)]



#### DataFrame으로 직접 읽기

format().load() 또는 csv() 함수로 csv 파일을 읽어서 DataFrame을 만들 수 있다.


```python
%%writefile data/ds_spark.csv
1,2,3,4
11,22,33,44
111,222,333,444
```

    Overwriting data/ds_spark.csv


#### format load

csv 패키지를 사용해서 읽어 본다. 우선 [Spark의 csv 패키지를 추가한다.](https://spark-packages.org/package/databricks/spark-csv) 패키지는 설정파일 ```spark-defaults.conf```에 추가할 수 있다.

```python
$ vim conf/spark-defaults.conf
spark.jars.packages=com.databricks:spark-csv_2.11:1.5.0
```

```format("csv").load("path")```이라고 하고, options() 설정을 미리 넣을 수 있다.


```python
df = spark\
        .read\
        .format('com.databricks.spark.csv')\
        .options(header='true', inferschema='true', delimiter=',')\
        .load(os.path.join('data','ds_spark.csv'))
```


```python
df.show()
```

    +---+---+---+---+
    |  1|  2|  3|  4|
    +---+---+---+---+
    | 11| 22| 33| 44|
    |111|222|333|444|
    +---+---+---+---+
    



```python
df.printSchema()
```

    root
     |-- 1: integer (nullable = true)
     |-- 2: integer (nullable = true)
     |-- 3: integer (nullable = true)
     |-- 4: integer (nullable = true)
    


inferschema를 제외하면, string으로 자동인식한다.


```python
df = spark\
        .read\
        .format('com.databricks.spark.csv')\
        .options(header='true', delimiter=',')\
        .load(os.path.join('data','ds_spark.csv'))
```


```python
df.printSchema()
```

    root
     |-- 1: string (nullable = true)
     |-- 2: string (nullable = true)
     |-- 3: string (nullable = true)
     |-- 4: string (nullable = true)
    


#### csv

또는 
```csv("path")```로 직접 DataFrame으로 읽을 수 있다.


```python
df = spark\
        .read\
        .options(header='true', inferschema='true', delimiter=',')\
        .csv(os.path.join('data', 'ds_spark.csv'))
df.show()
```

    +---+---+---+---+
    |  1|  2|  3|  4|
    +---+---+---+---+
    | 11| 22| 33| 44|
    |111|222|333|444|
    +---+---+---+---+
    


### S.4.5 tsv 파일 읽기

tsv (tab-separated values)는 **Tab으로 분리된 파일**을 말한다.
'\t'이 포함되어 있는 경우, 혹시 string으로 데이터타잎을 설정하기도 한다 (과거 Spark 버전에서)

TAB은 whitespace이므로 그냥 split()을 해도 된다.


```python
import numpy as np
np.array([float(x) for x in '1.658985	4.285136'.split()])
```




    array([1.658985, 4.285136])



URL로 가서 http://wiki.stat.ucla.edu/socr/index.php/SOCR_Data_Dinov_020108_HeightsWeights
마우스로 긁어서 50행만 복사해 보자.
18세 1993년 18세 이하 225,000 건의 키 (in), 몸무게 (lbs) 데이터이다.


```python
# %load data/ds_spark_heightweight.txt
1	65.78	112.99
2	71.52	136.49
3	69.40	153.03
4	68.22	142.34
5	67.79	144.30
6	68.70	123.30
7	69.80	141.49
8	70.01	136.46
9	67.90	112.37
10	66.78	120.67
11	66.49	127.45
12	67.62	114.14
13	68.30	125.61
14	67.12	122.46
15	68.28	116.09
16	71.09	140.00
17	66.46	129.50
18	68.65	142.97
19	71.23	137.90
20	67.13	124.04
21	67.83	141.28
22	68.88	143.54
23	63.48	97.90
24	68.42	129.50
25	67.63	141.85
26	67.21	129.72
27	70.84	142.42
28	67.49	131.55
29	66.53	108.33
30	65.44	113.89
31	69.52	103.30
32	65.81	120.75
33	67.82	125.79
34	70.60	136.22
35	71.80	140.10
36	69.21	128.75
37	66.80	141.80
38	67.66	121.23
39	67.81	131.35
40	64.05	106.71
41	68.57	124.36
42	65.18	124.86
43	69.66	139.67
44	67.97	137.37
45	65.98	106.45
46	68.67	128.76
47	66.88	145.68
48	67.70	116.82
49	69.82	143.62
50	69.09	134.93
```

#### RDD

우선 RDD로 읽어보자.


```python
from pyspark.sql.types import *
_tRdd=spark.sparkContext\
    .textFile(os.path.join('data','ds_spark_heightweight.txt'))
```

tsv는 '\t'로 분리해도 되고, split() 함수는 <TAB>을 포함한 whitespace로 분할하게 되므로 그냥 두어도 된다.


```python
#tRdd=rdd.map(lambda x:x.split('\t'))
_tRddSplitted = _tRdd.map(lambda x:x.split())
```

#### 형변환

위 tsv 파일에서 생성한 RDD를 탭으로 분리하면서, ```float()```로 형변환을 해보자.


```python
#import numpy as np
#myRdd=rdd.map(lambda line:np.array([float(x) for x in line.split('\t')]))
tRdd=_tRdd.map(lambda line:[float(x) for x in line.split('\t')])
tRdd.take(1)
```




    [[1.0, 65.78, 112.99]]



#### schema 설정

schema를 **자동으로 설정하면, string**으로 읽혀진다.
schema를 설정한다고 해도, string -> integer, double로 형변환은 이루어지지 않는다.
string의 **형변환을 명시적**으로 해주어야 한다.

```python
mySchema = StructType([
    StructField("id", IntegerType(), True),
    StructField("weight", DoubleType(), True),
    StructField("height", DoubleType(), True)
])
myDf=spark.createDataFrame(myRdd, mySchema)
```

#### DataFrame 생성

위 tRdd로부터 컬럼명을 주어 DataFrame을 생성해보자.


```python
tDfNamed = spark.createDataFrame(tRdd, ["id","weight","height"])
```


```python
tDfNamed.printSchema()
```

    root
     |-- id: double (nullable = true)
     |-- weight: double (nullable = true)
     |-- height: double (nullable = true)
    



```python
tDfNamed.take(1)
```




    [Row(id=1.0, weight=65.78, height=112.99)]



### 컬럼을 split으로 분할

text() 함수를 이용해서 파일을 읽을 수 있다.


```python
tDftxt = spark.read.text(os.path.join('data','ds_spark_heightweight.txt'))
```

<TAB>으로 분리되지 못해 전체를 변수명 value, 타입은 string으로 읽고 있어, 이를 분리해야 한다.


```python
tDftxt.printSchema()
```

    root
     |-- value: string (nullable = true)
    


pyspark.sql.functions은 함수이므로, import할 경우

* ```import pyspark.sql.functions.split``` 이렇게 하지 않고, 
* ```from pyspark.sql.functions import split``` 이렇게 한다.


```python
from pyspark.sql.functions import split

split_col = split(tDftxt['value'], '\t')
```


```python
분리된 컬럼은 getItem() 함수로 가져와서 각 각 weight, height 컬럼이 된다.
```


```python
split_col.getItem(1)
```




    Column<b'split(value, \t, -1)[1]'>




```python
tDftxt = tDftxt.withColumn('weight', split_col.getItem(1))
tDftxt = tDftxt.withColumn('height', split_col.getItem(2))
```


```python
tDftxt.show()
```

    +---------------+------+------+
    |          value|weight|height|
    +---------------+------+------+
    | 1	65.78	112.99| 65.78|112.99|
    | 2	71.52	136.49| 71.52|136.49|
    | 3	69.40	153.03| 69.40|153.03|
    | 4	68.22	142.34| 68.22|142.34|
    | 5	67.79	144.30| 67.79|144.30|
    | 6	68.70	123.30| 68.70|123.30|
    | 7	69.80	141.49| 69.80|141.49|
    | 8	70.01	136.46| 70.01|136.46|
    | 9	67.90	112.37| 67.90|112.37|
    |10	66.78	120.67| 66.78|120.67|
    |11	66.49	127.45| 66.49|127.45|
    |12	67.62	114.14| 67.62|114.14|
    |13	68.30	125.61| 68.30|125.61|
    |14	67.12	122.46| 67.12|122.46|
    |15	68.28	116.09| 68.28|116.09|
    |16	71.09	140.00| 71.09|140.00|
    |17	66.46	129.50| 66.46|129.50|
    |18	68.65	142.97| 68.65|142.97|
    |19	71.23	137.90| 71.23|137.90|
    |20	67.13	124.04| 67.13|124.04|
    +---------------+------+------+
    only showing top 20 rows
    


### Row의 split (앞 tDf에서 split??)

Row 객체의 값은 아래와 같이 단순하게 split 할 수 없으며, 오류가 발생한다. ```dict```로 변환한 후 그 값을 split해야 한다.

```python
_name.rdd.map(lambda line:[line.split(',')])
```

from pyspark.sql import Row
r=Row(name=u'kim, js')
rd=r.asDict()
print (rd.values().split(','))

#### csv 함수로 tsv 읽기


```python
tDf = spark\
    .read\
    .options(header='false', inferschema='true', delimiter='\t')\
    .csv(os.path.join('data', 'ds_spark_heightweight.txt'))
tDf.show()
```

    +---+-----+------+
    |_c0|  _c1|   _c2|
    +---+-----+------+
    |  1|65.78|112.99|
    |  2|71.52|136.49|
    |  3| 69.4|153.03|
    |  4|68.22|142.34|
    |  5|67.79| 144.3|
    |  6| 68.7| 123.3|
    |  7| 69.8|141.49|
    |  8|70.01|136.46|
    |  9| 67.9|112.37|
    | 10|66.78|120.67|
    | 11|66.49|127.45|
    | 12|67.62|114.14|
    | 13| 68.3|125.61|
    | 14|67.12|122.46|
    | 15|68.28|116.09|
    | 16|71.09| 140.0|
    | 17|66.46| 129.5|
    | 18|68.65|142.97|
    | 19|71.23| 137.9|
    | 20|67.13|124.04|
    +---+-----+------+
    only showing top 20 rows
    


### S.4.6 JSON 파일에서 생성

#### JSON
JSON은 JavaScript Object Notation, 즉 자바스크립트에서 사용되는 표기법. 사람이 읽을 수 있는 텍스트로 표기하며, key-value 쌍으로 되어 있다.
현재 널리 쓰이고 있어 XML 대용으로 널리 쓰이고 있다.
Spark example 폴더에 있는 ```people.json``` JSON 파일이다.

```python
{"name":"Michael"}
{"name":"Andy", "age":30}
{"name":"Justin", "age":19}
```

### 트위터 데이터

트위터 데이터는 JSON 형식을 가지고 있다.
아래는 트윗 1개의 샘플이다.
실제 트윗 데이터를 구할 수 없다면 아래 샘플을 파일로 저장한 후 사용하면 된다.

```python
{"contributors": null, "truncated": false, "text": "RT @soompi: #SEVENTEEN’s Mingyu, Jin Se Yeon, And Leeteuk To MC For 2016 Super Seoul Dream Concert \nhttps://t.co/1XRSaRBbE0 https://t.co/fi…", "is_quote_status": false, "in_reply_to_status_id": null, "id": 801657325836763136, "favorite_count": 0, "entities": {"symbols": [], "user_mentions": [{"id": 17659206, "indices": [3, 10], "id_str": "17659206", "screen_name": "soompi", "name": "Soompi"}], "hashtags": [{"indices": [12, 22], "text": "SEVENTEEN"}], "urls": [{"url": "https://t.co/1XRSaRBbE0", "indices": [100, 123], "expanded_url": "http://www.soompi.com/2016/11/20/seventeens-mingyu-jin-se-yeon-leeteuk-mc-dream-concert/", "display_url": "soompi.com/2016/11/20/sev…"}]}, "retweeted": false, "coordinates": null, "source": "<a href=\"http://twitter.com/download/android\" rel=\"nofollow\">Twitter for Android</a>", "in_reply_to_screen_name": null, "in_reply_to_user_id": null, "retweet_count": 1487, "id_str": "801657325836763136", "favorited": false, "retweeted_status": {"contributors": null, "truncated": false, "text": "#SEVENTEEN’s Mingyu, Jin Se Yeon, And Leeteuk To MC For 2016 Super Seoul Dream Concert \nhttps://t.co/1XRSaRBbE0 https://t.co/fifXHpF8or", "is_quote_status": false, "in_reply_to_status_id": null, "id": 800593781586132993, "favorite_count": 1649, "entities": {"symbols": [], "user_mentions": [], "hashtags": [{"indices": [0, 10], "text": "SEVENTEEN"}], "urls": [{"url": "https://t.co/1XRSaRBbE0", "indices": [88, 111], "expanded_url": "http://www.soompi.com/2016/11/20/seventeens-mingyu-jin-se-yeon-leeteuk-mc-dream-concert/", "display_url": "soompi.com/2016/11/20/sev…"}], "media": [{"expanded_url": "https://twitter.com/soompi/status/800593781586132993/photo/1", "display_url": "pic.twitter.com/fifXHpF8or", "url": "https://t.co/fifXHpF8or", "media_url_https": "https://pbs.twimg.com/media/CxxHMk8UsAA4cUT.jpg", "id_str": "800593115165798400", "sizes": {"small": {"h": 382, "resize": "fit", "w": 680}, "large": {"h": 449, "resize": "fit", "w": 800}, "medium": {"h": 449, "resize": "fit", "w": 800}, "thumb": {"h": 150, "resize": "crop", "w": 150}}, "indices": [112, 135], "type": "photo", "id": 800593115165798400, "media_url": "http://pbs.twimg.com/media/CxxHMk8UsAA4cUT.jpg"}]}, "retweeted": false, "coordinates": null, "source": "<a href=\"https://about.twitter.com/products/tweetdeck\" rel=\"nofollow\">TweetDeck</a>", "in_reply_to_screen_name": null, "in_reply_to_user_id": null, "retweet_count": 1487, "id_str": "800593781586132993", "favorited": false, "user": {"follow_request_sent": false, "has_extended_profile": true, "profile_use_background_image": true, "default_profile_image": false, "id": 17659206, "profile_background_image_url_https": "https://pbs.twimg.com/profile_background_images/699864769/1cdde0a85f5c0a994ae1fb06d545a5ec.png", "verified": true, "translator_type": "none", "profile_text_color": "999999", "profile_image_url_https": "https://pbs.twimg.com/profile_images/792117259489583104/4khJk3zz_normal.jpg", "profile_sidebar_fill_color": "000000", "entities": {"url": {"urls": [{"url": "http://t.co/3evT80UlR9", "indices": [0, 22], "expanded_url": "http://www.soompi.com", "display_url": "soompi.com"}]}, "description": {"urls": []}}, "followers_count": 987867, "profile_sidebar_border_color": "000000", "id_str": "17659206", "profile_background_color": "1E1E1E", "listed_count": 3982, "is_translation_enabled": true, "utc_offset": -28800, "statuses_count": 80038, "description": "The original K-pop community. We take gifs, OTPs, and reporting on your bias' fashion choices seriously. But not rumors. Ain't nobody got time for that.", "friends_count": 3532, "location": "Worldwide", "profile_link_color": "31B6F4", "profile_image_url": "http://pbs.twimg.com/profile_images/792117259489583104/4khJk3zz_normal.jpg", "following": false, "geo_enabled": false, "profile_banner_url": "https://pbs.twimg.com/profile_banners/17659206/1478803767", "profile_background_image_url": "http://pbs.twimg.com/profile_background_images/699864769/1cdde0a85f5c0a994ae1fb06d545a5ec.png", "screen_name": "soompi", "lang": "en", "profile_background_tile": true, "favourites_count": 1493, "name": "Soompi", "notifications": false, "url": "http://t.co/3evT80UlR9", "created_at": "Wed Nov 26 20:48:27 +0000 2008", "contributors_enabled": false, "time_zone": "Pacific Time (US & Canada)", "protected": false, "default_profile": false, "is_translator": false}, "geo": null, "in_reply_to_user_id_str": null, "possibly_sensitive": false, "lang": "en", "created_at": "Mon Nov 21 06:56:46 +0000 2016", "in_reply_to_status_id_str": null, "place": null, "extended_entities": {"media": [{"expanded_url": "https://twitter.com/soompi/status/800593781586132993/photo/1", "display_url": "pic.twitter.com/fifXHpF8or", "url": "https://t.co/fifXHpF8or", "media_url_https": "https://pbs.twimg.com/media/CxxHMk8UsAA4cUT.jpg", "id_str": "800593115165798400", "sizes": {"small": {"h": 382, "resize": "fit", "w": 680}, "large": {"h": 449, "resize": "fit", "w": 800}, "medium": {"h": 449, "resize": "fit", "w": 800}, "thumb": {"h": 150, "resize": "crop", "w": 150}}, "indices": [112, 135], "type": "photo", "id": 800593115165798400, "media_url": "http://pbs.twimg.com/media/CxxHMk8UsAA4cUT.jpg"}]}, "metadata": {"iso_language_code": "en", "result_type": "recent"}}, "user": {"follow_request_sent": false, "has_extended_profile": false, "profile_use_background_image": true, "default_profile_image": true, "id": 791090169818521600, "profile_background_image_url_https": null, "verified": false, "translator_type": "none", "profile_text_color": "333333", "profile_image_url_https": "https://abs.twimg.com/sticky/default_profile_images/default_profile_6_normal.png", "profile_sidebar_fill_color": "DDEEF6", "entities": {"description": {"urls": []}}, "followers_count": 0, "profile_sidebar_border_color": "C0DEED", "id_str": "791090169818521600", "profile_background_color": "F5F8FA", "listed_count": 0, "is_translation_enabled": false, "utc_offset": null, "statuses_count": 96, "description": "", "friends_count": 7, "location": "", "profile_link_color": "1DA1F2", "profile_image_url": "http://abs.twimg.com/sticky/default_profile_images/default_profile_6_normal.png", "following": false, "geo_enabled": false, "profile_background_image_url": null, "screen_name": "enriquesanq", "lang": "es", "profile_background_tile": false, "favourites_count": 161, "name": "Enrique santos", "notifications": false, "url": null, "created_at": "Wed Oct 26 01:32:49 +0000 2016", "contributors_enabled": false, "time_zone": null, "protected": false, "default_profile": true, "is_translator": false}, "geo": null, "in_reply_to_user_id_str": null, "possibly_sensitive": false, "lang": "en", "created_at": "Thu Nov 24 05:22:55 +0000 2016", "in_reply_to_status_id_str": null, "place": null, "metadata": {"iso_language_code": "en", "result_type": "recent"}}
```

#### 파일에서 트윗 읽기

그러나 JSON파일은 **JSON이 아니라 문자열**이다. 파일에서 읽은 후 **JSON으로 변환**을 해야 한다.
트윗은 '\n'으로 하나씩 구분되어 있고 ```readlines()``` 함수로 전체를 읽을 수 있다.


```python
import os
_jfname=os.path.join('src','ds_twitter_seoul_3.json')
with open(_jfname, 'rb') as f:
    data = f.readlines()
```


```python
import json
data_json_str = json.loads(data[0])
```


```python
type(data_json_str)
```




    dict




```python
len(data_json_str)
```




    26



#### pandas에서  트윗 읽기


```python
type(data)
```




    list




```python
type(data[0])
```




    bytes



위 Tweet데이터는 json형식이다. 따라서 list 구조로 만들어 주려면, 앞 뒤로 대괄호 ```[ ]```를 넣고, 각 tweet은 컴마로 연결한다.
```join()``` 함수는 인자를 구분자 ","로 병합한다.

```python
",".join(["A", "B", "C"]) # A,B,C
```

data는 bytes이고, 이를 join()하는 함수는 str을 대상으로 하기 때문에, 문자열에 ```b```를 붙여 bytes로 변환한 후 연결한다.



```python
data_json_str = b"[" + b','.join(data) + b"]"
```

아직 JSON으로 변환이 되지 않았고, 문자열 전체 길이를 알아 보자. 파일의 크기를 알 수 있다.


```python
len(data_json_str)
```




    11310468



이제 Pandas로 읽어 보자. Pandas 라이브러리에서 제공하는 read_json()함수를 사용한다.

read_json()에 경로를 넘겨주어도 되지 않나??????


```python
import pandas as pd

tweetPd = pd.read_json(data_json_str)
```

shape()은 행과 열의 갯수를 알려준다.


```python
tweetPd.shape
```




    (2013, 30)



count() 함수는 행의 갯수를 나타내는데, 숫자가 서로 다른 것은 비워있는 경우가 서로 다르기 때문이다.


```python
print (tweetPd.count())
```

    contributors                    0
    truncated                    2013
    text                         2013
    is_quote_status              2013
    in_reply_to_status_id         131
    id                           2013
    favorite_count               2013
    entities                     2013
    retweeted                    2013
    coordinates                    55
    source                       2013
    in_reply_to_screen_name       153
    in_reply_to_user_id           153
    retweet_count                2013
    id_str                       2013
    favorited                    2013
    retweeted_status             1304
    user                         2013
    geo                            55
    in_reply_to_user_id_str       153
    possibly_sensitive           1097
    lang                         2013
    created_at                   2013
    in_reply_to_status_id_str     131
    place                          65
    metadata                     2013
    extended_entities             506
    quoted_status_id              133
    quoted_status                  19
    quoted_status_id_str          133
    dtype: int64


컬럼 'id'를 10개만 읽어 보자.


```python
tweetPd['id'][:10]
```




    0    801657325836763136
    1    801657325677400064
    2    801657307637678080
    3    801657305628430336
    4    801657297449586688
    5    801657287697895424
    6    801657280760397824
    7    801657276788523008
    8    801657268177604608
    9    801657258400616449
    Name: id, dtype: int64



#### DataFrame에서 트윗 읽기



```python
jfile= os.path.join('src','ds_twitter_seoul_3.json')

tweetDf= spark.read.json(jfile)
```


```python
tweetDf.printSchema()
```

    root
     |-- contributors: string (nullable = true)
     |-- coordinates: struct (nullable = true)
     |    |-- coordinates: array (nullable = true)
     |    |    |-- element: double (containsNull = true)
     |    |-- type: string (nullable = true)
     |-- created_at: string (nullable = true)
     |-- entities: struct (nullable = true)
     |    |-- hashtags: array (nullable = true)
     |    |    |-- element: struct (containsNull = true)
     |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |-- text: string (nullable = true)
     |    |-- media: array (nullable = true)
     |    |    |-- element: struct (containsNull = true)
     |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |-- id: long (nullable = true)
     |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |-- media_url: string (nullable = true)
     |    |    |    |-- media_url_https: string (nullable = true)
     |    |    |    |-- sizes: struct (nullable = true)
     |    |    |    |    |-- large: struct (nullable = true)
     |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- medium: struct (nullable = true)
     |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- small: struct (nullable = true)
     |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- thumb: struct (nullable = true)
     |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |-- source_status_id: long (nullable = true)
     |    |    |    |-- source_status_id_str: string (nullable = true)
     |    |    |    |-- source_user_id: long (nullable = true)
     |    |    |    |-- source_user_id_str: string (nullable = true)
     |    |    |    |-- type: string (nullable = true)
     |    |    |    |-- url: string (nullable = true)
     |    |-- symbols: array (nullable = true)
     |    |    |-- element: string (containsNull = true)
     |    |-- urls: array (nullable = true)
     |    |    |-- element: struct (containsNull = true)
     |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |-- url: string (nullable = true)
     |    |-- user_mentions: array (nullable = true)
     |    |    |-- element: struct (containsNull = true)
     |    |    |    |-- id: long (nullable = true)
     |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |-- name: string (nullable = true)
     |    |    |    |-- screen_name: string (nullable = true)
     |-- extended_entities: struct (nullable = true)
     |    |-- media: array (nullable = true)
     |    |    |-- element: struct (containsNull = true)
     |    |    |    |-- additional_media_info: struct (nullable = true)
     |    |    |    |    |-- description: string (nullable = true)
     |    |    |    |    |-- embeddable: boolean (nullable = true)
     |    |    |    |    |-- monetizable: boolean (nullable = true)
     |    |    |    |    |-- source_user: struct (nullable = true)
     |    |    |    |    |    |-- contributors_enabled: boolean (nullable = true)
     |    |    |    |    |    |-- created_at: string (nullable = true)
     |    |    |    |    |    |-- default_profile: boolean (nullable = true)
     |    |    |    |    |    |-- default_profile_image: boolean (nullable = true)
     |    |    |    |    |    |-- description: string (nullable = true)
     |    |    |    |    |    |-- entities: struct (nullable = true)
     |    |    |    |    |    |    |-- description: struct (nullable = true)
     |    |    |    |    |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |    |    |    |-- url: struct (nullable = true)
     |    |    |    |    |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |    |    |-- favourites_count: long (nullable = true)
     |    |    |    |    |    |-- follow_request_sent: boolean (nullable = true)
     |    |    |    |    |    |-- followers_count: long (nullable = true)
     |    |    |    |    |    |-- following: boolean (nullable = true)
     |    |    |    |    |    |-- friends_count: long (nullable = true)
     |    |    |    |    |    |-- geo_enabled: boolean (nullable = true)
     |    |    |    |    |    |-- has_extended_profile: boolean (nullable = true)
     |    |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |    |-- is_translation_enabled: boolean (nullable = true)
     |    |    |    |    |    |-- is_translator: boolean (nullable = true)
     |    |    |    |    |    |-- lang: string (nullable = true)
     |    |    |    |    |    |-- listed_count: long (nullable = true)
     |    |    |    |    |    |-- location: string (nullable = true)
     |    |    |    |    |    |-- name: string (nullable = true)
     |    |    |    |    |    |-- notifications: boolean (nullable = true)
     |    |    |    |    |    |-- profile_background_color: string (nullable = true)
     |    |    |    |    |    |-- profile_background_image_url: string (nullable = true)
     |    |    |    |    |    |-- profile_background_image_url_https: string (nullable = true)
     |    |    |    |    |    |-- profile_background_tile: boolean (nullable = true)
     |    |    |    |    |    |-- profile_banner_url: string (nullable = true)
     |    |    |    |    |    |-- profile_image_url: string (nullable = true)
     |    |    |    |    |    |-- profile_image_url_https: string (nullable = true)
     |    |    |    |    |    |-- profile_link_color: string (nullable = true)
     |    |    |    |    |    |-- profile_sidebar_border_color: string (nullable = true)
     |    |    |    |    |    |-- profile_sidebar_fill_color: string (nullable = true)
     |    |    |    |    |    |-- profile_text_color: string (nullable = true)
     |    |    |    |    |    |-- profile_use_background_image: boolean (nullable = true)
     |    |    |    |    |    |-- protected: boolean (nullable = true)
     |    |    |    |    |    |-- screen_name: string (nullable = true)
     |    |    |    |    |    |-- statuses_count: long (nullable = true)
     |    |    |    |    |    |-- time_zone: string (nullable = true)
     |    |    |    |    |    |-- translator_type: string (nullable = true)
     |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |    |    |-- utc_offset: long (nullable = true)
     |    |    |    |    |    |-- verified: boolean (nullable = true)
     |    |    |    |    |-- title: string (nullable = true)
     |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |-- id: long (nullable = true)
     |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |-- media_url: string (nullable = true)
     |    |    |    |-- media_url_https: string (nullable = true)
     |    |    |    |-- sizes: struct (nullable = true)
     |    |    |    |    |-- large: struct (nullable = true)
     |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- medium: struct (nullable = true)
     |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- small: struct (nullable = true)
     |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- thumb: struct (nullable = true)
     |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |-- source_status_id: long (nullable = true)
     |    |    |    |-- source_status_id_str: string (nullable = true)
     |    |    |    |-- source_user_id: long (nullable = true)
     |    |    |    |-- source_user_id_str: string (nullable = true)
     |    |    |    |-- type: string (nullable = true)
     |    |    |    |-- url: string (nullable = true)
     |    |    |    |-- video_info: struct (nullable = true)
     |    |    |    |    |-- aspect_ratio: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- duration_millis: long (nullable = true)
     |    |    |    |    |-- variants: array (nullable = true)
     |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |-- bitrate: long (nullable = true)
     |    |    |    |    |    |    |-- content_type: string (nullable = true)
     |    |    |    |    |    |    |-- url: string (nullable = true)
     |-- favorite_count: long (nullable = true)
     |-- favorited: boolean (nullable = true)
     |-- geo: struct (nullable = true)
     |    |-- coordinates: array (nullable = true)
     |    |    |-- element: double (containsNull = true)
     |    |-- type: string (nullable = true)
     |-- id: long (nullable = true)
     |-- id_str: string (nullable = true)
     |-- in_reply_to_screen_name: string (nullable = true)
     |-- in_reply_to_status_id: long (nullable = true)
     |-- in_reply_to_status_id_str: string (nullable = true)
     |-- in_reply_to_user_id: long (nullable = true)
     |-- in_reply_to_user_id_str: string (nullable = true)
     |-- is_quote_status: boolean (nullable = true)
     |-- lang: string (nullable = true)
     |-- metadata: struct (nullable = true)
     |    |-- iso_language_code: string (nullable = true)
     |    |-- result_type: string (nullable = true)
     |-- place: struct (nullable = true)
     |    |-- bounding_box: struct (nullable = true)
     |    |    |-- coordinates: array (nullable = true)
     |    |    |    |-- element: array (containsNull = true)
     |    |    |    |    |-- element: array (containsNull = true)
     |    |    |    |    |    |-- element: double (containsNull = true)
     |    |    |-- type: string (nullable = true)
     |    |-- contained_within: array (nullable = true)
     |    |    |-- element: string (containsNull = true)
     |    |-- country: string (nullable = true)
     |    |-- country_code: string (nullable = true)
     |    |-- full_name: string (nullable = true)
     |    |-- id: string (nullable = true)
     |    |-- name: string (nullable = true)
     |    |-- place_type: string (nullable = true)
     |    |-- url: string (nullable = true)
     |-- possibly_sensitive: boolean (nullable = true)
     |-- quoted_status: struct (nullable = true)
     |    |-- contributors: string (nullable = true)
     |    |-- coordinates: struct (nullable = true)
     |    |    |-- coordinates: array (nullable = true)
     |    |    |    |-- element: double (containsNull = true)
     |    |    |-- type: string (nullable = true)
     |    |-- created_at: string (nullable = true)
     |    |-- entities: struct (nullable = true)
     |    |    |-- hashtags: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- text: string (nullable = true)
     |    |    |-- media: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- media_url: string (nullable = true)
     |    |    |    |    |-- media_url_https: string (nullable = true)
     |    |    |    |    |-- sizes: struct (nullable = true)
     |    |    |    |    |    |-- large: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- medium: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- small: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- thumb: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- source_status_id: long (nullable = true)
     |    |    |    |    |-- source_status_id_str: string (nullable = true)
     |    |    |    |    |-- source_user_id: long (nullable = true)
     |    |    |    |    |-- source_user_id_str: string (nullable = true)
     |    |    |    |    |-- type: string (nullable = true)
     |    |    |    |    |-- url: string (nullable = true)
     |    |    |-- symbols: array (nullable = true)
     |    |    |    |-- element: string (containsNull = true)
     |    |    |-- urls: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- url: string (nullable = true)
     |    |    |-- user_mentions: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- name: string (nullable = true)
     |    |    |    |    |-- screen_name: string (nullable = true)
     |    |-- extended_entities: struct (nullable = true)
     |    |    |-- media: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- media_url: string (nullable = true)
     |    |    |    |    |-- media_url_https: string (nullable = true)
     |    |    |    |    |-- sizes: struct (nullable = true)
     |    |    |    |    |    |-- large: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- medium: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- small: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- thumb: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- source_status_id: long (nullable = true)
     |    |    |    |    |-- source_status_id_str: string (nullable = true)
     |    |    |    |    |-- source_user_id: long (nullable = true)
     |    |    |    |    |-- source_user_id_str: string (nullable = true)
     |    |    |    |    |-- type: string (nullable = true)
     |    |    |    |    |-- url: string (nullable = true)
     |    |-- favorite_count: long (nullable = true)
     |    |-- favorited: boolean (nullable = true)
     |    |-- geo: struct (nullable = true)
     |    |    |-- coordinates: array (nullable = true)
     |    |    |    |-- element: double (containsNull = true)
     |    |    |-- type: string (nullable = true)
     |    |-- id: long (nullable = true)
     |    |-- id_str: string (nullable = true)
     |    |-- in_reply_to_screen_name: string (nullable = true)
     |    |-- in_reply_to_status_id: long (nullable = true)
     |    |-- in_reply_to_status_id_str: string (nullable = true)
     |    |-- in_reply_to_user_id: long (nullable = true)
     |    |-- in_reply_to_user_id_str: string (nullable = true)
     |    |-- is_quote_status: boolean (nullable = true)
     |    |-- lang: string (nullable = true)
     |    |-- metadata: struct (nullable = true)
     |    |    |-- iso_language_code: string (nullable = true)
     |    |    |-- result_type: string (nullable = true)
     |    |-- place: struct (nullable = true)
     |    |    |-- bounding_box: struct (nullable = true)
     |    |    |    |-- coordinates: array (nullable = true)
     |    |    |    |    |-- element: array (containsNull = true)
     |    |    |    |    |    |-- element: array (containsNull = true)
     |    |    |    |    |    |    |-- element: double (containsNull = true)
     |    |    |    |-- type: string (nullable = true)
     |    |    |-- contained_within: array (nullable = true)
     |    |    |    |-- element: string (containsNull = true)
     |    |    |-- country: string (nullable = true)
     |    |    |-- country_code: string (nullable = true)
     |    |    |-- full_name: string (nullable = true)
     |    |    |-- id: string (nullable = true)
     |    |    |-- name: string (nullable = true)
     |    |    |-- place_type: string (nullable = true)
     |    |    |-- url: string (nullable = true)
     |    |-- possibly_sensitive: boolean (nullable = true)
     |    |-- quoted_status_id: long (nullable = true)
     |    |-- quoted_status_id_str: string (nullable = true)
     |    |-- retweet_count: long (nullable = true)
     |    |-- retweeted: boolean (nullable = true)
     |    |-- source: string (nullable = true)
     |    |-- text: string (nullable = true)
     |    |-- truncated: boolean (nullable = true)
     |    |-- user: struct (nullable = true)
     |    |    |-- contributors_enabled: boolean (nullable = true)
     |    |    |-- created_at: string (nullable = true)
     |    |    |-- default_profile: boolean (nullable = true)
     |    |    |-- default_profile_image: boolean (nullable = true)
     |    |    |-- description: string (nullable = true)
     |    |    |-- entities: struct (nullable = true)
     |    |    |    |-- description: struct (nullable = true)
     |    |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |-- url: struct (nullable = true)
     |    |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |-- favourites_count: long (nullable = true)
     |    |    |-- follow_request_sent: boolean (nullable = true)
     |    |    |-- followers_count: long (nullable = true)
     |    |    |-- following: boolean (nullable = true)
     |    |    |-- friends_count: long (nullable = true)
     |    |    |-- geo_enabled: boolean (nullable = true)
     |    |    |-- has_extended_profile: boolean (nullable = true)
     |    |    |-- id: long (nullable = true)
     |    |    |-- id_str: string (nullable = true)
     |    |    |-- is_translation_enabled: boolean (nullable = true)
     |    |    |-- is_translator: boolean (nullable = true)
     |    |    |-- lang: string (nullable = true)
     |    |    |-- listed_count: long (nullable = true)
     |    |    |-- location: string (nullable = true)
     |    |    |-- name: string (nullable = true)
     |    |    |-- notifications: boolean (nullable = true)
     |    |    |-- profile_background_color: string (nullable = true)
     |    |    |-- profile_background_image_url: string (nullable = true)
     |    |    |-- profile_background_image_url_https: string (nullable = true)
     |    |    |-- profile_background_tile: boolean (nullable = true)
     |    |    |-- profile_banner_url: string (nullable = true)
     |    |    |-- profile_image_url: string (nullable = true)
     |    |    |-- profile_image_url_https: string (nullable = true)
     |    |    |-- profile_link_color: string (nullable = true)
     |    |    |-- profile_sidebar_border_color: string (nullable = true)
     |    |    |-- profile_sidebar_fill_color: string (nullable = true)
     |    |    |-- profile_text_color: string (nullable = true)
     |    |    |-- profile_use_background_image: boolean (nullable = true)
     |    |    |-- protected: boolean (nullable = true)
     |    |    |-- screen_name: string (nullable = true)
     |    |    |-- statuses_count: long (nullable = true)
     |    |    |-- time_zone: string (nullable = true)
     |    |    |-- translator_type: string (nullable = true)
     |    |    |-- url: string (nullable = true)
     |    |    |-- utc_offset: long (nullable = true)
     |    |    |-- verified: boolean (nullable = true)
     |-- quoted_status_id: long (nullable = true)
     |-- quoted_status_id_str: string (nullable = true)
     |-- retweet_count: long (nullable = true)
     |-- retweeted: boolean (nullable = true)
     |-- retweeted_status: struct (nullable = true)
     |    |-- contributors: string (nullable = true)
     |    |-- coordinates: struct (nullable = true)
     |    |    |-- coordinates: array (nullable = true)
     |    |    |    |-- element: double (containsNull = true)
     |    |    |-- type: string (nullable = true)
     |    |-- created_at: string (nullable = true)
     |    |-- entities: struct (nullable = true)
     |    |    |-- hashtags: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- text: string (nullable = true)
     |    |    |-- media: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- media_url: string (nullable = true)
     |    |    |    |    |-- media_url_https: string (nullable = true)
     |    |    |    |    |-- sizes: struct (nullable = true)
     |    |    |    |    |    |-- large: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- medium: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- small: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- thumb: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- source_status_id: long (nullable = true)
     |    |    |    |    |-- source_status_id_str: string (nullable = true)
     |    |    |    |    |-- source_user_id: long (nullable = true)
     |    |    |    |    |-- source_user_id_str: string (nullable = true)
     |    |    |    |    |-- type: string (nullable = true)
     |    |    |    |    |-- url: string (nullable = true)
     |    |    |-- symbols: array (nullable = true)
     |    |    |    |-- element: string (containsNull = true)
     |    |    |-- urls: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- url: string (nullable = true)
     |    |    |-- user_mentions: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- name: string (nullable = true)
     |    |    |    |    |-- screen_name: string (nullable = true)
     |    |-- extended_entities: struct (nullable = true)
     |    |    |-- media: array (nullable = true)
     |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |-- additional_media_info: struct (nullable = true)
     |    |    |    |    |    |-- description: string (nullable = true)
     |    |    |    |    |    |-- embeddable: boolean (nullable = true)
     |    |    |    |    |    |-- monetizable: boolean (nullable = true)
     |    |    |    |    |    |-- title: string (nullable = true)
     |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |-- media_url: string (nullable = true)
     |    |    |    |    |-- media_url_https: string (nullable = true)
     |    |    |    |    |-- sizes: struct (nullable = true)
     |    |    |    |    |    |-- large: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- medium: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- small: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- thumb: struct (nullable = true)
     |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |-- source_status_id: long (nullable = true)
     |    |    |    |    |-- source_status_id_str: string (nullable = true)
     |    |    |    |    |-- source_user_id: long (nullable = true)
     |    |    |    |    |-- source_user_id_str: string (nullable = true)
     |    |    |    |    |-- type: string (nullable = true)
     |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |    |-- video_info: struct (nullable = true)
     |    |    |    |    |    |-- aspect_ratio: array (nullable = true)
     |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |-- duration_millis: long (nullable = true)
     |    |    |    |    |    |-- variants: array (nullable = true)
     |    |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |    |-- bitrate: long (nullable = true)
     |    |    |    |    |    |    |    |-- content_type: string (nullable = true)
     |    |    |    |    |    |    |    |-- url: string (nullable = true)
     |    |-- favorite_count: long (nullable = true)
     |    |-- favorited: boolean (nullable = true)
     |    |-- geo: struct (nullable = true)
     |    |    |-- coordinates: array (nullable = true)
     |    |    |    |-- element: double (containsNull = true)
     |    |    |-- type: string (nullable = true)
     |    |-- id: long (nullable = true)
     |    |-- id_str: string (nullable = true)
     |    |-- in_reply_to_screen_name: string (nullable = true)
     |    |-- in_reply_to_status_id: long (nullable = true)
     |    |-- in_reply_to_status_id_str: string (nullable = true)
     |    |-- in_reply_to_user_id: long (nullable = true)
     |    |-- in_reply_to_user_id_str: string (nullable = true)
     |    |-- is_quote_status: boolean (nullable = true)
     |    |-- lang: string (nullable = true)
     |    |-- metadata: struct (nullable = true)
     |    |    |-- iso_language_code: string (nullable = true)
     |    |    |-- result_type: string (nullable = true)
     |    |-- place: struct (nullable = true)
     |    |    |-- bounding_box: struct (nullable = true)
     |    |    |    |-- coordinates: array (nullable = true)
     |    |    |    |    |-- element: array (containsNull = true)
     |    |    |    |    |    |-- element: array (containsNull = true)
     |    |    |    |    |    |    |-- element: double (containsNull = true)
     |    |    |    |-- type: string (nullable = true)
     |    |    |-- contained_within: array (nullable = true)
     |    |    |    |-- element: string (containsNull = true)
     |    |    |-- country: string (nullable = true)
     |    |    |-- country_code: string (nullable = true)
     |    |    |-- full_name: string (nullable = true)
     |    |    |-- id: string (nullable = true)
     |    |    |-- name: string (nullable = true)
     |    |    |-- place_type: string (nullable = true)
     |    |    |-- url: string (nullable = true)
     |    |-- possibly_sensitive: boolean (nullable = true)
     |    |-- quoted_status: struct (nullable = true)
     |    |    |-- contributors: string (nullable = true)
     |    |    |-- coordinates: string (nullable = true)
     |    |    |-- created_at: string (nullable = true)
     |    |    |-- entities: struct (nullable = true)
     |    |    |    |-- hashtags: array (nullable = true)
     |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |-- text: string (nullable = true)
     |    |    |    |-- media: array (nullable = true)
     |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |-- media_url: string (nullable = true)
     |    |    |    |    |    |-- media_url_https: string (nullable = true)
     |    |    |    |    |    |-- sizes: struct (nullable = true)
     |    |    |    |    |    |    |-- large: struct (nullable = true)
     |    |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |    |-- medium: struct (nullable = true)
     |    |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |    |-- small: struct (nullable = true)
     |    |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |    |-- thumb: struct (nullable = true)
     |    |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- type: string (nullable = true)
     |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |-- symbols: array (nullable = true)
     |    |    |    |    |-- element: string (containsNull = true)
     |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |-- user_mentions: array (nullable = true)
     |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |-- name: string (nullable = true)
     |    |    |    |    |    |-- screen_name: string (nullable = true)
     |    |    |-- extended_entities: struct (nullable = true)
     |    |    |    |-- media: array (nullable = true)
     |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |-- id: long (nullable = true)
     |    |    |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |-- media_url: string (nullable = true)
     |    |    |    |    |    |-- media_url_https: string (nullable = true)
     |    |    |    |    |    |-- sizes: struct (nullable = true)
     |    |    |    |    |    |    |-- large: struct (nullable = true)
     |    |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |    |-- medium: struct (nullable = true)
     |    |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |    |-- small: struct (nullable = true)
     |    |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |    |-- thumb: struct (nullable = true)
     |    |    |    |    |    |    |    |-- h: long (nullable = true)
     |    |    |    |    |    |    |    |-- resize: string (nullable = true)
     |    |    |    |    |    |    |    |-- w: long (nullable = true)
     |    |    |    |    |    |-- type: string (nullable = true)
     |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |-- favorite_count: long (nullable = true)
     |    |    |-- favorited: boolean (nullable = true)
     |    |    |-- geo: string (nullable = true)
     |    |    |-- id: long (nullable = true)
     |    |    |-- id_str: string (nullable = true)
     |    |    |-- in_reply_to_screen_name: string (nullable = true)
     |    |    |-- in_reply_to_status_id: long (nullable = true)
     |    |    |-- in_reply_to_status_id_str: string (nullable = true)
     |    |    |-- in_reply_to_user_id: long (nullable = true)
     |    |    |-- in_reply_to_user_id_str: string (nullable = true)
     |    |    |-- is_quote_status: boolean (nullable = true)
     |    |    |-- lang: string (nullable = true)
     |    |    |-- metadata: struct (nullable = true)
     |    |    |    |-- iso_language_code: string (nullable = true)
     |    |    |    |-- result_type: string (nullable = true)
     |    |    |-- place: struct (nullable = true)
     |    |    |    |-- bounding_box: struct (nullable = true)
     |    |    |    |    |-- coordinates: array (nullable = true)
     |    |    |    |    |    |-- element: array (containsNull = true)
     |    |    |    |    |    |    |-- element: array (containsNull = true)
     |    |    |    |    |    |    |    |-- element: double (containsNull = true)
     |    |    |    |    |-- type: string (nullable = true)
     |    |    |    |-- contained_within: array (nullable = true)
     |    |    |    |    |-- element: string (containsNull = true)
     |    |    |    |-- country: string (nullable = true)
     |    |    |    |-- country_code: string (nullable = true)
     |    |    |    |-- full_name: string (nullable = true)
     |    |    |    |-- id: string (nullable = true)
     |    |    |    |-- name: string (nullable = true)
     |    |    |    |-- place_type: string (nullable = true)
     |    |    |    |-- url: string (nullable = true)
     |    |    |-- possibly_sensitive: boolean (nullable = true)
     |    |    |-- quoted_status_id: long (nullable = true)
     |    |    |-- quoted_status_id_str: string (nullable = true)
     |    |    |-- retweet_count: long (nullable = true)
     |    |    |-- retweeted: boolean (nullable = true)
     |    |    |-- source: string (nullable = true)
     |    |    |-- text: string (nullable = true)
     |    |    |-- truncated: boolean (nullable = true)
     |    |    |-- user: struct (nullable = true)
     |    |    |    |-- contributors_enabled: boolean (nullable = true)
     |    |    |    |-- created_at: string (nullable = true)
     |    |    |    |-- default_profile: boolean (nullable = true)
     |    |    |    |-- default_profile_image: boolean (nullable = true)
     |    |    |    |-- description: string (nullable = true)
     |    |    |    |-- entities: struct (nullable = true)
     |    |    |    |    |-- description: struct (nullable = true)
     |    |    |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |    |    |-- element: string (containsNull = true)
     |    |    |    |    |-- url: struct (nullable = true)
     |    |    |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |-- favourites_count: long (nullable = true)
     |    |    |    |-- follow_request_sent: boolean (nullable = true)
     |    |    |    |-- followers_count: long (nullable = true)
     |    |    |    |-- following: boolean (nullable = true)
     |    |    |    |-- friends_count: long (nullable = true)
     |    |    |    |-- geo_enabled: boolean (nullable = true)
     |    |    |    |-- has_extended_profile: boolean (nullable = true)
     |    |    |    |-- id: long (nullable = true)
     |    |    |    |-- id_str: string (nullable = true)
     |    |    |    |-- is_translation_enabled: boolean (nullable = true)
     |    |    |    |-- is_translator: boolean (nullable = true)
     |    |    |    |-- lang: string (nullable = true)
     |    |    |    |-- listed_count: long (nullable = true)
     |    |    |    |-- location: string (nullable = true)
     |    |    |    |-- name: string (nullable = true)
     |    |    |    |-- notifications: boolean (nullable = true)
     |    |    |    |-- profile_background_color: string (nullable = true)
     |    |    |    |-- profile_background_image_url: string (nullable = true)
     |    |    |    |-- profile_background_image_url_https: string (nullable = true)
     |    |    |    |-- profile_background_tile: boolean (nullable = true)
     |    |    |    |-- profile_banner_url: string (nullable = true)
     |    |    |    |-- profile_image_url: string (nullable = true)
     |    |    |    |-- profile_image_url_https: string (nullable = true)
     |    |    |    |-- profile_link_color: string (nullable = true)
     |    |    |    |-- profile_sidebar_border_color: string (nullable = true)
     |    |    |    |-- profile_sidebar_fill_color: string (nullable = true)
     |    |    |    |-- profile_text_color: string (nullable = true)
     |    |    |    |-- profile_use_background_image: boolean (nullable = true)
     |    |    |    |-- protected: boolean (nullable = true)
     |    |    |    |-- screen_name: string (nullable = true)
     |    |    |    |-- statuses_count: long (nullable = true)
     |    |    |    |-- time_zone: string (nullable = true)
     |    |    |    |-- translator_type: string (nullable = true)
     |    |    |    |-- url: string (nullable = true)
     |    |    |    |-- utc_offset: long (nullable = true)
     |    |    |    |-- verified: boolean (nullable = true)
     |    |-- quoted_status_id: long (nullable = true)
     |    |-- quoted_status_id_str: string (nullable = true)
     |    |-- retweet_count: long (nullable = true)
     |    |-- retweeted: boolean (nullable = true)
     |    |-- scopes: struct (nullable = true)
     |    |    |-- followers: boolean (nullable = true)
     |    |-- source: string (nullable = true)
     |    |-- text: string (nullable = true)
     |    |-- truncated: boolean (nullable = true)
     |    |-- user: struct (nullable = true)
     |    |    |-- contributors_enabled: boolean (nullable = true)
     |    |    |-- created_at: string (nullable = true)
     |    |    |-- default_profile: boolean (nullable = true)
     |    |    |-- default_profile_image: boolean (nullable = true)
     |    |    |-- description: string (nullable = true)
     |    |    |-- entities: struct (nullable = true)
     |    |    |    |-- description: struct (nullable = true)
     |    |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |    |-- url: struct (nullable = true)
     |    |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |-- favourites_count: long (nullable = true)
     |    |    |-- follow_request_sent: boolean (nullable = true)
     |    |    |-- followers_count: long (nullable = true)
     |    |    |-- following: boolean (nullable = true)
     |    |    |-- friends_count: long (nullable = true)
     |    |    |-- geo_enabled: boolean (nullable = true)
     |    |    |-- has_extended_profile: boolean (nullable = true)
     |    |    |-- id: long (nullable = true)
     |    |    |-- id_str: string (nullable = true)
     |    |    |-- is_translation_enabled: boolean (nullable = true)
     |    |    |-- is_translator: boolean (nullable = true)
     |    |    |-- lang: string (nullable = true)
     |    |    |-- listed_count: long (nullable = true)
     |    |    |-- location: string (nullable = true)
     |    |    |-- name: string (nullable = true)
     |    |    |-- notifications: boolean (nullable = true)
     |    |    |-- profile_background_color: string (nullable = true)
     |    |    |-- profile_background_image_url: string (nullable = true)
     |    |    |-- profile_background_image_url_https: string (nullable = true)
     |    |    |-- profile_background_tile: boolean (nullable = true)
     |    |    |-- profile_banner_url: string (nullable = true)
     |    |    |-- profile_image_url: string (nullable = true)
     |    |    |-- profile_image_url_https: string (nullable = true)
     |    |    |-- profile_link_color: string (nullable = true)
     |    |    |-- profile_sidebar_border_color: string (nullable = true)
     |    |    |-- profile_sidebar_fill_color: string (nullable = true)
     |    |    |-- profile_text_color: string (nullable = true)
     |    |    |-- profile_use_background_image: boolean (nullable = true)
     |    |    |-- protected: boolean (nullable = true)
     |    |    |-- screen_name: string (nullable = true)
     |    |    |-- statuses_count: long (nullable = true)
     |    |    |-- time_zone: string (nullable = true)
     |    |    |-- translator_type: string (nullable = true)
     |    |    |-- url: string (nullable = true)
     |    |    |-- utc_offset: long (nullable = true)
     |    |    |-- verified: boolean (nullable = true)
     |-- source: string (nullable = true)
     |-- text: string (nullable = true)
     |-- truncated: boolean (nullable = true)
     |-- user: struct (nullable = true)
     |    |-- contributors_enabled: boolean (nullable = true)
     |    |-- created_at: string (nullable = true)
     |    |-- default_profile: boolean (nullable = true)
     |    |-- default_profile_image: boolean (nullable = true)
     |    |-- description: string (nullable = true)
     |    |-- entities: struct (nullable = true)
     |    |    |-- description: struct (nullable = true)
     |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |-- url: string (nullable = true)
     |    |    |-- url: struct (nullable = true)
     |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |-- element: struct (containsNull = true)
     |    |    |    |    |    |-- display_url: string (nullable = true)
     |    |    |    |    |    |-- expanded_url: string (nullable = true)
     |    |    |    |    |    |-- indices: array (nullable = true)
     |    |    |    |    |    |    |-- element: long (containsNull = true)
     |    |    |    |    |    |-- url: string (nullable = true)
     |    |-- favourites_count: long (nullable = true)
     |    |-- follow_request_sent: boolean (nullable = true)
     |    |-- followers_count: long (nullable = true)
     |    |-- following: boolean (nullable = true)
     |    |-- friends_count: long (nullable = true)
     |    |-- geo_enabled: boolean (nullable = true)
     |    |-- has_extended_profile: boolean (nullable = true)
     |    |-- id: long (nullable = true)
     |    |-- id_str: string (nullable = true)
     |    |-- is_translation_enabled: boolean (nullable = true)
     |    |-- is_translator: boolean (nullable = true)
     |    |-- lang: string (nullable = true)
     |    |-- listed_count: long (nullable = true)
     |    |-- location: string (nullable = true)
     |    |-- name: string (nullable = true)
     |    |-- notifications: boolean (nullable = true)
     |    |-- profile_background_color: string (nullable = true)
     |    |-- profile_background_image_url: string (nullable = true)
     |    |-- profile_background_image_url_https: string (nullable = true)
     |    |-- profile_background_tile: boolean (nullable = true)
     |    |-- profile_banner_url: string (nullable = true)
     |    |-- profile_image_url: string (nullable = true)
     |    |-- profile_image_url_https: string (nullable = true)
     |    |-- profile_link_color: string (nullable = true)
     |    |-- profile_sidebar_border_color: string (nullable = true)
     |    |-- profile_sidebar_fill_color: string (nullable = true)
     |    |-- profile_text_color: string (nullable = true)
     |    |-- profile_use_background_image: boolean (nullable = true)
     |    |-- protected: boolean (nullable = true)
     |    |-- screen_name: string (nullable = true)
     |    |-- statuses_count: long (nullable = true)
     |    |-- time_zone: string (nullable = true)
     |    |-- translator_type: string (nullable = true)
     |    |-- url: string (nullable = true)
     |    |-- utc_offset: long (nullable = true)
     |    |-- verified: boolean (nullable = true)
    



```python
tweetDf.count()
```




    2013



Spark DataFrame에서도 위 Pandas와 컬럼 'id'를 10개 출력할 수 있다.


```python
tweetDf.select('id', 'text').show(10)
```

    +------------------+--------------------+
    |                id|                text|
    +------------------+--------------------+
    |801657325836763136|RT @soompi: #SEVE...|
    |801657325677400064|RT @hobuing: also...|
    |801657307637678080|Retweeted The Seo...|
    |801657305628430336|RT @heochan_th: 1...|
    |801657297449586688|RT @heochan_th: 1...|
    |801657287697895424|go to seoul, sout...|
    |801657280760397824|RT @heochan_th: [...|
    |801657276788523008|RT @lesliehung: @...|
    |801657268177604608|RT @almightykeybe...|
    |801657258400616449|RT @always_yena: ...|
    +------------------+--------------------+
    only showing top 10 rows
    


## 문제: 월드컵 데이터 JSON의 URL에서 DataFrame 생성

url에서 데이터를 직접 읽어 DataFrame을 생성하는 방법은 지원되지 않고 있다.
requests 라이브러리를 사용하여, url에서 데이터를 가져오고, DataFrame을 생성하여 보자.

### URL에서 JSON 읽기

웹에서 읽는 JSON은 문자열이라는 점에 유의한다. 따라서 json()함수로 변환하는 것이 필요하다.
```python
[
  {
    "Competition": "World Cup",
    "Year": 1930,
    "Team": "Argentina",
    "Number": "",
    "Position": "GK",
    "FullName": "Ãngel Bossio",
    "Club": "Club AtlÃ©tico Talleres de Remedios de Escalada",
    "ClubCountry": "Argentina",
    "DateOfBirth": "1905-5-5",
    "IsCaptain": false
  },
  {
    "Competition": "World Cup",
    ...
    "IsCaptain": false
  },
  ...
]
```


```python
import requests
```


```python
r=requests.get("https://raw.githubusercontent.com/jokecamp/FootballData/master/World%20Cups/all-world-cup-players.json")
```

requuest.get() 함수는 Response를 반환한다.
이러한 Reponse에 포함된 텍스트를 사용하게 된다.
웹에서 읽은 텍스트는 어떤 자료구조를 가졌다고 하더라도 단순 문자열이다. 텍스트를 살펴보고, 구조가 있다면 알맞게 변환해주어야 한다.


```python
print("Type of Response: ", type(r))
```

    Type of Response:  <class 'requests.models.Response'>


앞서 읽은 텍스트를 json으로 읽는다.


```python
wc=r.json()
```


```python
print (type(wc), type(wc[0]))
```

    <class 'list'> <class 'dict'>


wc는 아래와 같이 ```[ {...}, {...}, ...]```, 즉 배열내에 dictionary가 컴마로 연결되어 있다.
이와 같이 파일에서 읽을 경우, 그 구조를 정확하게 이해하는 것이 중요하다.

```python
[{'Competition': 'World Cup',
  'Year': 1930,
  'Team': 'Argentina',
  'Number': '',
  'Position': 'GK',
  'FullName': 'Ãngel Bossio',
  'Club': 'Club AtlÃ©tico Talleres de Remedios de Escalada',
  'ClubCountry': 'Argentina',
  'DateOfBirth': '1905-5-5',
  'IsCaptain': False},
 {'Competition': 'World Cup',
  'Year': 1930,
  'Team': 'Argentina',
  'Number': '',
  'Position': 'GK',
  'FullName': 'Juan Botasso',
  'Club': 'Quilmes AtlÃ©tico Club',
  'ClubCountry': 'Argentina',
  'DateOfBirth': '1908-10-23',
  'IsCaptain': False},
 ... ]
 ```


```python
wc[0]
```




    {'Competition': 'World Cup',
     'Year': 1930,
     'Team': 'Argentina',
     'Number': '',
     'Position': 'GK',
     'FullName': 'Ãngel Bossio',
     'Club': 'Club AtlÃ©tico Talleres de Remedios de Escalada',
     'ClubCountry': 'Argentina',
     'DateOfBirth': '1905-5-5',
     'IsCaptain': False}



### DataFrame 생성

위 ```wc```는 JSON을 포함하는 리스트이다.
JSON은 key, value로 구성되어, 컬럼명을 key에서 가져올 수 있다.
이전 버전에서는 가능했으나, 현재는 Python dict로부터 DataFrame을 생성하려면 아래와 같이 pyspark.sql.Row를 활용해야 한다.


```python
_wcDf=spark.createDataFrame(wc)
```

    /home/jsl/.local/lib/python3.6/site-packages/pyspark/sql/session.py:378: UserWarning: inferring schema from dict is deprecated,please use pyspark.sql.Row instead
      warnings.warn("inferring schema from dict is deprecated,"


### Row를 이용해서 Dictionary에서 DataFrame 생성하기

앞서 살펴보았듯이, wc는 리스트이고 그 요소는 dictionary이다.
dictionary 구조를 풀어서 Row에 넘겨주어야 한다.
우선 Python에서 어떻게 인자를 풀어서 넘겨주는지 배워보자.

인자가 복수일 경우 **리스트, dictionary와 같은 구조**로 묶게 되는데, **함수 호출하면서 풀어야할 필요가 있을 경우에 별표 연산자** ```*args```, ```**args```를 사용한다.


#### 별표 1개 ```*```: **리스트**에서 인자 풀어내기

예를 들어, range() 함수는 시작과 끝을 인자로 가지는 함수이다.
시작과 끝이라는 2개의 인자를 함수에 넘겨주어 출력해보자.


```python
myList = [1, 6]
```


```python
list(range(1,6))
```




    [1, 2, 3, 4, 5]



다음과 같이 정수인자 자리에 리스트를 넘겨주면 타입오류가 발생한다.


```python
list(range(myList))  # TypeError: 'list' object cannot be interpreted as an integer
```


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    <ipython-input-62-7028d29cee9e> in <module>
    ----> 1 list(range(myList))  # TypeError: 'list' object cannot be interpreted as an integer
    

    TypeError: 'list' object cannot be interpreted as an integer


시작, 끝 변수가 따로 없을 경우, **리스트를 넘겨주고 여기서 풀어서 하나씩 인자를 사용할 경우** ```*```를 사용한다.


```python
list(range(*myList))  # unpack args and get 1 for the start and 6 for the end
```




    [1, 2, 3, 4, 5]



다음 예를 보자. f()는 복수의 인자를 받아서 하나씩 출력하는 함수이다.
f()를 호출할 때 여러 개를 넘겨주면, 1개만 받으므로 타입오류가 발생한다.


```python
def f(args):
    for i in args:
        print(i, end="~")

f(0, 1, 2, 3)
```


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    <ipython-input-63-4972646694d6> in <module>
          3         print(i, end="~")
          4 
    ----> 5 f(0, 1, 2, 3)
    

    TypeError: f() takes 1 positional argument but 4 were given


복수의 인자를 받게 될 것이라면 * 연산자를 사용한다. 그러면 복수의 인자를 풀어서 사용하게 된다.


```python
def f(*args):
    for i in args:
        print(i, end="~")

f(0, 1, 2, 3)
```

    0~1~2~3~

#### 별표 2 개 ```**```: **dictionary**에서 인자 풀어내기

dictionary를 넘겨주고 여기서 key, value를 하나씩 사용할 경우 ```**```를 사용한다.


```python
def printCapital(name, year):
    print(f"{name} in {year}")

myDict = {"name": "jsl", "year": 2020}
printCapital(**myDict)
```

    jsl in 2020


#### dictionary인자를 풀어서 Row에 넘겨주기

반복문으로 하나의 dictionary를 가져와 ```Row()```를 생성하고 있다.
별표 2개를 겹쳐 쓴 ```**```는 dictionary를 풀어서 Row 생성자에 넘겨주고 있다.


```python
from pyspark.sql import Row

wcDf = spark.createDataFrame(Row(**x) for x in wc)
```


```python
wcDf.printSchema()
```

    root
     |-- Competition: string (nullable = true)
     |-- Year: long (nullable = true)
     |-- Team: string (nullable = true)
     |-- Number: string (nullable = true)
     |-- Position: string (nullable = true)
     |-- FullName: string (nullable = true)
     |-- Club: string (nullable = true)
     |-- ClubCountry: string (nullable = true)
     |-- DateOfBirth: string (nullable = true)
     |-- IsCaptain: boolean (nullable = true)
    


데이터가 잘 읽혔는지, 일부를 출력해보자. 아래에서 보듯이, 아르헨티나 문자가 유니코드로 잘 출력이 되고 있다.


```python
wcDf.take(1)
```




    [Row(Competition='World Cup', Year=1930, Team='Argentina', Number='', Position='GK', FullName='Ãngel Bossio', Club='Club AtlÃ©tico Talleres de Remedios de Escalada', ClubCountry='Argentina', DateOfBirth='1905-5-5', IsCaptain=False)]



###  DataFrame의 shema 설정

wcRdd에서 DataFrame을 생성하려고 하면 아래와 같이 schema를 설정할 수 있다.
그러나 영어가 아닌 다른 나라의 유니코드 문자 또는 생년월일의 결측값 등으로 오류가 발생한다.
자동으로 인식한 Schema가 더 낫게 인식하고 있다.

```python
from pyspark.sql.types import *
wcSchema=StructType([
    StructField("Club", StringType(), True),
    StructField("ClubCountry", StringType(), True),
    StructField("Competition", StringType(), True),
    StructField("DateOfBirth", DateType(), True),
    StructField("FullName", StringType(), True),
    StructField("IsCaptain", BooleanType(), True),
    StructField("Number", IntegerType(), True),
    StructField("Position", StringType(), True),
    StructField("Team", StringType(), True),
    StructField("Year", IntegerType(), True)
])
```

### RDD 생성

RDD를 통해서 DataFrame 생성하기


```python
wcRdd=spark.sparkContext.parallelize(wc)
```


```python
wcRdd.take(1)
```




    [{'Competition': 'World Cup',
      'Year': 1930,
      'Team': 'Argentina',
      'Number': '',
      'Position': 'GK',
      'FullName': 'Ãngel Bossio',
      'Club': 'Club AtlÃ©tico Talleres de Remedios de Escalada',
      'ClubCountry': 'Argentina',
      'DateOfBirth': '1905-5-5',
      'IsCaptain': False}]




```python
wcDfFromRdd = spark.createDataFrame(wcRdd)
wcDfFromRdd.printSchema()
```

    root
     |-- Club: string (nullable = true)
     |-- ClubCountry: string (nullable = true)
     |-- Competition: string (nullable = true)
     |-- DateOfBirth: string (nullable = true)
     |-- FullName: string (nullable = true)
     |-- IsCaptain: boolean (nullable = true)
     |-- Number: string (nullable = true)
     |-- Position: string (nullable = true)
     |-- Team: string (nullable = true)
     |-- Year: long (nullable = true)
    


    /home/jsl/.local/lib/python3.6/site-packages/pyspark/sql/session.py:398: UserWarning: Using RDD of dict to inferSchema is deprecated. Use pyspark.sql.Row instead
      warnings.warn("Using RDD of dict to inferSchema is deprecated. "



```python
wcDfFromRdd.take(1)
```




    [Row(Club='Club AtlÃ©tico Talleres de Remedios de Escalada', ClubCountry='Argentina', Competition='World Cup', DateOfBirth='1905-5-5', FullName='Ãngel Bossio', IsCaptain=False, Number='', Position='GK', Team='Argentina', Year=1930)]



### 결측 값


* ```null```은 결측, 즉 "no value" 또는 "nothing"을 말한다.
* ```NaN```은 "Not a Number", 즉 수학에서 0.0/0.0과 같이 의미가 없는 연산의 결과를 말한다.

```IsCaptain``` 항목은 boolean 타입이라 isnan() 함수가 오류를 발생한다. ```'isnan(`IsCaptain`)' due to data type mismatch: argument 1 requires (double or float) type, however, '`IsCaptain`' is of boolean type.```
그래서 항목에서 제거하고 확인하기로 하자.


```python
cols = wcDf.columns
```


```python
cols.remove('IsCaptain')
```


```python
from pyspark.sql.functions import isnan, when, count, col
wcDf.select([count(when(isnan(c) | col(c).isNull(), c)).alias(c) for c in cols]).show()
```

    +-----------+----+----+------+--------+--------+----+-----------+-----------+
    |Competition|Year|Team|Number|Position|FullName|Club|ClubCountry|DateOfBirth|
    +-----------+----+----+------+--------+--------+----+-----------+-----------+
    |          0|   0|   0|     0|       0|       0|   0|          0|          0|
    +-----------+----+----+------+--------+--------+----+-----------+-----------+
    


### 형변환

스키마를 자동으로 유추한 형식을 보면,
DateOfBirth, Number 컬럼이 문자열로 인식되어 있어 만족스럽지 못하다.
컬럼 'DateOfBirth'는 'DoB'로 ```DateType()```, 컬럼 'Number'는 'NumberInt'로 "Integer" 형으로 설정해보자.

#### ```DateType``` 형변환

* Python ```datetime```을 사용한 DateType 컬럼 생성

Python ```datetime```은 날짜 문자열을 다음과 같은 년, 월, 일 형식으로 변환해준다.

표시 | 설명               |예
---|--------------------|--------------------
%d | 숫자로 표현한 2자리 일자 | 01, 02, ..., 31
%m | 숫자로 표현한 2자리 월  | 01, 02, ..., 12
%y | 숫자로 표현한 2자리 년수 (세기 불포함) | 00, 01, ..., 99
%Y | 숫자로 표현한 4자리 년수 (세기 포함)  | 0001, 0001, ..., 2020


```python
from datetime import datetime
print (datetime.strptime("11/25/1991", '%m/%d/%Y'))
```

    1991-11-25 00:00:00


datetime을 사용해서 일자를 DateType()으로 변환해보자.
이 때 사용자정의함수 udf()를 만들어 사용한다.


```python
from pyspark.sql.functions import udf
from pyspark.sql.types import DateType
toDate = udf(lambda x: datetime.strptime(x, '%m/%d/%Y'), DateType())
```

withColumn()은 컬럼을 추가하는 명령어이다. 방금 만든 udf 함수를 호출해서 형변환을 하고 있다.


```python
wcDf = wcDf.withColumn('date1', toDate(wcDf['DateOfBirth']))
```

그 결과를 출력하면 아래와 같이 ```ValueError```가 출력된다.

```python
wcDf.take(1)

ValueError: time data '1905-5-5' does not match format '%m/%d/%Y'
```

* pyspark ```to_date()``` 함수

이번에는 pyspark에서 제공하는 to_date() 함수를 사용하자.
방금 오류가 발생한 컬럼은 제거하고 해보자.


```python
wcDf = wcDf.drop('date1')
```

to_date() 함수는 string (StringType)을 date (DateType) 으로 형변환한다.


```python
from pyspark.sql.functions import to_date

_wcDfCasted=wcDf.withColumn('date2', to_date(wcDf['DateOfBirth'], 'yyyy-MM-dd'))
```

* pyspark ```cast()``` 함수

pyspark의 cast() 함수로 형변환을 해보자.
DateOfBirth와 더불어 Number 컬럼도 같이 Integer로 형변환을 해보자.


```python
from pyspark.sql.types import DateType

wcDfCasted = _wcDfCasted.withColumn('date3', _wcDfCasted['DateOfBirth'].cast(DateType()))
wcDfCasted = wcDfCasted.withColumn('NumberInt', wcDfCasted['Number'].cast("integer"))
```


```python
wcDfCasted.printSchema()
```

    root
     |-- Competition: string (nullable = true)
     |-- Year: long (nullable = true)
     |-- Team: string (nullable = true)
     |-- Number: string (nullable = true)
     |-- Position: string (nullable = true)
     |-- FullName: string (nullable = true)
     |-- Club: string (nullable = true)
     |-- ClubCountry: string (nullable = true)
     |-- DateOfBirth: string (nullable = true)
     |-- IsCaptain: boolean (nullable = true)
     |-- date2: date (nullable = true)
     |-- date3: date (nullable = true)
     |-- NumberInt: integer (nullable = true)
    


그대로 출력하면 오류가 발생할 수 있다.
```SparkUpgradeException: You may get a different result due to the upgrading of Spark 3.0```라는 오류가 발생하면
**```spark.sql.legacy.timeParserPolicy=LEGACY```**라고 설정을 변경해주어야 한다.

아래와 같이 to_data() 또는 cast() 함수를 사용하여 일자 형변환 결과를 올바르게 출력하고 있다.


```python
spark.sql("set spark.sql.legacy.timeParserPolicy=LEGACY")
wcDfCasted.take(1)
```




    [Row(Competition='World Cup', Year=1930, Team='Argentina', Number='', Position='GK', FullName='Ãngel Bossio', Club='Club AtlÃ©tico Talleres de Remedios de Escalada', ClubCountry='Argentina', DateOfBirth='1905-5-5', IsCaptain=False, date2=datetime.date(1905, 5, 5), date3=datetime.date(1905, 5, 5), NumberInt=None)]



### 코드 정리

지금까지 작성한 코드는 **상호대화형**으로 작성되었다.
배우거나 연습하기 위해 작성된 코드가 포함되었기 때문에 기능이 중복된 코드가 있을 수 있다.

프로그램을 하는 **목적**이 무엇인지를 생각하고, **필요한 코드**만 남겨두자.
URL에서 JSON 데이터를 읽어 DataFrame을 생성하고, 한 줄 출력하는 코드만 필요하다.
일괄실행하기 위해서는 spark를 생성하는 코드가 있어야 한다.
정리하면서 라이브러리는 앞 부분으로 위치시킨다.
주석도 코드를 이해하는데 도움이 되므로 넣어준다.
프로그램 절차에 맞추어 수정이 필요한 부분도 있다. 예를 들어 wcDfCasted는 wcDf로 변경하였다.

아래 프로그램에 main() 함수를 넣어서 정리하면 더욱 좋겠다.


```python
import os
import requests
from pyspark.sql import Row
from pyspark.sql.types import DateType

import pyspark
os.environ["PYSPARK_PYTHON"]="/usr/bin/python3"
os.environ["PYSPARK_DRIVER_PYTHON"]="/usr/bin/python3"

myConf=pyspark.SparkConf()
spark = pyspark.sql.SparkSession\
    .builder\
    .master("local")\
    .appName("myApp")\
    .config(conf=myConf)\
    .getOrCreate()

spark.sql("set spark.sql.legacy.timeParserPolicy=LEGACY")

# read url json
r=requests.get("https://raw.githubusercontent.com/jokecamp/FootballData/master/World%20Cups/all-world-cup-players.json")
wc=r.json()

# read dictionary into Row
wcDf = spark.createDataFrame(Row(**x) for x in wc)

# cast DoB string into date, Number string into integer
wcDfCasted = wcDf.withColumn('date3', wcDf['DateOfBirth'].cast(DateType()))
wcDfCasted = wcDfCasted.withColumn('NumberInt', wcDfCasted['Number'].cast("integer"))

wcDfCasted.take(1)
```




    [Row(Competition='World Cup', Year=1930, Team='Argentina', Number='', Position='GK', FullName='Ãngel Bossio', Club='Club AtlÃ©tico Talleres de Remedios de Escalada', ClubCountry='Argentina', DateOfBirth='1905-5-5', IsCaptain=False, date3=datetime.date(1905, 5, 5), NumberInt=None)]



### S.4.7 Parquet 파일 읽기, 쓰기

Parquet(파케이)는 나무조각을 붙여넣은 마룻바닥이라는 뜻으로, Apache Hadoop에서는 컬럼별로 저장하는 데이터 압축형식으로, DataFrame에서 이 파일을 쓰거나, 읽을 수 있다. 

```python
-rw-r--r-- 1 jsl jsl 522 10월  7 15:27 part-r-00000-0318688b-018f-4e55-858b-b4b78ac56532.snappy.parquet
-rw-r--r-- 1 jsl jsl   0 10월  7 15:27 _SUCCESS
```

앞서 사용했던 ```_myDf```를 parquet형식으로 저장해보자.


```python
_myDf.write.parquet(os.path.join("data","people.parquet"))
```

앞서 쓰여진 parquet으로부터 읽어서 출력할 수 있다.


```python
_pDf=spark.read.parquet(os.path.join("data","people.parquet"))
```


```python
_pDf.show()
```

    +----+-------+
    | age|   name|
    +----+-------+
    |null|Michael|
    |  30|   Andy|
    |  19| Justin|
    +----+-------+
    


## S.5 DataFrame API 사용해 보기

### 빈 DataFrame 생성

비어있는 DataFrame을 생성하려면, 빈 schema를 설정하고 emptyRdd()를 사용해준다.


```python
from pyspark.sql.types import StructType

schema = StructType([])
emptyDf = spark.createDataFrame(spark.sparkContext.emptyRDD(), schema)
```


```python
emptyDf.printSchema()
```

    root
    


### range

비어 있는 DataFrame보다는 일련의 수를 가진 DataFrame을 만들어 보자.
```range(start, end=None, setp=1, numPartitions=None)``` 함수는 Python range() 함수와 같이 정수를 생성하고, 컬럼명 id를 설정한다.


```python
spark.range(0, 10, 2).show()
```

    +---+
    | id|
    +---+
    |  0|
    |  2|
    |  4|
    |  6|
    |  8|
    +---+
    


함수를 사용하려면, DataFrame이 만들어져 있어야 하고, 그 결과 컬럼이 생성된다. DataFrame을 생성하지 않고, 함수를 실행해보기에 range() 함수는 유용하다. current_date()를 출력해보자.


```python
from pyspark.sql import functions as F
spark.range(1).select(F.current_date()).show()
```

    +--------------+
    |current_date()|
    +--------------+
    |    2020-10-14|
    +--------------+
    


컬럼명을 alias()로 정하면서, unix_timestamp()를 출력해보자.


```python
spark.range(1).select(F.unix_timestamp().alias("current_timestamp")).show()
```

    +-----------------+
    |current_timestamp|
    +-----------------+
    |       1602666060|
    +-----------------+
    


그 결과 값만을 추출하려면 조금 복잡하다. rdd로 변환하고, collect() 한후 인덱스를 사용한다.


```python
spark.range(1).select(F.unix_timestamp().alias("current_timestamp")).rdd.collect()[0]['current_timestamp']
```




    1602666226



### 컬럼 추가 withColumn, 컬럼 삭제 Drop

```withColumn()```은 열을 추가한다.
앞서 사용하였던 키, 몸무게 파일에서 DataFrame을 생성하고 컬럼을 생성해보자.


```python
tDf = spark\
    .read\
    .options(header='false', inferschema='true', delimiter='\t')\
    .csv(os.path.join('data', 'ds_spark_heightweight.txt'))
```

컬럼은 자동 생성이 되었으므로, ```'_c0', '_c1', '_c2'```이고 사용하기 불편하므로 의미있는 컬럼으로 변경하자.


```python
tDf.columns
```




    ['_c0', '_c1', '_c2']



기존에 있는 ```_1```행을 ```integer```로 형변환해서 ```id```행을 만들고, 기존의 ```_1```행을 삭제해보자.
```drop()```은 열을 삭제할 때 사용한다. 같은 방식으로 나머지도 변경해보자.
컬럼은 점연산자 tDf._c0 또는 인덱스 tDf['_c1'] 어느 방식이나 모두 가능하다.

cast() 함수에는 "integer" 또는 Spark의 데이터타입 IntegerType()이라고 적어도 된다.
또는 statement로 적어도 된다.
```python
from pyspark.sql import functions as F
tDf.withColumn("id", F.expr("CAST(id AS INTEGER)"))
```


```python
tDf = tDf.withColumn("id", tDf._c0.cast("integer"))
tDf = tDf.withColumn("height", tDf['_c1'].cast("double"))
tDf = tDf.withColumn("weight", tDf['_c2'].cast("double"))
```


```python
tDf.printSchema()
```

    root
     |-- _c0: integer (nullable = true)
     |-- _c1: double (nullable = true)
     |-- _c2: double (nullable = true)
     |-- id: integer (nullable = true)
     |-- height: double (nullable = true)
     |-- weight: double (nullable = true)
    


컬럼이 중복되었으므로, 삭제하자. 하나씩 또는 다음과 같이 한꺼번에 삭제할 수 있다.


```python
tDf = tDf.drop('_c0').drop('_c1').drop('_c2')
```


```python
tDf.printSchema()
```

    root
     |-- id: integer (nullable = true)
     |-- height: double (nullable = true)
     |-- weight: double (nullable = true)
    


형변환이 잘 되었고 컬럼명이 변경된 결과를 볼 수 있다.


```python
tDf.take(1)
```




    [Row(id=1, height=65.78, weight=112.99)]



### 사용자정의 함수 udf

UDF User Defined Functions는 사용자 정의함수로서 DataFrame의 withColumn() 함수와 같이 사용되어 새로운 컬럼을만드는 경우 유용하다.
보통 함수와 같이 함수명과 반환 값을 미리 정의해서 lambda 함수 또는 다른 함수를 사용할 수 있다.
**다른 함수를 직접 사용할 수 없고, udf()를 통해서 호출**해야 한다.
udf()는 **코드가 복잡한 경우에 함수를 분리해서 처리**하면 유용하겠다.


앞서 Pandas로 저장한 csv파일을 읽어서 DataFrame을 만들어 보자.
header를 있는 그대로 가져오고,
schema도 현재 데이터에서 추정하도록 설정한다.


```python
myDf = spark\
        .read.format('com.databricks.spark.csv')\
        .options(header='true', inferschema='true')\
        .load(os.path.join('data','myDf.csv'))
```

```header='true'```라고 설정을 해서, 컬럼명은 있는 그대로 year, name, height로 읽는다.
printSchema()를 출력해보면, year는 정수, name은 문자열, height는 정수로 정의되어 있다.


```python
myDf.printSchema()
```

    root
     |-- _c0: integer (nullable = true)
     |-- year: integer (nullable = true)
     |-- name: string (nullable = true)
     |-- height: integer (nullable = true)
    



```python
myDf.show()
```

    +---+----+-------+------+
    |_c0|year|   name|height|
    +---+----+-------+------+
    |  0|   1|kim, js|   170|
    |  1|   1|lee, sm|   175|
    |  2|   2|lim, yg|   180|
    |  3|   2|    lee|   170|
    +---+----+-------+------+
    


#### udf함수로 대문자 변환 withColumn

대문자로 변환하고 반환하는 함수를 만들어보자.


```python
def uppercase(s):
    return s.upper()
```


```python
uppercase("a")
```




    'A'



앞서 정의된 함수를 바로 호출하면 오류가 발생한다.


```python
myDf = myDf.withColumn("nameUpper", uppercase(myDf.name))
```


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    <ipython-input-36-85e35a83822c> in <module>
    ----> 1 myDf = myDf.withColumn("nameUpper", uppercase(myDf.name))
    

    <ipython-input-33-cc3ea1335364> in uppercase(s)
          1 def uppercase(s):
    ----> 2     return s.upper()
    

    TypeError: 'Column' object is not callable


udf()를 통해 함수를 호출해야 한다.
반환값의 타잎을 ```StringType()```으로 정의하고 있다는 점에 유의하자.


```python
from pyspark.sql.types import StringType
from pyspark.sql.functions import udf

upperUdf = udf(uppercase, StringType())
```

앞서 정의한 udf() 함수를 withColumn()과 같이 사용하자.


```python
myDf = myDf.withColumn("nameUpper", upperUdf(myDf['name']))
```


```python
myDf.show()
```

    +---+----+-------+------+---------+
    |_c0|year|   name|height|nameUpper|
    +---+----+-------+------+---------+
    |  0|   1|kim, js|   170|  KIM, JS|
    |  1|   1|lee, sm|   175|  LEE, SM|
    |  2|   2|lim, yg|   180|  LIM, YG|
    |  3|   2|    lee|   170|      LEE|
    +---+----+-------+------+---------+
    


#### udf함수로 Double변환 withColumn

udf() 함수에 lambda를 사용해도 된다.
**withColumn()에서 다른 함수를 호출하지 못하고, udf() 함수를 통해** 호출한다.

앞서 배운 ```withColumn()```에서 udf를 호출해서 ```height``` 컬럼을 ```long```에서 ```DoubleType()```으로 만들어 보자.



```python
from pyspark.sql.functions import udf
from pyspark.sql.types import DoubleType

toDoublefunc = udf(lambda x: float(x), DoubleType())
myDf = myDf.withColumn("heightD", toDoublefunc(myDf.height))
```

```dtypes```로 컬럼명과 데이터타입을 확인해보자.


```python
myDf.dtypes
```




    [('_c0', 'int'),
     ('year', 'int'),
     ('name', 'string'),
     ('height', 'int'),
     ('heightD', 'double')]



#### udf함수로 정수변환 withColumn

year의 문자열을 정수로 형변환 해보자


```python
#from pyspark.sql.functions import udf, struct
from pyspark.sql.functions import udf
from pyspark.sql.types import IntegerType

toint=udf(lambda x:int(x), IntegerType())
myDf=myDf.withColumn("yearI", toint(myDf['year']))
```


```python
myDf.printSchema()
```

    root
     |-- _c0: integer (nullable = true)
     |-- year: integer (nullable = true)
     |-- name: string (nullable = true)
     |-- height: integer (nullable = true)
     |-- heightD: double (nullable = true)
     |-- yearI: integer (nullable = true)
    



```python
myDf.show()
```

    +---+----+-------+------+-------+-----+
    |_c0|year|   name|height|heightD|yearI|
    +---+----+-------+------+-------+-----+
    |  0|   1|kim, js|   170|  170.0|    1|
    |  1|   1|lee, sm|   175|  175.0|    1|
    |  2|   2|lim, yg|   180|  180.0|    2|
    |  3|   2|    lee|   170|  170.0|    2|
    +---+----+-------+------+-------+-----+
    


yearI 컬럼명만 선택하여 조회해보자.


```python
myDf.select('yearI').show()
```

    +-----+
    |yearI|
    +-----+
    |    1|
    |    1|
    |    2|
    |    2|
    +-----+
    


#### udf함수로 조건에 따른 withColumn

udf()함수를 사용하여, 조건에 따라 컬럼을 생성해보자.
키를 175를 기준으로 이분화하고 반환값을 ```StringType()```으로 정의해보자.


```python
from pyspark.sql.types import StringType
from pyspark.sql.functions import udf

height_udf = udf(lambda height: "taller" if height >=175 else "shorter", StringType())
heightDf=myDf.withColumn("height>175", height_udf(myDf.heightD))
```


```python
heightDf.show()
```

    +---+----+-------+------+-------+-----+---------+----------+
    |_c0|year|   name|height|heightD|yearI|nameUpper|height>175|
    +---+----+-------+------+-------+-----+---------+----------+
    |  0|   1|kim, js|   170|  170.0|    1|  KIM, JS|   shorter|
    |  1|   1|lee, sm|   175|  175.0|    1|  LEE, SM|    taller|
    |  2|   2|lim, yg|   180|  180.0|    2|  LIM, YG|    taller|
    |  3|   2|    lee|   170|  170.0|    2|      LEE|   shorter|
    +---+----+-------+------+-------+-----+---------+----------+
    


### 컬럼명 변경 withColumnRenamed

앞서 ```withCoumun()``` 명령어로 열을 추가해 보았다. ```withColumnRenamed()``` 함수는 컬럼명을 변경한다. 컬럼명을 ```name```에서 ```full name```으로 변경해보자.


```python
tDf=tDf.withColumnRenamed('id','ID')
```


```python
tDf.show(3)
```

    +---+------+------+
    | ID|height|weight|
    +---+------+------+
    |  1| 65.78|112.99|
    |  2| 71.52|136.49|
    |  3|  69.4|153.03|
    +---+------+------+
    only showing top 3 rows
    


### 그래프

#### plot

Spark에는 그래프를 그리는 기능이 없다.
Python matplotlib을 이용해서 그래프를 표현한다.
2차원 ```plot()```을 하기 위해서는, x와 y축 값이 필요하다.
DataFrame -> RDD로 변환하고, map() 함수를 사용하여 배열로부터 weight, height를 분리해야 한다.


```python
_weightRdd=tDf.rdd.map(lambda fields:fields[1]).collect()
_heightRdd=tDf.rdd.map(lambda fields:fields[2]).collect()
```

x, y 값이 잘 추출되었는지 확인해보자.


```python
import numpy as np
print (np.array(_weightRdd)[:5])
print (np.array(_heightRdd)[:5])
```

    [65.78 71.52 69.4  68.22 67.79]
    [112.99 136.49 153.03 142.34 144.3 ]


앞서 추출한 height, weight RDD를 numpy array를 사용해서 2차원 x, y로 변환해준다.


```python
%matplotlib inline
import numpy as np
import matplotlib.pyplot as plt

plt.plot(np.array(_weightRdd), np.array(_heightRdd),'o')
plt.show()
```


![png](ds4_spark_df_files/ds4_spark_df_295_0.png)


#### boxplot, violinplot

하나의 컬럼만 선택해서 그래프를 그려보자.
* boxplot의 가운데 박스 IQR (Interquartile Range)은 4분위 값의 2, 3번째와 50% 값, 즉 25%~75%의 구간을 의미한다.
그리고 위 아래 가로선은 IQR 값의 1.5배되는 구역을 나타낸다. 값의 분포와 outlier를 확인하기 편리하다.
* violin plot 역시 boxplot과 유사한 기능이지만, 밀도를 보여주고 있다.


```python
height = tDf.select("height").toPandas()
```


```python
height.describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>height</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>50.00000</td>
    </tr>
    <tr>
      <th>mean</th>
      <td>68.05240</td>
    </tr>
    <tr>
      <th>std</th>
      <td>1.82398</td>
    </tr>
    <tr>
      <th>min</th>
      <td>63.48000</td>
    </tr>
    <tr>
      <th>25%</th>
      <td>66.94000</td>
    </tr>
    <tr>
      <th>50%</th>
      <td>67.86500</td>
    </tr>
    <tr>
      <th>75%</th>
      <td>69.18000</td>
    </tr>
    <tr>
      <th>max</th>
      <td>71.80000</td>
    </tr>
  </tbody>
</table>
</div>




```python
import matplotlib.pyplot as plt
import seaborn as sns

fig = plt.figure(figsize=(20, 10))
ax1 = fig.add_subplot(1, 2, 1)  # subplot location
ax1 = plt.boxplot(height)

ax2 = fig.add_subplot(1, 2, 2)
ax2 = sns.violinplot(data=height)
```


![png](ds4_spark_df_files/ds4_spark_df_299_0.png)


### aggregate functions

avg, collect_list, countDistinct, count, kurtosis, max, min, mean, skewness, stddev, sum, variance 등의 함수가 지원된다.

#### dictionary 형식

agg() 함수에 dictionary 형식으로 ```컬렴명: aggregate functions```으로 적어준다.


```python
tDf.agg({"height":"count"}).show()
```

    +-------------+
    |count(height)|
    +-------------+
    |           50|
    +-------------+
    



```python
tDf.agg({"height":"kurtosis"}).show()
```

    +--------------------+
    |    kurtosis(height)|
    +--------------------+
    |-0.00944222604387468|
    +--------------------+
    



```python
tDf.agg({"height":"avg"}).show()
```

    +-----------------+
    |      avg(height)|
    +-----------------+
    |68.05240000000002|
    +-----------------+
    


#### F 함수


```python
from pyspark.sql import functions as F
tDf.agg(F.min("height")).show()
```

    +-----------+
    |min(height)|
    +-----------+
    |      63.48|
    +-----------+
    


### 컬럼 조회 select

컬럼은 아래와 같이 ```select()```로 선택할 수 있다.
컬럼은 그 명칭을 사용하거나 ```myDf.name```, 인덱스 ```myDf['name']```로 선택할 수 있다.

컬럼 선택 | 예제 | 권고
-----|-----|-----
점 연산자로 컬럼을 선택 | myDf.name | N
인덱스로 컬럼을 선택 | myDf['name'] | Y

#### 컬럼명으로 직접 조회 못해

컬럼명만 아래와 같이 적어주어도 조회할 수 없다.


```python
myDf['name']
```




    Column<b'name'>



또는 컬럼 데이터를 조회하기 위해 컬럼에 show(), collect() 함수를 사용해서는 안된다. ```select()```를 사용해서 컬럼을 읽을 수 있다.


```python
myDf['name'].show()
```


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    <ipython-input-7-2fdc8d3e858e> in <module>
    ----> 1 myDf['name'].show()
    

    TypeError: 'Column' object is not callable


#### 컬럼 조회


```python
_name=myDf.select('name')
_name.show()
```

    +-------+
    |   name|
    +-------+
    |kim, js|
    |lee, sm|
    |lim, yg|
    |    lee|
    +-------+
    


여러 컬럼을 조회하기 위해서는 해당 컬럼을 넣어주면 된다.


```python
_name=myDf.select('name', 'height').show()
```

    +-------+------+
    |   name|height|
    +-------+------+
    |kim, js|   170|
    |lee, sm|   175|
    |lim, yg|   180|
    |    lee|   170|
    +-------+------+
    


또는 리스트를 넣어주어 여러 컬럼을 선택할 수도 있다.
**리스트를 풀어야 하므로, 앞서 배웠던 ```*``` 연산자**를 사용한다.


```python
cols = ['name', 'height']
myDf.select(*cols).show()
```

    +-------+------+
    |   name|height|
    +-------+------+
    |kim, js|   170|
    |lee, sm|   175|
    |lim, yg|   180|
    |    lee|   170|
    +-------+------+
    


#### 컬럼을 List로 변환

select() 함수는 Row()로 구성된 컬럼을 선택하게 된다.


```python
myDf.select('name').collect()
```




    [Row(name='kim, js'),
     Row(name='lee, sm'),
     Row(name='lim, yg'),
     Row(name='lee')]



List로 변환하려면, map() 함수로는 가능하지 않다.


```python
myDf.select('name').rdd.map(lambda x: x).collect()
```




    [Row(name='kim, js'),
     Row(name='lee, sm'),
     Row(name='lim, yg'),
     Row(name='lee')]



2차원 Row List를 제거하기 위해서 인덱스를 사용하면 된다.


```python
myDf.select('name').rdd.map(lambda x: x[0]).collect()
```




    ['kim, js', 'lee, sm', 'lim, yg', 'lee']



또는  flatMap()으로 2차원 구조 Row List를 1차원 List로 변환한다.


```python
myDf.select('name').rdd.flatMap(lambda x: x).collect()
```




    ['kim, js', 'lee, sm', 'lim, yg', 'lee']



#### select like

```%``` 연산자는 0 또는 그 이상의 문자를 의미한다.


```python
myDf.select("name", "height", myDf.name.like("%lee%")).show()
```

    +-------+------+---------------+
    |   name|height|name LIKE %lee%|
    +-------+------+---------------+
    |kim, js|   170|          false|
    |lee, sm|   175|           true|
    |lim, yg|   180|          false|
    |    lee|   170|           true|
    +-------+------+---------------+
    


#### select startswith


```python
myDf.select("name", "height", myDf.name.startswith("kim")).show()
```

    +-------+------+---------------------+
    |   name|height|startswith(name, kim)|
    +-------+------+---------------------+
    |kim, js|   170|                 true|
    |lee, sm|   175|                false|
    |lim, yg|   180|                false|
    |    lee|   170|                false|
    +-------+------+---------------------+
    


#### select endswith


```python
myDf.select("name", "height", myDf.name.endswith("lee")).show()
```

    +-------+------+-------------------+
    |   name|height|endswith(name, lee)|
    +-------+------+-------------------+
    |kim, js|   170|              false|
    |lee, sm|   175|              false|
    |lim, yg|   180|              false|
    |    lee|   170|               true|
    +-------+------+-------------------+
    


### alias

이름을 변경할 경우 alias() 함수를 사용할 수 있다.

DataFrame의 명칭을 변경해보자.


```python
myDf1 = myDf.alias("myDf1")
```

컬럼의 명칭을 변경해보자.
그러기 위해서는 우선 컬럼을 **```select()```** 함수로 골라낸 후, **```alias```**로 **컬럼명**을 정할 수 있다.
name 컬럼을 **```substr```**으로 1,3문자를 선택한다.


```python
myDf1.select(myDf1.name.substr(1,3).alias("short name")).show(3)
```

    +----------+
    |short name|
    +----------+
    |       kim|
    |       lee|
    |       lim|
    +----------+
    only showing top 3 rows
    


#### 행과 열을 선택 select, when, otherwise


```python
from pyspark.sql.functions import when
myDf.select("height", when(myDf.height < 175, 1).otherwise(0)).show()
```

    +------+------------------------------------------+
    |height|CASE WHEN (height < 175) THEN 1 ELSE 0 END|
    +------+------------------------------------------+
    |   170|                                         1|
    |   175|                                         0|
    |   180|                                         0|
    |   170|                                         1|
    +------+------------------------------------------+
    


alias 명령어로 컬럼을 변경해 줄 수 있다.


```python
from pyspark.sql.functions import when
myDf.select("height", (when(myDf.height < 175, 1).otherwise(0)).alias('<175')).show()
```

    +------+----+
    |height|<175|
    +------+----+
    |   170|   1|
    |   175|   0|
    |   180|   0|
    |   170|   1|
    +------+----+
    


또는 0, 1이 아닌 문자열로 생성할 수 있다.


```python
from pyspark.sql.functions import when
_myDf=myDf.select(when(myDf['heightD'] >175.0, ">175").otherwise("<175").alias("how tall"))
```


```python
_myDf.show()
```

    +--------+
    |how tall|
    +--------+
    |    <175|
    |    <175|
    |    >175|
    |    <175|
    +--------+
    


withColumn() 함수를 사용하면, DataFrame에 컬럼을 추가하게 된다.


```python
_myDf = myDf.withColumn('how tall', when(myDf['heightD'] >175.0, ">175").otherwise("<175"))
_myDf.show()
```

    +---+----+-------+------+-------+-----+---------+--------+
    |_c0|year|   name|height|heightD|yearI|nameUpper|how tall|
    +---+----+-------+------+-------+-----+---------+--------+
    |  0|   1|kim, js|   170|  170.0|    1|  KIM, JS|    <175|
    |  1|   1|lee, sm|   175|  175.0|    1|  LEE, SM|    <175|
    |  2|   2|lim, yg|   180|  180.0|    2|  LIM, YG|    >175|
    |  3|   2|    lee|   170|  170.0|    2|      LEE|    <175|
    +---+----+-------+------+-------+-----+---------+--------+
    


#### 행과 열을 선택  where, select


DataFrame은 관계형데이터베이스의 테이블과 매우 유사하다. **SQL 명령**을 사용하듯이 ```where()```, ```select()```, ```groupby()``` 함수를 사용할 수 있다.

* 행: ```where()```에 따라 컬럼의 **조건에 맞는 행을 선택**하고 (filter),
* 열: 앞서 배운 ```select()```로 열을 선택


```python
myDf.where(myDf['height'] < 175).show()
```

    +---+----+-------+------+---------+
    |_c0|year|   name|height|nameUpper|
    +---+----+-------+------+---------+
    |  0|   1|kim, js|   170|  KIM, JS|
    |  3|   2|    lee|   170|      LEE|
    +---+----+-------+------+---------+
    



```python
myDf.where(myDf['height'] < 175)\
    .select(myDf['name'], myDf['height']).show()
```

    +-------+------+
    |   name|height|
    +-------+------+
    |kim, js|   170|
    |    lee|   170|
    +-------+------+
    


### filter

```filter()```는 조건에 따라 데이터를 걸러낸다.
앞서 배웠던 ```where()```와 유사한 기능을 수행한다.


```python
myDf.filter(myDf['height'] > 175).show()
```

    +---+----+-------+------+-------+-----+---------+
    |_c0|year|   name|height|heightD|yearI|nameUpper|
    +---+----+-------+------+-------+-----+---------+
    |  2|   2|lim, yg|   180|  180.0|    2|  LIM, YG|
    +---+----+-------+------+-------+-----+---------+
    


### regexp_replace 컬럼의 내용 변경


```python
from pyspark.sql.functions import *

_heightDf = myDf.withColumn('nameNew', regexp_replace('name', 'lee', 'lim'))
_heightDf.show()
```

    +---+----+-------+------+---------+-------+
    |_c0|year|   name|height|nameUpper|nameNew|
    +---+----+-------+------+---------+-------+
    |  0|   1|kim, js|   170|  KIM, JS|kim, js|
    |  1|   1|lee, sm|   175|  LEE, SM|lim, sm|
    |  2|   2|lim, yg|   180|  LIM, YG|lim, yg|
    |  3|   2|    lee|   170|      LEE|    lim|
    +---+----+-------+------+---------+-------+
    


### groupBy

컬럼을 학년에 따라,
```groupBy()```하면 아래와 같이 데이터만 집단화하게 된다.
집단화하면 개수를 세거나, 합계를 내거나 어떤 통계량을 계산이 필요하다.


```python
myDf.groupby(myDf['year'])
```




    <pyspark.sql.group.GroupedData at 0x7f794caf5b38>



#### groupBy하고 max

컬럼을 기준으로 구분지어서 평균, 합계, 갯수, 최대, 최소 등을 구할 수 있다.
첫 컬럼 학년을 ```groupby()```해서 최대값 ```max()```를 구해보자.


```python
myDf.groupby(myDf['year']).max().show()
```

    +----+--------+---------+-----------+------------+----------+
    |year|max(_c0)|max(year)|max(height)|max(heightD)|max(yearI)|
    +----+--------+---------+-----------+------------+----------+
    |   1|       1|        1|        175|       175.0|         1|
    |   2|       3|        2|        180|       180.0|         2|
    +----+--------+---------+-----------+------------+----------+
    


#### groupBy, agg

```agg()```는 합계 함수를 계산할 수 있으며, 지원하는 함수는 ```avg, max, min, sum, count```이다.

```year``` 컬럼에 대해 ```agg()``` 함수로 계산할 수 있다.

dictionary 형식으로 key는 컬럼명, value는 합계 함수를 적어준다.
예를 들어 {"heightD":"avg"}에서 "heightD"는 컬럼명, "avg"는 합계함수이다.


```python
myDf.groupBy('year').agg({"heightD":"avg"}).show()
```

    +----+------------+
    |year|avg(heightD)|
    +----+------------+
    |   1|       172.5|
    |   2|       175.0|
    +----+------------+
    


#### groupBy 국가별 인원수

월드컵 데이터를 groupBy 해보자.


```python
wcDf.groupBy(wcDf.ClubCountry).count().show()
```

    +-----------+-----+
    |ClubCountry|count|
    +-----------+-----+
    |   England |    4|
    |   Paraguay|   93|
    |     Russia|   51|
    |        POL|   11|
    |        BRA|   27|
    |    Senegal|    1|
    |     Sweden|  154|
    |   Colombia|    1|
    |        FRA|  155|
    |        ALG|    8|
    |   England |    1|
    |       RUS |    1|
    |     Turkey|   65|
    |      Zaire|   22|
    |       Iraq|   22|
    |    Germany|  206|
    |        RSA|   16|
    |        ITA|  224|
    |        UKR|   38|
    |        GHA|    8|
    +-----------+-----+
    only showing top 20 rows
    


#### groupBy 국가별 포지션별 인원수


```python
wcDf.groupBy('ClubCountry').pivot('Position').count().show()
```

    +-----------+----+----+----+----+----+
    |ClubCountry|    |  DF|  FW|  GK|  MF|
    +-----------+----+----+----+----+----+
    |   England |null|null|   2|null|   2|
    |   Paraguay|null|  26|  37|  10|  20|
    |     Russia|null|  20|  11|   4|  16|
    |        POL|null|   2|   2|   3|   4|
    |        BRA|null|   7|   5|   4|  11|
    |    Senegal|null|null|null|   1|null|
    |     Sweden|null|  40|  47|  25|  42|
    |   Colombia|null|null|   1|null|null|
    |        ALG|null|   2|null|   6|null|
    |        FRA|null|  46|  41|  18|  50|
    |   England |null|null|null|null|   1|
    |       RUS |null|null|null|   1|null|
    |     Turkey|null|  20|  13|  12|  20|
    |      Zaire|null|   6|   5|   3|   8|
    |       Iraq|null|   6|   4|   3|   9|
    |    Germany|null|  64|  51|  16|  75|
    |        RSA|null|   5|   2|   3|   6|
    |        UKR|null|  13|   7|   4|  14|
    |        ITA|null|  74|  42|  19|  89|
    |        CMR|null|   1|   1|   1|null|
    +-----------+----+----+----+----+----+
    only showing top 20 rows
    


### F 함수

또는 아래와 같이 별도 ```pyspark.sql.functions```을 사용할 수 있다.
앞서 pyspark.sql.functions은 함수이므로, ```from pyspark.sql.functions import split``` 이렇게 한다.
또는 ```from pyspark.sql import functions as F```라고 한다.


```python
from pyspark.sql import functions as F

myDf.agg(F.min(myDf.heightD),F.max(myDf.heightD),F.avg(myDf.heightD),F.sum(myDf.heightD)).show()
```

    +------------+------------+------------+------------+
    |min(heightD)|max(heightD)|avg(heightD)|sum(heightD)|
    +------------+------------+------------+------------+
    |       170.0|       180.0|      173.75|       695.0|
    +------------+------------+------------+------------+
    


### 행 추가

행을 추가하려면, DataFrame을 서로 합치는 방법으로 가능하다.
추가할 행으로 DataFrame을 만들고, union() 함수로 합쳐야 한다.

createFrame() 함수에는 리스트를 넣어주어야 한다.


```python
toAppendDf = spark.createDataFrame([Row(4, 1, "choi, js", 177)])
```


```python
_myDf = myDf.union(toAppendDf)
```


```python
_myDf.show()
```

    +---+----+--------+------+
    |_c0|year|    name|height|
    +---+----+--------+------+
    |  0|   1| kim, js|   170|
    |  1|   1| lee, sm|   175|
    |  2|   2| lim, yg|   180|
    |  3|   2|     lee|   170|
    |  4|   1|choi, js|   177|
    +---+----+--------+------+
    


### partition

#### partition 개수


```python
myDf.rdd.getNumPartitions()
```




    1



#### repartition

repartition()은 partition의 개수를 늘리거나 줄이거나 재설정한다.


```python
_myDf = myDf.repartition(4)
print(_myDf.rdd.getNumPartitions())
```

    4


#### coalesce

coalesce()는 partition을 **줄일 때** 사용한다.
앞서 4개의 partition을 가진 ```_myDf```를 2로 줄여보자.


```python
_myDf2 = _myDf.coalesce(2)
print(_myDf2.rdd.getNumPartitions())
```

    2


### 통계 요약 describe

column이 연산가능한 데이터타잎인 경우, 요약 값을 볼 수 있다.


```python
myDf.describe().show()
```

    +-------+------------------+------------------+------------------+
    |summary|            height|           heightD|             yearI|
    +-------+------------------+------------------+------------------+
    |  count|                 4|                 4|                 4|
    |   mean|            173.75|            173.75|               1.5|
    | stddev|4.7871355387816905|4.7871355387816905|0.5773502691896257|
    |    min|               170|             170.0|                 1|
    |    max|               180|             180.0|                 2|
    +-------+------------------+------------------+------------------+
    


### 결측값

결측값을 채우는 함수이다.
* df.na.fill(0) 모든 컬럼의 na를 0으로 교체
* df.fillna( { 'c0':0, 'c1':0 } ) 컬럼 c0, c1의 na를 0으로 교체

결측값을 삭제할 수도 있다.
* df.na.drop(subset=["c0"])


```python
from pyspark.sql import functions as F
myDf.where(F.col("height").isNull())
```




    DataFrame[_c0: int, year: int, name: string, height: int, nameUpper: string]




```python
from pyspark.sql.functions import isnan, when, count, col
myDf.select([count(when(isnan(c), c)).alias(c) for c in myDf.columns]).show()
```

    +---+----+----+------+---------+
    |_c0|year|name|height|nameUpper|
    +---+----+----+------+---------+
    |  0|   0|   0|     0|        0|
    +---+----+----+------+---------+
    



```python
from pyspark.sql.functions import isnan, when, count, col
myDf.select([count(when(col(c).isNull(), c)).alias(c) for c in myDf.columns]).show()
```

    +---+----+----+------+---------+
    |_c0|year|name|height|nameUpper|
    +---+----+----+------+---------+
    |  0|   0|   0|     0|        0|
    +---+----+----+------+---------+
    


## 문제: 년별 분기별 대여건수

서울시 열린데이터 https://data.seoul.go.kr/ 에서 제공하는 ```서울특별시_공공자전거 일별 대여건수_(2018~2019.03).csv```를 분석해보자.
파일은 웹 검색을 해서 다운로드해서 사용하면 된다.
데이터는 일자별로, 대여건수이이고, 몇 줄만 출력해보면 다음과 같다.

|      date| count|
|----------|------|
|2018-01-01|  4950|
|2018-01-02|  7136|
|2018-01-03|  7156|
|2018-01-04|  7102|
|2018-01-05|  7705|

### 문제 1-1: 년도별 대여건수 합계
데이터는 2018, 2019년 15개월 간의 대여건수이다. 년도별로 대여건수의 합계를 계산해서 출력하자.

|year|sum(count)|
|----|----------|
|2018|  10124874|
|2019|   1871935|


### 문제 1-2: 년도별, 월별 대여건수 합계
년별, 월별로 대여건수를 계산하여 합계를 계산하여 출력한다.

### 문제 1-3: 년도별, 월별 대여건수 그래프
문제 1-2의 출력을 선 그래프로 그려보자.

### 데이터 읽기

서울시 열린데이터에서 데이터 ```서울특별시_공공자전거 일별 대여건수_(2018~2019.03).csv``` 를 다운로드 받아서 저장한다.
csv 형식으로 schema는 자동 인식하도록 읽는다.
일자는 ```timestamp```로 건수는 ```integer```로 인식되었다.


```python
_bicycle = spark.read.format('com.databricks.spark.csv')\
    .options(header='true', inferschema='true').load('data/seoulBicycleDailyCount_2018_201903.csv')
```


```python
_bicycle.printSchema()
```

    root
     |-- date: string (nullable = true)
     |--  count: integer (nullable = true)
    


전체 건수는 455건, 5건의 데이터만 읽어보자.


```python
_bicycle.count()
```




    455




```python
_bicycle.show(5)
```

    +----------+------+
    |      date| count|
    +----------+------+
    |2018-01-01|  4950|
    |2018-01-02|  7136|
    |2018-01-03|  7156|
    |2018-01-04|  7102|
    |2018-01-05|  7705|
    +----------+------+
    only showing top 5 rows
    


### 컬럼명 변경

앞서 보듯이 파일을 읽으면서 컬럼명이 인식되었는데 " count"가 맨 앞에 공백이 하나 있게 되어 변경해보자.
일단 붙여진 컬럼의 명칭을 변경하려면 ```withColumnRenamed()```를 연결하여 사용한다.


```python
bicycle=_bicycle\
    .withColumnRenamed("date", "Date")\
    .withColumnRenamed(" count", "Count")
```

### 컬럼 만들기: substr

```substr()``` 함수는 인자가 2개로서, 앞글자 '1'은 시작 '4'는 4글자를 의미한다.


```python
bicycle=bicycle.withColumn("year",bicycle.Date.substr(1, 4))
```


```python
bicycle.printSchema()
```

    root
     |-- Date: string (nullable = true)
     |-- Count: integer (nullable = true)
     |-- year: string (nullable = true)
    



```python
bicycle=bicycle.withColumn("month",bicycle.Date.substr(6, 2))
```


```python
bicycle.show(5)
```

    +----------+-----+----+-----+
    |      Date|Count|year|month|
    +----------+-----+----+-----+
    |2018-01-01| 4950|2018|   01|
    |2018-01-02| 7136|2018|   01|
    |2018-01-03| 7156|2018|   01|
    |2018-01-04| 7102|2018|   01|
    |2018-01-05| 7705|2018|   01|
    +----------+-----+----+-----+
    only showing top 5 rows
    


### 컬럼 만들기: F 함수

함수를 이용해 년, 월, 일 등을 추출할 수 있다.
먼저 앞서 생성된 column을 삭제하고 나서 해보자.
여러 컬럼을 삭제하기 위해서는 ```*```를 앞에 붙여 준다.
물론 하나씩 삭제할 수도 있고, 그러면 별표는 불필요하다.


```python
columns_to_drop = ['year','month']
df = bicycle.drop(*columns_to_drop)
```

year, month 컬럼이 삭제되고 Date, Count 컬럼만 남겨졌다.


```python
df.printSchema()
```

    root
     |-- Date: string (nullable = true)
     |-- Count: integer (nullable = true)
    


pyspark.sql.functions의 year(), month() 함수를 사용하여 년, 월을 추출하자.


```python
import pyspark.sql.functions as F
bicycle = bicycle\
    .withColumn('year', F.year('date'))\
    .withColumn('month', F.month('date'))
```


```python
bicycle.printSchema()
```

    root
     |-- Date: string (nullable = true)
     |-- Count: integer (nullable = true)
     |-- year: integer (nullable = true)
     |-- month: integer (nullable = true)
    


앞서 substr() 결과와 비교해보자. year, month가 올바르게 추출되었다.


```python
bicycle.show(5)
```

    +----------+-----+----+-----+
    |      Date|Count|year|month|
    +----------+-----+----+-----+
    |2018-01-01| 4950|2018|    1|
    |2018-01-02| 7136|2018|    1|
    |2018-01-03| 7156|2018|    1|
    |2018-01-04| 7102|2018|    1|
    |2018-01-05| 7705|2018|    1|
    +----------+-----+----+-----+
    only showing top 5 rows
    


show()는 앞 부분 보여주고 있다. 월이 1만 보여서, 다른 월의 결과를 보고 싶다면 filter()해주면 된다.


```python
bicycle.filter(bicycle.month == 2).show(3)
```

    +----------+-----+----+-----+
    |      Date|Count|year|month|
    +----------+-----+----+-----+
    |2018-02-01| 5821|2018|    2|
    |2018-02-02| 6557|2018|    2|
    |2018-02-03| 3499|2018|    2|
    +----------+-----+----+-----+
    only showing top 3 rows
    


### 분기

1월 2월 3월은 1분기, 4~6은 2분기, 7~9는 3분기, 10~12는 4분기로 구분한다.


```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType
def classifyQuarter(s):
    q=""
    if 1<=s and s< 4:
        q="Q1"
    elif 4<=s and s<7:
        q="Q2"
    elif 7<=s and s<10:
        q="Q3"
    elif 10<=s and s<=12:
        q="Q4"
    else:
        q="no"
    return q
```


```python
quarter_udf = udf(classifyQuarter, StringType())
```


```python
bicycle=bicycle.withColumn("quarter", quarter_udf(bicycle.month))
```

잘 분류되었는지 건수를 확인해보자.


```python
bicycle.groupBy('quarter').count().show()
```

    +-------+-----+
    |quarter|count|
    +-------+-----+
    |     Q2|   91|
    |     Q1|  180|
    |     Q3|   92|
    |     Q4|   92|
    +-------+-----+
    



```python
bicycle.show(5)
```

    +----------+-----+----+-----+-------+
    |      Date|Count|year|month|quarter|
    +----------+-----+----+-----+-------+
    |2018-01-01| 4950|2018|    1|     Q1|
    |2018-01-02| 7136|2018|    1|     Q1|
    |2018-01-03| 7156|2018|    1|     Q1|
    |2018-01-04| 7102|2018|    1|     Q1|
    |2018-01-05| 7705|2018|    1|     Q1|
    +----------+-----+----+-----+-------+
    only showing top 5 rows
    


### 년도별 대여건수 합계


```python
bicycle.groupBy('year').agg({"count":"sum"}).show()
```

    +----+----------+
    |year|sum(count)|
    +----+----------+
    |2018|  10124874|
    |2019|   1871935|
    +----+----------+
    


### 분기별 대여건수 합계


```python
bicycle.groupBy('quarter').agg({"count":"sum"}).show()
```

    +-------+----------+
    |quarter|sum(count)|
    +-------+----------+
    |     Q2|   2860617|
    |     Q1|   2667704|
    |     Q3|   3585513|
    |     Q4|   2882975|
    +-------+----------+
    



```python
bicycle.groupBy('quarter').agg({"count":"avg"}).show()
```

    +-------+------------------+
    |quarter|        avg(count)|
    +-------+------------------+
    |     Q2|31435.351648351647|
    |     Q1|14820.577777777778|
    |     Q3|38972.967391304344|
    |     Q4|31336.684782608696|
    +-------+------------------+
    


### 년도별, 월별 (분기별) 대여건수 합계


```python
bicycle.groupBy('year').pivot('month').agg({"count":"sum"}).show()
```

    +----+------+------+------+------+------+-------+-------+-------+-------+-------+------+------+
    |year|     1|     2|     3|     4|     5|      6|      7|      8|      9|     10|    11|    12|
    +----+------+------+------+------+------+-------+-------+-------+-------+-------+------+------+
    |2018|164367|168741|462661|687885|965609|1207123|1100015|1037505|1447993|1420621|961532|500822|
    |2019|495573|471543|904819|  null|  null|   null|   null|   null|   null|   null|  null|  null|
    +----+------+------+------+------+------+-------+-------+-------+-------+-------+------+------+
    



```python
bicycle.groupBy('year').pivot('quarter').agg({"count":"sum"}).show()
```

    +----+-------+-------+-------+-------+
    |year|     Q1|     Q2|     Q3|     Q4|
    +----+-------+-------+-------+-------+
    |2018| 795769|2860617|3585513|2882975|
    |2019|1871935|   null|   null|   null|
    +----+-------+-------+-------+-------+
    


### Pandas pivot


```python
import pandas as pd
import numpy as np
bicycleP = bicycle.toPandas()
```

Pandas의 info() 함수는 DataFrame의 컬럼, 데이터타입 dtypes를 출력한다.


```python
bicycleP.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 455 entries, 0 to 454
    Data columns (total 5 columns):
     #   Column   Non-Null Count  Dtype 
    ---  ------   --------------  ----- 
     0   Date     455 non-null    object
     1   Count    455 non-null    int32 
     2   year     455 non-null    int32 
     3   month    455 non-null    int32 
     4   quarter  455 non-null    object
    dtypes: int32(3), object(2)
    memory usage: 12.6+ KB


* 년도별 대여건수 합계


```python
#bicycleP.groupby('year').aggregate({'Count':np.sum})
bicycleP.groupby('year').aggregate({'Count':'sum'})
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Count</th>
    </tr>
    <tr>
      <th>year</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2018</th>
      <td>10124874</td>
    </tr>
    <tr>
      <th>2019</th>
      <td>1871935</td>
    </tr>
  </tbody>
</table>
</div>



* 년도별, 월별 대여건수 합계

index는 행, columns는 열 데이터를 정의한다.


```python
pd.pivot_table(bicycleP, values = 'Count', index = ['year'], columns = ['month'], aggfunc= 'sum')
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>month</th>
      <th>1</th>
      <th>2</th>
      <th>3</th>
      <th>4</th>
      <th>5</th>
      <th>6</th>
      <th>7</th>
      <th>8</th>
      <th>9</th>
      <th>10</th>
      <th>11</th>
      <th>12</th>
    </tr>
    <tr>
      <th>year</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2018</th>
      <td>164367.0</td>
      <td>168741.0</td>
      <td>462661.0</td>
      <td>687885.0</td>
      <td>965609.0</td>
      <td>1207123.0</td>
      <td>1100015.0</td>
      <td>1037505.0</td>
      <td>1447993.0</td>
      <td>1420621.0</td>
      <td>961532.0</td>
      <td>500822.0</td>
    </tr>
    <tr>
      <th>2019</th>
      <td>495573.0</td>
      <td>471543.0</td>
      <td>904819.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



2018년만 선택해서 년도별 x 분기별 대여건수를 출력해보자.


```python
bicycleP2018=bicycleP[bicycleP['year']==2018]
```


```python
bicycleP2018byQ = pd.pivot_table(bicycleP2018, values = 'Count', index = ['year'], columns = ['quarter'], aggfunc= 'sum')
```

iloc은 정수 인덱스의 위치, ```[:, 0:4]```는 즉 모든 행, 컬럼은 0~4까지 (4는 제외) 데이터를 조회한다.


```python
bicycleP2018byQ.iloc[:,0:4]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>quarter</th>
      <th>Q1</th>
      <th>Q2</th>
      <th>Q3</th>
      <th>Q4</th>
    </tr>
    <tr>
      <th>year</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2018</th>
      <td>795769</td>
      <td>2860617</td>
      <td>3585513</td>
      <td>2882975</td>
    </tr>
  </tbody>
</table>
</div>



### 년별 월별 대여건수 그래프

앞장 RDD에서 만들어진 단어빈도는 리스트에 저장되었다. 따라서 리스트에서 데이터를 추출하여 그래프를 그렸다.
groupBy에서 생성된 월별 대여건수는 pandas로 변환하여 그려보자.

#### 년별 월별 대여건수 생성


```python
sumMonthly=bicycle.groupBy('year').pivot('month').agg({"count":"sum"})
```


```python
pdf=sumMonthly.toPandas()
```


```python
pdf.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>year</th>
      <th>1</th>
      <th>2</th>
      <th>3</th>
      <th>4</th>
      <th>5</th>
      <th>6</th>
      <th>7</th>
      <th>8</th>
      <th>9</th>
      <th>10</th>
      <th>11</th>
      <th>12</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2018</td>
      <td>164367</td>
      <td>168741</td>
      <td>462661</td>
      <td>687885.0</td>
      <td>965609.0</td>
      <td>1207123.0</td>
      <td>1100015.0</td>
      <td>1037505.0</td>
      <td>1447993.0</td>
      <td>1420621.0</td>
      <td>961532.0</td>
      <td>500822.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2019</td>
      <td>495573</td>
      <td>471543</td>
      <td>904819</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



#### 

위 데이터에서 'year' 컬럼이 없어야 그래프를 그릴 수 있다.
* drop() 명령어에 삭제할 컬럼명 'year'와 1을 적어준다.
0은 행 (index), 1은 컬럼을 삭제한다는 의미이다.
* transpose() 함수를 통해 열로 변환하여 (행 데이터는 plot을 할 수 없다), 그래프를 그린다.


```python
my=pdf.drop('year', 1).transpose()
```

위 pdf를 변환한 my를 출력하면, 명령어가 적용되어 year 컬럼이 삭제되고, transpose되어 있다.


```python
my
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>0</th>
      <th>1</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>164367.0</td>
      <td>495573.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>168741.0</td>
      <td>471543.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>462661.0</td>
      <td>904819.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>687885.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>5</th>
      <td>965609.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>6</th>
      <td>1207123.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>7</th>
      <td>1100015.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>8</th>
      <td>1037505.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>9</th>
      <td>1447993.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>10</th>
      <td>1420621.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>11</th>
      <td>961532.0</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>12</th>
      <td>500822.0</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



```plot()``` 함수는 컬럼을 적지 않으면 모든 컬럼에 대해서 plot한다.


```python
my.columns=[2018, 2019]
```


```python
my.plot(kind='line')
```




    <AxesSubplot:>




![png](ds4_spark_df_files/ds4_spark_df_448_1.png)


## S.6 Spark SQL



관계형 데이터베이스 RDB에서 사용하는 Sql을 사용하여 DataFrame으로부터 데이터를 조회할 수 있다. DataFrame과 달리, RDD는 비구조적인 경우에 사용하므로 테이블로 변환한 후 Sql을 사용하게 된다.

* Spark SQL 구성

구분 | 설명
-----|-----
Language API | Python, Java, Scala, Hive QL API를 제공
Schema RDD | RDD에 Schema를 적용해 임시 테이블로 변환한다.<br>createOrReplaceTempView<br>createGlobalTempView
Data Sources | 다양한 형식 지원 - HDFS, Cassandra, HBase, RDB

앞서 만들어 놓은 World Cup 데이터를 사용한다.


```python
wcDf.printSchema()
```

    root
     |-- Competition: string (nullable = true)
     |-- Year: long (nullable = true)
     |-- Team: string (nullable = true)
     |-- Number: string (nullable = true)
     |-- Position: string (nullable = true)
     |-- FullName: string (nullable = true)
     |-- Club: string (nullable = true)
     |-- ClubCountry: string (nullable = true)
     |-- DateOfBirth: string (nullable = true)
     |-- IsCaptain: boolean (nullable = true)
    


이제 임시 테이블 ```wc```를 만들고, Sql문으로 데이터를 조회해보자.


```python
wcDf.createOrReplaceTempView("wc")
spark.sql("select Club,Team,Year from wc").show(1)
```

    +--------------------+---------+----+
    |                Club|     Team|Year|
    +--------------------+---------+----+
    |Club AtlÃ©tico Ta...|Argentina|1930|
    +--------------------+---------+----+
    only showing top 1 row
    



```python
wcPlayers=spark.sql("select FullName,Club,Team,Year from wc")
wcPlayers.show(1)
```

    +------------+--------------------+---------+----+
    |    FullName|                Club|     Team|Year|
    +------------+--------------------+---------+----+
    |Ãngel Bossio|Club AtlÃ©tico Ta...|Argentina|1930|
    +------------+--------------------+---------+----+
    only showing top 1 row
    



```python
spark.catalog.listTables()
```




    [Table(name='wc', database=None, description=None, tableType='TEMPORARY', isTemporary=True)]



```wcPlayers```를 RDD로 변환해서 이름만 출력해 보자.


```python
namesRdd=wcPlayers.rdd.map(lambda x: "Full name: "+x[0])
for e in namesRdd.take(5):
    print (e)
```

    Full name: Ãngel Bossio
    Full name: Juan Botasso
    Full name: Roberto Cherro
    Full name: Alberto Chividini
    Full name: 


#### sql.functions and join

리스트에 포함되어 있는 과일에 고유번호를 할당해 보자.


```python
bucketDf=spark.createDataFrame([[1,["orange", "apple", "pineapple"]],
                                [2,["watermelon","apple","bananas"]]],
                               ["bucketId","items"])
```

```truncate```는 행의 값을 잘라내지 않고 출력한다.
```show(bucketDf.count(), truncate=False)```는 모든 행을 완전하게 출력한다.


```python
bucketDf.show(bucketDf.count(), truncate=False)
```

    +--------+----------------------------+
    |bucketId|items                       |
    +--------+----------------------------+
    |1       |[orange, apple, pineapple]  |
    |2       |[watermelon, apple, bananas]|
    +--------+----------------------------+
    


* explode

컬럼에 List 또는 배열이 포함된 경우 ```explode()``` 함수는 이를 flat해서 새로운 컬럼을 생성하게 된다.



```python
from pyspark.sql.functions import explode
bDf=bucketDf.select(bucketDf.bucketId, explode(bucketDf.items).alias('item'))
```


```python
bDf.show()
```

    +--------+----------+
    |bucketId|      item|
    +--------+----------+
    |       1|    orange|
    |       1|     apple|
    |       1| pineapple|
    |       2|watermelon|
    |       2|     apple|
    |       2|   bananas|
    +--------+----------+
    


또 다른 DataFrame을 생성해보자. 나중에 앞의 DataFrame과 join하게 된다.


```python
fDf=spark.createDataFrame([["orange", "F1"],
                            ["", "F2"],
                            ["pineapple","F3"],
                            ["watermelon","F4"],
                            ["bananas","F5"]],
                            ["item","itemId"])
```


```python
fDf.show()
```

    +----------+------+
    |      item|itemId|
    +----------+------+
    |    orange|    F1|
    |          |    F2|
    | pineapple|    F3|
    |watermelon|    F4|
    |   bananas|    F5|
    +----------+------+
    


* join

join은 ```inner, cross, outer, full, full_outer, left, left_outer, right, right_outer, left_semi, left_anti``` 여러 종류가 있다. ```inner```기준으로 item이 일치하지 않는 것은 제외하게 된다.


```python
joinDf=fDf.join(bDf, fDf.item==bDf.item, "inner")
```


```python
joinDf.select(fDf.itemId,fDf.item,bDf.bucketId).show()
```

    +------+----------+--------+
    |itemId|      item|bucketId|
    +------+----------+--------+
    |    F5|   bananas|       2|
    |    F1|    orange|       1|
    |    F3| pineapple|       1|
    |    F4|watermelon|       2|
    +------+----------+--------+
    


## 문제 S-1: 네트워크에 불법적으로 침입하는 사용자의 분석

### 문제

네트워크에 불법적으로 침입하는 시도는 허용되어서는 안된다.
1998년 MIT Lincoln Labs에서 DARPA Intrusion Detection Evaluation Program을 연구하였다.
이 데이터의 일부가 1999년 KDD로 만들어져 배포되고 있다.
https://kdd.ics.uci.edu/databases/kddcup99/kddcup99.html

### 해결

마지막 행에 attack의 유형이 구분되어 있다. 네트워크 침입 유형의 특징을 분석해 보자.
탐지예방 모델을 구축할 수 있다.


KDD데이터는 41 항목으로 구성되어 있다.

```python
연결(초) | duration: continuous.
프로토콜 (tcp,udp,etc) | protocol_type: symbolic.
서비스 (http,telnet, etc) | service: symbolic.
flag: symbolic.
src_bytes: continuous.
dst_bytes: continuous.
land: symbolic.
wrong_fragment: continuous.
urgent: continuous.
hot: continuous.
num_failed_logins: continuous.
logged_in: symbolic.
num_compromised: continuous.
root_shell: continuous.
su_attempted: continuous.
num_root: continuous.
num_file_creations: continuous.
num_shells: continuous.
num_access_files: continuous.
num_outbound_cmds: continuous.
is_host_login: symbolic.
is_guest_login: symbolic.
count: continuous.
srv_count: continuous.
serror_rate: continuous.
srv_serror_rate: continuous.
rerror_rate: continuous.
srv_rerror_rate: continuous.
same_srv_rate: continuous.
diff_srv_rate: continuous.
srv_diff_host_rate: continuous.
dst_host_count: continuous.
dst_host_srv_count: continuous.
dst_host_same_srv_rate: continuous.
dst_host_diff_srv_rate: continuous|.
dst_host_same_src_port_rate: continuous.
dst_host_srv_diff_host_rate: continuous.
dst_host_serror_rate: continuous.
dst_host_srv_serror_rate: continuous.
dst_host_rerror_rate: continuous.
dst_host_srv_rerror_rate: continuous.
```

### 파일 내려받기

KDD 파일은 **gz** 압축되어 있다. 파일 확장자 'gz'은 'gzip'이라는 압축 도구에서 생성된 파일이다. 지금은 WinZip에서 읽을 수 있다.


```python
import os
_url = 'http://kdd.ics.uci.edu/databases/kddcup99/kddcup.data_10_percent.gz'
_fname = os.path.join(os.getcwd(),'data','kddcup.data_10_percent.gz')
```

파일이 로컬 디렉토리 ```data```에 존재하면, 즉 이미 내려받았으므로 또 내려받지 않는다. 그렇지 않을 경우에만 ```urlretrieve()``` 함수로 내려받는다. 오류가 발생하면, (1) 파일이 없거나, (2) 파일을 모두 내려 받지 않았거나, (3) 파일이 깨져있을 수 있다. 내려받은 디렉토리로 가서 그 파일이 존재하는지, winzip같은 유틸리티로 해당 gz을 풀어보고 확인하든지, 적당한 에디터로 해당 파일에 내용이 있는지 확인해야 한다.


```python
from urllib.request import urlretrieve

if(not os.path.exists(_fname)):
    print ("{} data does not exist! retrieving..".format(_fname))
    _f=urlretrieve(_url,_fname)
```

### RDD 생성

**RDD**는 gz와 같은 **압축파일에서 데이터를 읽어서** 생성할 수 있다.

반면, DataFrame은 구조schema를 정의해야 하기 때문에 쉽지 않다. 여기서는 **오류**가 발생한다.
따라서 RDD를 생성하고 난 후, 그로부터 DataFrame을 생성하고, Sql을 사용한다.

```textFile()``` 함수로 RDD를 생성한다. ```count()```는 행의 수를 돌려주는 action 함수이다. action 함수는 바로 실행되므로 시간이 좀 걸린다.


```python
_rdd = spark.sparkContext.textFile(_fname)
```


```python
_rdd.count()
```




    494021




```python
_rdd.take(1)
```




    ['0,tcp,http,SF,181,5450,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,8,8,0.00,0.00,0.00,0.00,1.00,0.00,0.00,9,9,1.00,0.00,0.11,0.00,0.00,0.00,0.00,0.00,normal.']



```map()``` 함수를 사용하여 csv 형식으로 구성된 파일을 컴마(,)로 분리한다.


```python
_allRdd=_rdd.map(lambda x: x.split(','))
```


```python
_allRdd.take(1)
```




    [['0',
      'tcp',
      'http',
      'SF',
      '181',
      '5450',
      '0',
      '0',
      '0',
      '0',
      '0',
      '1',
      '0',
      '0',
      '0',
      '0',
      '0',
      '0',
      '0',
      '0',
      '0',
      '0',
      '8',
      '8',
      '0.00',
      '0.00',
      '0.00',
      '0.00',
      '1.00',
      '0.00',
      '0.00',
      '9',
      '9',
      '1.00',
      '0.00',
      '0.11',
      '0.00',
      '0.00',
      '0.00',
      '0.00',
      '0.00',
      'normal.']]



### 정상, 공격 건수

데이터가 ```normal```인 경우와 아닌 경우로 구분하자.
```filter()```는 41번째 행을 조건에 따라 데이터를 구분한다.
```count()``` 함수로 건수를 계산하면 'normal' 97,278, 'attack'은 396,743 건이다.

침입구분 | 건수
-------|-------
normal | 97278
attack | 396743
전체 | 494021


```python
_normalRdd=_allRdd.filter(lambda x: x[41]=="normal.")
_attackRdd=_allRdd.filter(lambda x: x[41]!="normal.")
```


```python
_normalRdd.count()
```




    97278




```python
_attackRdd.count()
```




    396743



### attack별 건수

**attack 종류**는 41번째 열에 구분되어 있다. 총 494,021건을 정상 'noraml'과 나머지는 'attack'으로 구분한다.
'attack'은 크게 4종류로 나눈다. DOS는 서비스 거부, R2L 원격침입, U2R은 루트권한침입, probing은 탐지이다.

attack 4종류 | 설명 | 41번째 열
-----|-----|-----
DOS | denial-of-service, e.g. syn flood | back, land, neptune, pod, smurf, teardrop
R2L | unauthorized access from a remote machine | ftp_write, guess_passwd, imap, multihop, phf, spy, warezclient, warezmaster
U2R | unauthorized access to local superuser (root) privileges | buffer_overflow, loadmodule, perl, rootkit
probing | surveillance and other probing | ipsweep, nmap, portsweep, satan

열41에 대해 건수를 세어보자.
```reduceByKey()```는 인자로 '함수'가 필요. 키별로 '함수를 사용해서' 계산한다.


```python
_41 = _allRdd.map(lambda x: (x[41], 1))
_41.reduceByKey(lambda x,y: x+y).collect()
```




    [('normal.', 97278),
     ('buffer_overflow.', 30),
     ('loadmodule.', 9),
     ('perl.', 3),
     ('neptune.', 107201),
     ('smurf.', 280790),
     ('guess_passwd.', 53),
     ('pod.', 264),
     ('teardrop.', 979),
     ('portsweep.', 1040),
     ('ipsweep.', 1247),
     ('land.', 21),
     ('ftp_write.', 8),
     ('back.', 2203),
     ('imap.', 12),
     ('satan.', 1589),
     ('phf.', 4),
     ('nmap.', 231),
     ('multihop.', 7),
     ('warezmaster.', 20),
     ('warezclient.', 1020),
     ('spy.', 2),
     ('rootkit.', 10)]



```groupByKey()```는 키별로 group한다. 위 ```reduceByKey()```와 달리 ```mapValues()```를 사용해 값을 별도로 계산한다는 점에 유의하자.


```python
_41 = _allRdd.map(lambda x: (x[41], 1))
def f(x): return len(x)
_41.groupByKey().mapValues(f).collect()
```




    [(u'guess_passwd.', 53),
     (u'nmap.', 231),
     (u'warezmaster.', 20),
     (u'rootkit.', 10),
     (u'warezclient.', 1020),
     (u'smurf.', 280790),
     (u'pod.', 264),
     (u'neptune.', 107201),
     (u'normal.', 97278),
     (u'spy.', 2),
     (u'ftp_write.', 8),
     (u'phf.', 4),
     (u'portsweep.', 1040),
     (u'teardrop.', 979),
     (u'buffer_overflow.', 30),
     (u'land.', 21),
     (u'imap.', 12),
     (u'loadmodule.', 9),
     (u'perl.', 3),
     (u'multihop.', 7),
     (u'back.', 2203),
     (u'ipsweep.', 1247),
     (u'satan.', 1589)]



### Dataframe 생성

열 0, 1, 2, 3, 4, 5, 41을 선별하여 스키마를 정해서 RDD를 생성한다.


```python
from pyspark.sql import Row

_csv = _rdd.map(lambda l: l.split(","))
_csvRdd = _csv.map(lambda p: 
    Row(
        duration=int(p[0]), 
        protocol=p[1],
        service=p[2],
        flag=p[3],
        src_bytes=int(p[4]),
        dst_bytes=int(p[5]),
        attack=p[41]
    )
)
```

RDD를 Dataframe으로 변환한다.



```python
_df=spark.createDataFrame(_csvRdd)

```


```python
_df.printSchema()
_df.show(5)
```

    root
     |-- attack: string (nullable = true)
     |-- dst_bytes: long (nullable = true)
     |-- duration: long (nullable = true)
     |-- flag: string (nullable = true)
     |-- protocol: string (nullable = true)
     |-- service: string (nullable = true)
     |-- src_bytes: long (nullable = true)
    
    +-------+---------+--------+----+--------+-------+---------+
    | attack|dst_bytes|duration|flag|protocol|service|src_bytes|
    +-------+---------+--------+----+--------+-------+---------+
    |normal.|     5450|       0|  SF|     tcp|   http|      181|
    |normal.|      486|       0|  SF|     tcp|   http|      239|
    |normal.|     1337|       0|  SF|     tcp|   http|      235|
    |normal.|     1337|       0|  SF|     tcp|   http|      219|
    |normal.|     2032|       0|  SF|     tcp|   http|      217|
    +-------+---------+--------+----+--------+-------+---------+
    only showing top 5 rows
    


### attack 분류

네트워크 침입이 'attack' 또는 'normal'에 따라 구분해서 ```attackB``` 컬럼을 생성한다.


```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType
attack_udf = udf(lambda x: "normal" if x =="normal." else "attack", StringType())
myDf=_df.withColumn("attackB", attack_udf(_df.attack))
```


```python
myDf.printSchema()
```

    root
     |-- attack: string (nullable = true)
     |-- dst_bytes: long (nullable = true)
     |-- duration: long (nullable = true)
     |-- flag: string (nullable = true)
     |-- protocol: string (nullable = true)
     |-- service: string (nullable = true)
     |-- src_bytes: long (nullable = true)
     |-- attackB: string (nullable = true)
    


네트워크 침입 attack을 세분화하여 normal, dos, r2l, u2r, probling으로 **5종류**로 구분한다.
구분 문자열이 **점('.')**으로 끝난다는 점에 주의하다.

attack 4종류 | 설명 | 41번째 열
-----|-----|-----
DOS | denial-of-service, e.g. syn flood | back, land, neptune, pod, smurf, teardrop
R2L | unauthorized access from a remote machine | ftp_write, guess_passwd, imap, multihop, phf, spy, warezclient, warezmaster
U2R | unauthorized access to local superuser (root) privileges | buffer_overflow, loadmodule, perl, rootkit
probing | surveillance and other probing | ipsweep, nmap, portsweep, satan

위 표에 따라 ```udf()``` 함수를 사용해서 if문으로 'noraml' 및 'attack'을 총 5가지 종류로 구분한다.
반환 값은 ```StringType()```이다.


```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType
def classify41(s):
    _5=""
    if s=="normal.":
        _5="normal"
    elif s=="back." or s=="land." or s=="neptune." or s=="pod." or s=="smurf." or s=="teardrop.":
        _5="dos"
    elif s=="ftp_write." or s=="guess_passwd." or s=="imap." or s=="multihop." or s=="phf." or\
        s=="spy." or s=="warezclient." or s=="warezmaster.":
        _5="r2l"
    elif s=="buffer_overflow." or s=="loadmodule." or s=="perl." or s=="rootkit.":
        _5="u2r"
    elif s=="ipsweep." or s=="nmap." or s=="portsweep." or s=="satan.":
        _5="probing"
    return _5

attack5_udf = udf(classify41, StringType())
```


```python
myDf=myDf.withColumn("attack5", attack5_udf(_df.attack))
```


```python
myDf.printSchema()
```

    root
     |-- attack: string (nullable = true)
     |-- dst_bytes: long (nullable = true)
     |-- duration: long (nullable = true)
     |-- flag: string (nullable = true)
     |-- protocol: string (nullable = true)
     |-- service: string (nullable = true)
     |-- src_bytes: long (nullable = true)
     |-- attackB: string (nullable = true)
     |-- attack5: string (nullable = true)
    


잘 분류되었는지 일부 데이터를 살펴보자.


```python
myDf.show(5)
```

    +-------+---------+--------+----+--------+-------+---------+-------+-------+
    | attack|dst_bytes|duration|flag|protocol|service|src_bytes|attackB|attack5|
    +-------+---------+--------+----+--------+-------+---------+-------+-------+
    |normal.|     5450|       0|  SF|     tcp|   http|      181| normal| normal|
    |normal.|      486|       0|  SF|     tcp|   http|      239| normal| normal|
    |normal.|     1337|       0|  SF|     tcp|   http|      235| normal| normal|
    |normal.|     1337|       0|  SF|     tcp|   http|      219| normal| normal|
    |normal.|     2032|       0|  SF|     tcp|   http|      217| normal| normal|
    +-------+---------+--------+----+--------+-------+---------+-------+-------+
    only showing top 5 rows
    


### attack, normal 특징 분석

```attack5``` 별로 건수를 세어보자.


```python
myDf.groupBy('attack5').count().show()
```

    +-------+------+
    |attack5| count|
    +-------+------+
    |probing|  4107|
    |    u2r|    52|
    | normal| 97278|
    |    r2l|  1126|
    |    dos|391458|
    +-------+------+
    


```attack5``` 별로 공격의 특징을 분석해보자. 어떤 ```protocol```, ```src_bytes```, ```duration```이 어떤지 계산할 수 있다.


```python
myDf.groupBy("protocol").count().show()
```

    +--------+------+
    |protocol| count|
    +--------+------+
    |     tcp|190065|
    |     udp| 20354|
    |    icmp|283602|
    +--------+------+
    



```python
myDf.groupBy('attackB','protocol').count().show()
```

    +-------+--------+------+
    |attackB|protocol| count|
    +-------+--------+------+
    | normal|     udp| 19177|
    | normal|    icmp|  1288|
    | normal|     tcp| 76813|
    | attack|    icmp|282314|
    | attack|     tcp|113252|
    | attack|     udp|  1177|
    +-------+--------+------+
    



```python
myDf.groupBy('attackB').pivot('protocol').count().show()
```

    +-------+------+------+-----+
    |attackB|  icmp|   tcp|  udp|
    +-------+------+------+-----+
    | normal|  1288| 76813|19177|
    | attack|282314|113252| 1177|
    +-------+------+------+-----+
    



```python
myDf.groupBy('attack5').pivot('protocol').avg('src_bytes').show()
```

    +-------+------------------+------------------+------------------+
    |attack5|              icmp|               tcp|               udp|
    +-------+------------------+------------------+------------------+
    |probing|10.700793650793651| 261454.6003016591|25.235897435897435|
    |    u2r|              null| 960.8979591836735|13.333333333333334|
    | normal| 91.47049689440993|1439.3120305156679| 98.01220211711947|
    |    r2l|              null|271972.57460035523|              null|
    |    dos| 936.2672084368129| 1090.303422435458|              28.0|
    +-------+------------------+------------------+------------------+
    



```python
myDf.groupBy('attack5').avg('duration').show()
```

    +-------+--------------------+
    |attack5|       avg(duration)|
    +-------+--------------------+
    |probing|   485.0299488677867|
    |    u2r|    80.9423076923077|
    | normal|  216.65732231336992|
    |    r2l|   559.7522202486679|
    |    dos|7.254929008986916E-4|
    +-------+--------------------+
    



```python
from pyspark.sql import functions as F
myDf.groupBy('attackB').pivot('protocol').agg(F.max('dst_bytes')).show()
```

    +-------+----+-------+---+
    |attackB|icmp|    tcp|udp|
    +-------+----+-------+---+
    | normal|   0|5134218|516|
    | attack|   0|5155468| 74|
    +-------+----+-------+---+
    


좀 더 세밀한 조건으로 ```duration>1000)```, ```dst_bytes==0```인 경우의 건수를 계산할 수 있다.


```python
myDf.select("protocol", "duration", "dst_bytes")\
    .filter(_df.duration>1000)\
    .filter(_df.dst_bytes==0)\
    .groupBy("protocol")\
    .count()\
    .show()
```

    +--------+-----+
    |protocol|count|
    +--------+-----+
    |     tcp|  139|
    +--------+-----+
    


### SQL

SQL을 사용해보자. 위에 사용했던 ```_df```에서 임시 테이블 ```_tab```을 생성한다.


```python
_df.registerTempTable("_tab")
```


```python
tcp_interactions = spark.sql(
"""
    SELECT duration, dst_bytes FROM _tab
    WHERE protocol = 'tcp' AND duration > 1000 AND dst_bytes = 0
""")
```


```python
tcp_interactions.show(5)
```

    +--------+---------+
    |duration|dst_bytes|
    +--------+---------+
    |    5057|        0|
    |    5059|        0|
    |    5051|        0|
    |    5056|        0|
    |    5051|        0|
    +--------+---------+
    only showing top 5 rows
    



```python
tcp_interactions_out = tcp_interactions.rdd\
    .map(lambda p: "Duration: {}, Dest. bytes: {}".format(p.duration, p.dst_bytes))
```


```python
for i,ti_out in enumerate(tcp_interactions_out.collect()):
    if(i%10==0):
        print ti_out
```

    Duration: 5057, Dest. bytes: 0
    Duration: 5043, Dest. bytes: 0
    Duration: 5046, Dest. bytes: 0
    Duration: 5051, Dest. bytes: 0
    Duration: 5057, Dest. bytes: 0
    Duration: 5063, Dest. bytes: 0
    Duration: 42448, Dest. bytes: 0
    Duration: 40121, Dest. bytes: 0
    Duration: 31709, Dest. bytes: 0
    Duration: 30619, Dest. bytes: 0
    Duration: 22616, Dest. bytes: 0
    Duration: 21455, Dest. bytes: 0
    Duration: 13998, Dest. bytes: 0
    Duration: 12933, Dest. bytes: 0


## 문제 S-2: Twitter JSON 데이터 읽기

* [nok] 현재 디렉토리 _tweet.json
    * src/ds_twitter_3.py로 변경 (ds_twitter_3.json으로 저장)



* Twitter JSON을 읽을 경우

구분 | 예
-------|-------
unicode를 사용하면 backslash | "{\"created_at\":\"Sun Nov 13 00:05:19 +0000 2016\"
보통 | {"created_at":"Sun Nov 13 00:05:19 +0000 2016"

* allowBackslashEscapingAnyCharacter

json 파일을 읽어서 DataFrame을 생성해보자. 
아래와 같이 ```read.json()``` 하거나 또는 ```read.load.format("json").load()``` 이라고 해도 된다.


```python
t2df= spark.read.json(os.path.join("src","ds_twitter_seoul_3.json"))
print type(t2df)
```

    <class 'pyspark.sql.dataframe.DataFrame'>


트윗의 'id','lang','text' 컬럼만을 선택해서 한 줄을 출력해보자.


```python
res=t2df.select('id','lang','text').take(1)
```


```python
for e in res:
    print e['id'],e['lang'],e['text']
```

    801657325836763136 en RT @soompi: #SEVENTEEN’s Mingyu, Jin Se Yeon, And Leeteuk To MC For 2016 Super Seoul Dream Concert 
    https://t.co/1XRSaRBbE0 https://t.co/fi…



```python
twitterDF= spark.read.json(os.path.join("src","ds_twitter_1_noquote.json"))
```


```python
twitterDF.printSchema()
```

    root
     |-- contributors: string (nullable = true)
     |-- coordinates: string (nullable = true)
     |-- created_at: string (nullable = true)
     |-- entities: struct (nullable = true)
     |    |-- hashtags: array (nullable = true)
     |    |    |-- element: string (containsNull = true)
     |    |-- symbols: array (nullable = true)
     |    |    |-- element: string (containsNull = true)
     |    |-- urls: array (nullable = true)
     |    |    |-- element: string (containsNull = true)
     |    |-- user_mentions: array (nullable = true)
     |    |    |-- element: string (containsNull = true)
     |-- favorite_count: long (nullable = true)
     |-- favorited: boolean (nullable = true)
     |-- geo: string (nullable = true)
     |-- id: long (nullable = true)
     |-- id_str: string (nullable = true)
     |-- in_reply_to_screen_name: string (nullable = true)
     |-- in_reply_to_status_id: string (nullable = true)
     |-- in_reply_to_status_id_str: string (nullable = true)
     |-- in_reply_to_user_id: string (nullable = true)
     |-- in_reply_to_user_id_str: string (nullable = true)
     |-- is_quote_status: boolean (nullable = true)
     |-- lang: string (nullable = true)
     |-- place: string (nullable = true)
     |-- retweet_count: long (nullable = true)
     |-- retweeted: boolean (nullable = true)
     |-- source: string (nullable = true)
     |-- text: string (nullable = true)
     |-- truncated: boolean (nullable = true)
     |-- user: struct (nullable = true)
     |    |-- contributors_enabled: boolean (nullable = true)
     |    |-- created_at: string (nullable = true)
     |    |-- default_profile: boolean (nullable = true)
     |    |-- default_profile_image: boolean (nullable = true)
     |    |-- description: string (nullable = true)
     |    |-- entities: struct (nullable = true)
     |    |    |-- description: struct (nullable = true)
     |    |    |    |-- urls: array (nullable = true)
     |    |    |    |    |-- element: string (containsNull = true)
     |    |-- favourites_count: long (nullable = true)
     |    |-- follow_request_sent: boolean (nullable = true)
     |    |-- followers_count: long (nullable = true)
     |    |-- following: boolean (nullable = true)
     |    |-- friends_count: long (nullable = true)
     |    |-- geo_enabled: boolean (nullable = true)
     |    |-- has_extended_profile: boolean (nullable = true)
     |    |-- id: long (nullable = true)
     |    |-- id_str: string (nullable = true)
     |    |-- is_translation_enabled: boolean (nullable = true)
     |    |-- is_translator: boolean (nullable = true)
     |    |-- lang: string (nullable = true)
     |    |-- listed_count: long (nullable = true)
     |    |-- location: string (nullable = true)
     |    |-- name: string (nullable = true)
     |    |-- notifications: boolean (nullable = true)
     |    |-- profile_background_color: string (nullable = true)
     |    |-- profile_background_image_url: string (nullable = true)
     |    |-- profile_background_image_url_https: string (nullable = true)
     |    |-- profile_background_tile: boolean (nullable = true)
     |    |-- profile_image_url: string (nullable = true)
     |    |-- profile_image_url_https: string (nullable = true)
     |    |-- profile_link_color: string (nullable = true)
     |    |-- profile_sidebar_border_color: string (nullable = true)
     |    |-- profile_sidebar_fill_color: string (nullable = true)
     |    |-- profile_text_color: string (nullable = true)
     |    |-- profile_use_background_image: boolean (nullable = true)
     |    |-- protected: boolean (nullable = true)
     |    |-- screen_name: string (nullable = true)
     |    |-- statuses_count: long (nullable = true)
     |    |-- time_zone: string (nullable = true)
     |    |-- translator_type: string (nullable = true)
     |    |-- url: string (nullable = true)
     |    |-- utc_offset: string (nullable = true)
     |    |-- verified: boolean (nullable = true)
    



```python
twitterDF.select('text').show()
```

    +---------------+
    |           text|
    +---------------+
    |Hello 21 160924|
    +---------------+
    



```python
twitterDF.registerTempTable("twitter")
spark.sql("select text from twitter").show()
```

    +---------------+
    |           text|
    +---------------+
    |Hello 21 160924|
    +---------------+
    


## 문제 S-3: 뉴욕에서 출생한 신생아 분석

### 뉴욕에서 출생한 신생아가 년도별 성별에 차이가 있을까?

뉴욕에서 2007년 출생한 유아의 기록이다.
https://health.data.ny.gov/Health/Baby-Names-Beginning-2007/jxy9-yhdk

Column Name | 설명
-----|-----
Year | Year data was collected.
First Name | 이름
County | Location where the baby’s mother resided as stated on their birth certificate.
Sex | F= Female M= Male
Count | Five (5) or more of the same baby name in a county outside of NYC; Ten (10) or more of the same baby name in a NYC borough.


https://catalog.data.gov/dataset

data/dataGovbabyNames.json 은 메타데이터가 있어서 kaggle.com의 baby names를 사용해서 분석?


```requests.get()``` 함수를 사용해서 url로부터 데이터를 읽어 오면 string이다 (예: ```r.iter_lines()```하면 문자 1개씩 가져옴). response를 json으로 읽으면 된다.


```python
import json
import requests
_url="https://health.data.ny.gov/api/views/jxy9-yhdk/rows.json?accessType=DOWNLOAD"
_json=requests.get(_url).json()
```

* json데이터는 meta, data로 구분해서 만들어져 있다.
    * data는 Python List로 구성되어 있다 (앞서 Python dict에서 생성하는 경우와 비교해 본다.)
    * data의 건수는 52,252건



```python
_json.keys()
```




    dict_keys(['meta', 'data'])




```python

```


```python
_json['meta']['view']['name']
```




    'Baby Names: Beginning 2007'




```python
_jsonList=_json['data']
print (len(_jsonList))
```

    70499


_jsonList=_json['data']
print len(_jsonList)



```python
_json['data'][0]
```




    [u'row-56s8~p76n_zvtk',
     u'00000000-0000-0000-39E1-487334739D3C',
     0,
     1527713235,
     None,
     1527713235,
     None,
     u'{ }',
     u'2016',
     u'DAVID',
     u'Kings',
     u'M',
     u'231']



* Python list로부터 Spark Dataframe을 생성한다.


```python
_df=spark.createDataFrame(_json['data'])
_df.count()
```




    145570



* schema를 정하지 않았으므로 임의로 생성된 속성을 사용하고 있다.


```python
_df.printSchema()
```

    root
     |-- _1: long (nullable = true)
     |-- _2: string (nullable = true)
     |-- _3: long (nullable = true)
     |-- _4: long (nullable = true)
     |-- _5: string (nullable = true)
     |-- _6: long (nullable = true)
     |-- _7: string (nullable = true)
     |-- _8: string (nullable = true)
     |-- _9: string (nullable = true)
     |-- _10: string (nullable = true)
     |-- _11: string (nullable = true)
     |-- _12: string (nullable = true)
     |-- _13: string (nullable = true)
    


* 컬럼명을 새로 정의한다. 


```python

_myDf = _df.select('_9','_10','_11','_12','_13')
```


```python
_myDf=_myDf.withColumn('count',_df['_13'].cast("integer")).drop('_13')
_myDf=_myDf.withColumnRenamed('_9','year')
_myDf=_myDf.withColumnRenamed('_10','fname')
_myDf=_myDf.withColumnRenamed('_11','county')
_myDf=_myDf.withColumnRenamed('_12','sex')
_myDf.printSchema()
_myDf.show(5)
```

    root
     |-- year: string (nullable = true)
     |-- fname: string (nullable = true)
     |-- county: string (nullable = true)
     |-- sex: string (nullable = true)
     |-- count: integer (nullable = true)
    
    +----+-------+-----------+---+-----+
    |year|  fname|     county|sex|count|
    +----+-------+-----------+---+-----+
    |2013|  GAVIN|ST LAWRENCE|  M|    9|
    |2013|   LEVI|ST LAWRENCE|  M|    9|
    |2013|  LOGAN|   NEW YORK|  M|   44|
    |2013| HUDSON|   NEW YORK|  M|   49|
    |2013|GABRIEL|   NEW YORK|  M|   50|
    +----+-------+-----------+---+-----+
    only showing top 5 rows
    



```python
_myDf.filter(_myDf['fname'] == u'GAVIN').show(2)
```

    +----+-----+-----------+---+-----+
    |year|fname|     county|sex|count|
    +----+-----+-----------+---+-----+
    |2013|GAVIN|ST LAWRENCE|  M|    9|
    |2013|GAVIN|    SUFFOLK|  M|   54|
    +----+-----+-----------+---+-----+
    only showing top 2 rows
    


* Sql을 사용할 수 있다.


```python
_myDf.registerTempTable("babyNames")
spark.sql("select distinct(fname) from babyNames").show(5)
```

    +------+
    | fname|
    +------+
    |MILANA|
    |  JADE|
    |  ANNA|
    |HUNTER|
    |ANJALI|
    +------+
    only showing top 5 rows
    


* 년도별 성별 빈도수를 계산한다.


```python
_myDf.sort('year').groupBy('year').pivot('sex').count().show()
```

    +----+-----+-----+
    |year|    F|    M|
    +----+-----+-----+
    |2007| 3002| 3365|
    |2008| 3039| 3442|
    |2009| 2917| 3395|
    |2010| 2925| 3267|
    |2011| 2918| 3298|
    |2012| 2872| 3292|
    |2013| 2836| 3322|
    |2014| 4121| 4241|
    |2015|50803|42515|
    +----+-----+-----+
    


## 문제 S-4: 우버 택시의 운행기록 분석

* 질문: 2015년 가장 많은 운행을 한 base는?
https://github.com/tmcgrath/spark-with-python-course/blob/master/Spark-SQL-CSV-with-Python.ipynb

* fivethirtyeight
    * git clone https://github.com/fivethirtyeight/uber-tlc-foil-response.git
        daily Uber trip statistics in January and February 2015

dispatching_base_number | date | active_vehicles | trips
----------|----------|----------|----------
B02512 | 1/1/2015 | 190 | 1132
B02765 | 1/1/2015 | 225 | 1765


### Data


```python
import os
data_home=os.path.join(os.environ['HOME'],"Code/git/else/uber-tlc-foil-response")
uberCsv=os.path.join(data_home,"Uber-Jan-Feb-FOIL.csv")
```

### RDD


```python
_rdd = spark.sparkContext.textFile(uberCsv)
```

* header는 속성 명을 가지고 있다.
* header를 제외하고 읽기


```python
header = _rdd.first() #extract header
_rdd = _rdd.filter(lambda x:x != header)

print (_rdd.count())
print (f"Header: {header}")
```

    354
    Header: dispatching_base_number,date,active_vehicles,trips


* csv는 comma seperated 형식이므로, ','로 분리
* 첫번째 열에서 key값을 추출한다 (header값 포함)


```python
_myRdd = _rdd.map(lambda line: line.split(","))
```


```python
_row0keys=_myRdd.map(lambda row: row[0]).distinct().collect()

print (_row0keys)
```

    ['B02512', 'B02765', 'B02764', 'B02682', 'B02617', 'B02598']



```python
_myRdd.filter(lambda row: "B02512" in row).count()
```




    59



* B02512인 경우, trips가 2000보다 큰 레코드 수집


```python
_myRdd.filter(lambda row: "B02512" in row).filter(lambda row: int(row[3])>2000).collect()
```




    [['B02512', '1/30/2015', '256', '2016'],
     ['B02512', '2/5/2015', '264', '2022'],
     ['B02512', '2/12/2015', '269', '2092'],
     ['B02512', '2/13/2015', '281', '2408'],
     ['B02512', '2/14/2015', '236', '2055'],
     ['B02512', '2/19/2015', '250', '2120'],
     ['B02512', '2/20/2015', '272', '2380'],
     ['B02512', '2/21/2015', '238', '2149'],
     ['B02512', '2/27/2015', '272', '2056']]




```python
_myRdd.map(lambda x: (x[0], int(x[3]))).reduceByKey(lambda k,v: k + v).collect()
```




    [('B02512', 93786),
     ('B02765', 193670),
     ('B02764', 1914449),
     ('B02682', 662509),
     ('B02617', 725025),
     ('B02598', 540791)]




```python
def countPartitions(id,iterator): 
    c = 0 
    for _ in iterator: 
        c += 1
    yield (id,c) 
_wc=wc.mapPartitions(countPartitions)
```


    ---------------------------------------------------------------------------

    NameError                                 Traceback (most recent call last)

    <ipython-input-128-715387e656bb> in <module>
          4         c += 1
          5     yield (id,c)
    ----> 6 _wc=wc.mapPartitions(countPartitions)
    

    NameError: name 'wc' is not defined


### DataFrame

spark.createDataFrame()으로 읽으면, header가 ```_1, _2, _3, _4```로 주어진다.
read() 함수의 option을 설정해서 header를 컬럼명으로 읽도록 하자.


```python
uberDf = spark\
            .read\
            .option("header", "true")\
            .csv(uberCsv)
```


```python
uberDf.count()
```




    354




```python
uberDf.show(3)
```

    +-----------------------+--------+---------------+-----+
    |dispatching_base_number|    date|active_vehicles|trips|
    +-----------------------+--------+---------------+-----+
    |                 B02512|1/1/2015|            190| 1132|
    |                 B02765|1/1/2015|            225| 1765|
    |                 B02764|1/1/2015|           3427|29421|
    +-----------------------+--------+---------------+-----+
    only showing top 3 rows
    



```python
uberDf.printSchema()
```

    root
     |-- dispatching_base_number: string (nullable = true)
     |-- date: string (nullable = true)
     |-- active_vehicles: string (nullable = true)
     |-- trips: string (nullable = true)
    


trips는 현재 문자열로 되어 있어서, 정수로 형변환을 해준다.


```python
_myDf=uberDf.withColumn('iTrips', uberDf['trips'].cast("integer")).drop('trips')
```


```python
_myDf=_myDf.withColumnRenamed('dispatching_base_number','baseNum')
```


```python
_myDf.printSchema()
```

    root
     |-- baseNum: string (nullable = true)
     |-- date: string (nullable = true)
     |-- active_vehicles: string (nullable = true)
     |-- iTrips: integer (nullable = true)
    



```python
_myDf.show(5)
```

    +-------+--------+---------------+------+
    |baseNum|    date|active_vehicles|iTrips|
    +-------+--------+---------------+------+
    | B02512|1/1/2015|            190|  1132|
    | B02765|1/1/2015|            225|  1765|
    | B02764|1/1/2015|           3427| 29421|
    | B02682|1/1/2015|            945|  7679|
    | B02617|1/1/2015|           1228|  9537|
    +-------+--------+---------------+------+
    only showing top 5 rows
    



```python
_myDf.groupBy('baseNum','iTrips').count().show()
```

    +-------+------+-----+
    |baseNum|iTrips|count|
    +-------+------+-----+
    | B02682| 13355|    1|
    | B02765|  2623|    1|
    | B02617| 12016|    1|
    | B02764| 36318|    1|
    | B02682|  4414|    1|
    | B02617|  4325|    1|
    | B02764| 31173|    1|
    | B02617| 10664|    1|
    | B02598| 11897|    1|
    | B02617| 11401|    1|
    | B02512|  1831|    1|
    | B02765|  7658|    1|
    | B02617| 13688|    1|
    | B02764| 31957|    1|
    | B02682|  8010|    1|
    | B02764| 33756|    1|
    | B02512|  2056|    1|
    | B02512|   791|    1|
    | B02512|  2380|    1|
    | B02598|  2957|    1|
    +-------+------+-----+
    only showing top 20 rows
    



```python

```


```python

```


```python

```


```python

```

## 문제 S-5: JDBC를 사용해서 데이터 읽기

* jdbc를 연결하는 방식은 Java와 같이 'driver', 'url'을 설정하면 된다.
* 여기서는 sqlite를 실습한다.

* sqlite와 같이 Spark 패키지가 없는 경우, jar를 다운로드하고
설정파일 conf/spark-defaults.conf에 'spark.driver.extraClassPath'를 추가한다.



```python
print spark.conf.get('spark.driver.extraClassPath')
```

    /home/jsl/Code/git/bb/jsl/pyds/lib/sqlite-jdbc-3.14.2.jar



```python
df=spark.read.format('jdbc')\
    .options(
        url="jdbc:sqlite:/home/jsl/Code/git/bb/sd/src/ds_sql_hello.db",
        dbtable="customer",
        driver="org.sqlite.JDBC"
    ).load()

df.show()
```

    +---+------+
    |cid|c_name|
    +---+------+
    |  1|   Kim|
    +---+------+
    


### MySql

* Spark 패키지를 제공하지 않는다. jar를 'spark.driver.extraClassPath'에 추가하고 사용한다.

* 읽기
```python
dfmysql = sqlContext.read.format('jdbc')\
        .options(
          url='jdbc:mysql://localhost/database_name',
          driver='com.mysql.jdbc.Driver',
          dbtable='SourceTableName',
          user='your_user_name',
          password='your_password')\
        .load()
```

* 쓰기
```python
destination_df.write.format('jdbc')\
        .options(
          url='jdbc:mysql://localhost/database_name',
          driver='com.mysql.jdbc.Driver',
          dbtable='DestinationTableName',
          user='your_user_name',
          password='your_password')\
        .mode('append')\
        .save()
```

```python
bin/spark-submit --jars mysql-connector-java-5.1.40-bin.jar
      /path_to_your_program/spark_database.py
```

## S.7 MongoDB Spark connector

* Spark에서 MongoDB에 저장된 데이터를 읽어 온다.
* 참조: pymongo-spark (Spark와 PyMongo를 사용하는 Python 라이브러리, 설치하려면 pip install pymongo-spark)


### S.7.1 설정

* 참조 https://docs.mongodb.com/spark-connector/
* 설정파일 conf/spark-defaults.conf 수정
    * Spark 버전에 맞는 jar를 선택한다.
    * MongoDB<3.2인 경우, spark.mongodb.input.partitioner가 필요하다.
    * packages 여러 개를 넣을 경우에는 컴마로 분리한다.

```python
$vim conf/spark-defaults.conf 
spark.jars.packages=org.mongodb.spark:mongo-spark-connector_2.10:1.1.0
spark.mongodb.input.partitioner=MongoPaginateBySizePartitioner
```


```python
print spark.conf.get('spark.jars.packages')
```

    graphframes:graphframes:0.4.0-spark2.0-s_2.11,org.mongodb.spark:mongo-spark-connector_2.10:2.0.0,com.databricks:spark-csv_2.11:1.5.0


### S.7.2 uri

* SparkSession에 uri를 설정할 수 있다. 연결에 필요한 ip, database, collection을 정의한다.

```python
spark = pyspark.sql.SparkSession.builder\
    .master("local")\
    .appName("myApp")\
    .config("spark.mongodb.input.uri", "mongodb://127.0.0.1/myDB.ds_spark_df_mongo") \
    .config("spark.mongodb.output.uri", "mongodb://127.0.0.1/myDB.ds_spark_df_mongo") \
    .getOrCreate()
```

* 또는 실행시점에 설정할 수 있다 (아래 참조)


### S.7.3 MongoDB Python API

* format은 "com.mongodb.spark.sql.DefaultSource"로 설정한다.
* 'option'을 사용해서 실행시점에 Database, Colleciton 명을 설정할 수 있다.

구분 | 명령어 예
-----|-----
쓰기 | DataFrame.write.format("com.mongodb.spark.sql.DefaultSource")\<br>.mode("overwrite")\<br>.option("uri","mongodb://127.0.0.1/myDB.ds_spark_ml")\<br>.save()
읽기 | spark.read.format("com.mongodb.spark.sql.DefaultSource")\<br>.option("uri","mongodb://127.0.0.1/ds_twitter.seoul")\<br>.load()


### S.7.4 연습으로 쓰기, 읽기


```python
people = spark.createDataFrame([("kim",10),("lee",20),("choi",30),("park",40)],["name", "age"])
```


```python
people.write.format("com.mongodb.spark.sql.DefaultSource")\
    .mode("append")\
    .option("uri","mongodb://127.0.0.1/myDB.ds_spark_ml")\
    .save()
```


```python
df = spark.read.format("com.mongodb.spark.sql.DefaultSource")\
    .option("uri","mongodb://127.0.0.1/myDB.ds_spark_ml")\
    .load()
```


```python
df.printSchema()
```

    root
     |-- _id: struct (nullable = true)
     |    |-- oid: string (nullable = true)
     |-- age: long (nullable = true)
     |-- name: string (nullable = true)
    



```python
df.select('name').show(3)
```

    +----+
    |name|
    +----+
    | kim|
    | lee|
    |choi|
    +----+
    only showing top 3 rows
    



```python
df = spark.read.format("com.mongodb.spark.sql.DefaultSource")\
    .option("uri","mongodb://127.0.0.1/ds_twitter.seoul")\
    .load()
df.select('text').show(5)
```

    +--------------------+
    |                text|
    +--------------------+
    |RT @always_gd: #B...|
    |RT @InfiniteUpdat...|
    |RT @InfiniteUpdat...|
    |RT @PartOfJimin: ...|
    |RT @MHDEFB: มาแล้...|
    +--------------------+
    only showing top 5 rows
    



```python
df.columns
```




    ['_id',
     'contributors',
     'coordinates',
     'created_at',
     'entities',
     'extended_entities',
     'favorite_count',
     'favorited',
     'geo',
     'id',
     'id_str',
     'in_reply_to_screen_name',
     'in_reply_to_status_id',
     'in_reply_to_status_id_str',
     'in_reply_to_user_id',
     'in_reply_to_user_id_str',
     'is_quote_status',
     'lang',
     'metadata',
     'place',
     'possibly_sensitive',
     'quoted_status',
     'quoted_status_id',
     'quoted_status_id_str',
     'retweet_count',
     'retweeted',
     'retweeted_status',
     'source',
     'text',
     'truncated',
     'user']



## S.8 spark-submit

* spark-submit는 일괄실행 (self-contained app in quick-start 참조)

* MongoDB를 사용하려면, spark-defaults.conf에 jar를 추가한다 (앞서 미리 설정하였다.)

* spark-submit을 실행하기 전, 'conf/log4j.properties'를 수정 log level을 ERROR로 설정하였다.
```python
log4j.rootCategory=ERROR, console
```

### S.8.1 간단한 작업

* DataFrame 만들고, 출력하기



```python
%%writefile src/ds_spark_sql.py
#!/usr/bin/env python
# -*- coding: UTF-8 -*-
import pyspark

def doIt():
    d = [{'name': 'Alice', 'age': 1}]
    print spark.createDataFrame(d).collect()

if __name__ == "__main__":
    myConf=pyspark.SparkConf()
    spark = pyspark.sql.SparkSession.builder\
        .master("local")\
        .appName("myApp")\
        .config(conf=myConf)\
        .getOrCreate()
    doIt()
    spark.stop()

```

    Overwriting src/ds_spark_sql.py



```python
!/home/jsl/Downloads/spark-2.0.0-bin-hadoop2.7/bin/spark-submit src/ds_spark_sql.py
```

    Ivy Default Cache set to: /home/jsl/.ivy2/cache
    The jars for the packages stored in: /home/jsl/.ivy2/jars
    :: loading settings :: url = jar:file:/home/jsl/Downloads/spark-2.0.0-bin-hadoop2.7/jars/ivy-2.4.0.jar!/org/apache/ivy/core/settings/ivysettings.xml
    graphframes#graphframes added as a dependency
    org.mongodb.spark#mongo-spark-connector_2.10 added as a dependency
    com.databricks#spark-csv_2.11 added as a dependency
    :: resolving dependencies :: org.apache.spark#spark-submit-parent;1.0
    	confs: [default]
    	found graphframes#graphframes;0.4.0-spark2.0-s_2.11 in spark-packages
    	found com.typesafe.scala-logging#scala-logging-api_2.11;2.1.2 in central
    	found com.typesafe.scala-logging#scala-logging-slf4j_2.11;2.1.2 in central
    	found org.scala-lang#scala-reflect;2.11.0 in central
    	found org.slf4j#slf4j-api;1.7.7 in central
    	found org.mongodb.spark#mongo-spark-connector_2.10;2.0.0 in central
    	found org.mongodb#mongo-java-driver;3.2.2 in central
    	found com.databricks#spark-csv_2.11;1.5.0 in central
    	found org.apache.commons#commons-csv;1.1 in central
    	found com.univocity#univocity-parsers;1.5.1 in central
    :: resolution report :: resolve 245ms :: artifacts dl 7ms
    	:: modules in use:
    	com.databricks#spark-csv_2.11;1.5.0 from central in [default]
    	com.typesafe.scala-logging#scala-logging-api_2.11;2.1.2 from central in [default]
    	com.typesafe.scala-logging#scala-logging-slf4j_2.11;2.1.2 from central in [default]
    	com.univocity#univocity-parsers;1.5.1 from central in [default]
    	graphframes#graphframes;0.4.0-spark2.0-s_2.11 from spark-packages in [default]
    	org.apache.commons#commons-csv;1.1 from central in [default]
    	org.mongodb#mongo-java-driver;3.2.2 from central in [default]
    	org.mongodb.spark#mongo-spark-connector_2.10;2.0.0 from central in [default]
    	org.scala-lang#scala-reflect;2.11.0 from central in [default]
    	org.slf4j#slf4j-api;1.7.7 from central in [default]
    	---------------------------------------------------------------------
    	|                  |            modules            ||   artifacts   |
    	|       conf       | number| search|dwnlded|evicted|| number|dwnlded|
    	---------------------------------------------------------------------
    	|      default     |   10  |   0   |   0   |   0   ||   10  |   0   |
    	---------------------------------------------------------------------
    :: retrieving :: org.apache.spark#spark-submit-parent
    	confs: [default]
    	0 artifacts copied, 10 already retrieved (0kB/7ms)
    /home/jsl/Downloads/spark-2.0.0-bin-hadoop2.7/python/lib/pyspark.zip/pyspark/sql/session.py:316: UserWarning: inferring schema from dict is deprecated,please use pyspark.sql.Row instead
      warnings.warn("inferring schema from dict is deprecated,"
    [Row(age=1, name=u'Alice')]


### S.8.2 MongoDB

* Database, Collection 읽기, 쓰기


```python
%%writefile src/ds_spark_mongo.py
#!/usr/bin/env python
# -*- coding: UTF-8 -*-
import pyspark
def doIt():
    print "---------RESULT-----------"
    print "------mongodb write-------"
    myRdd = spark.sparkContext.parallelize([
        ("js", 150),
        ("Gandalf", 1000),
        ("Thorin", 195),
        ("Balin", 178),
        ("Kili", 77),
        ("Dwalin", 169),
        ("Oin", 167),
        ("Gloin", 158),
        ("Fili", 82),
        ("Bombur", None)
    ])
    myDf = spark.createDataFrame(myRdd, ["name", "age"])
    print myDf
    myDf.write.format("com.mongodb.spark.sql.DefaultSource").mode("overwrite").save()
    print "---------read-----------"
    df = spark.read.format("com.mongodb.spark.sql.DefaultSource").load()
    print df.printSchema()
    df.registerTempTable("myTable")
    myTab = spark.sql("SELECT name, age FROM myTable WHERE age >= 100")
    myTab.show()

if __name__ == "__main__":
    myConf=pyspark.SparkConf()
    spark = pyspark.sql.SparkSession.builder\
        .master("local")\
        .appName("myApp")\
        .config("spark.mongodb.input.uri", "mongodb://127.0.0.1/myDB.ds_spark_df_mongo") \
        .config("spark.mongodb.output.uri", "mongodb://127.0.0.1/myDB.ds_spark_df_mongo") \
        .getOrCreate()
    doIt()
    spark.stop()

```

    Overwriting src/ds_spark_mongo.py



```python
!/home/jsl/Downloads/spark-2.0.0-bin-hadoop2.7/bin/spark-submit src/ds_spark_mongo.py
```

    Ivy Default Cache set to: /home/jsl/.ivy2/cache
    The jars for the packages stored in: /home/jsl/.ivy2/jars
    :: loading settings :: url = jar:file:/home/jsl/Downloads/spark-2.0.0-bin-hadoop2.7/jars/ivy-2.4.0.jar!/org/apache/ivy/core/settings/ivysettings.xml
    graphframes#graphframes added as a dependency
    org.mongodb.spark#mongo-spark-connector_2.10 added as a dependency
    com.databricks#spark-csv_2.11 added as a dependency
    :: resolving dependencies :: org.apache.spark#spark-submit-parent;1.0
    	confs: [default]
    	found graphframes#graphframes;0.4.0-spark2.0-s_2.11 in spark-packages
    	found com.typesafe.scala-logging#scala-logging-api_2.11;2.1.2 in central
    	found com.typesafe.scala-logging#scala-logging-slf4j_2.11;2.1.2 in central
    	found org.scala-lang#scala-reflect;2.11.0 in central
    	found org.slf4j#slf4j-api;1.7.7 in central
    	found org.mongodb.spark#mongo-spark-connector_2.10;2.0.0 in central
    	found org.mongodb#mongo-java-driver;3.2.2 in central
    	found com.databricks#spark-csv_2.11;1.5.0 in central
    	found org.apache.commons#commons-csv;1.1 in central
    	found com.univocity#univocity-parsers;1.5.1 in central
    :: resolution report :: resolve 250ms :: artifacts dl 8ms
    	:: modules in use:
    	com.databricks#spark-csv_2.11;1.5.0 from central in [default]
    	com.typesafe.scala-logging#scala-logging-api_2.11;2.1.2 from central in [default]
    	com.typesafe.scala-logging#scala-logging-slf4j_2.11;2.1.2 from central in [default]
    	com.univocity#univocity-parsers;1.5.1 from central in [default]
    	graphframes#graphframes;0.4.0-spark2.0-s_2.11 from spark-packages in [default]
    	org.apache.commons#commons-csv;1.1 from central in [default]
    	org.mongodb#mongo-java-driver;3.2.2 from central in [default]
    	org.mongodb.spark#mongo-spark-connector_2.10;2.0.0 from central in [default]
    	org.scala-lang#scala-reflect;2.11.0 from central in [default]
    	org.slf4j#slf4j-api;1.7.7 from central in [default]
    	---------------------------------------------------------------------
    	|                  |            modules            ||   artifacts   |
    	|       conf       | number| search|dwnlded|evicted|| number|dwnlded|
    	---------------------------------------------------------------------
    	|      default     |   10  |   0   |   0   |   0   ||   10  |   0   |
    	---------------------------------------------------------------------
    :: retrieving :: org.apache.spark#spark-submit-parent
    	confs: [default]
    	0 artifacts copied, 10 already retrieved (0kB/7ms)
    ---------RESULT-----------
    ------mongodb write-------
    DataFrame[name: string, age: bigint]
    ---------read-----------
    root
     |-- _id: struct (nullable = true)
     |    |-- oid: string (nullable = true)
     |-- age: long (nullable = true)
     |-- name: string (nullable = true)
    
    None
    +-------+----+
    |   name| age|
    +-------+----+
    |     js| 150|
    |Gandalf|1000|
    | Thorin| 195|
    |  Balin| 178|
    | Dwalin| 169|
    |    Oin| 167|
    |  Gloin| 158|
    +-------+----+
    


## 문제 S-6: MongoDB 저장된 열린데이터 읽어오는 spark-submit

* MongoDB에 저장된 데이터 읽기

구분 | 명
-----|-----
Database | ds_open_subwayPassengersDb
Collection | db_open_subwayTable
key | JSON 계층구조를 따라 읽는다. CardSubwayStatisticsService.row.RIDE_PASGR_NUM

* MongoDB shell

```python
$ mongo
> use ds_open_subwayPassengersDb
switched to db ds_rest_subwayPassengers_mongo_db
> show tables
db_open_subwayTable
system.indexes
> db.db_open_subwayTable.find().limit(1)
{ "_id" : ObjectId("57fa386ff5e6e94359c033e9"), "CardSubwayStatisticsService" : { "row" : [ { "COMMT" : "", "RIDE_PASGR_NUM" : 111275, "WORK_DT" : "20130723", "LINE_NUM" : "중앙선", "SUB_STA_NM" : "용문", "ALIGHT_PASGR_NUM" : 108878, "USE_MON" : "201306" }, { "COMMT" : "", "RIDE_PASGR_NUM" : 11495, "WORK_DT" : "20130723", "LINE_NUM" : "중앙선", "SUB_STA_NM" : "원덕", "ALIGHT_PASGR_NUM" : 10964, "USE_MON" : "201306" }, { "COMMT" : "", "RIDE_PASGR_NUM" : 118103, "WORK_DT" : "20130723", "LINE_NUM" : "중앙선", "SUB_STA_NM" : "양평", "ALIGHT_PASGR_NUM" : 116604, "USE_MON" : "201306" }, { "COMMT" : "", "RIDE_PASGR_NUM" : 10590, "WORK_DT" : "20130723", "LINE_NUM" : "중앙선", "SUB_STA_NM" : "오빈", "ALIGHT_PASGR_NUM" : 10020, "USE_MON" : "201306" }, { "COMMT" : "", "RIDE_PASGR_NUM" : 26304, "WORK_DT" : "20130723", "LINE_NUM" : "중앙선", "SUB_STA_NM" : "아신", "ALIGHT_PASGR_NUM" : 26358, "USE_MON" : "201306" } ], "RESULT" : { "MESSAGE" : "정상 처리되었습니다", "CODE" : "INFO-000" }, "list_total_count" : 530 } }
```

* MongoDB에서 읽기


```python
df = spark.read.format("com.mongodb.spark.sql.DefaultSource")\
    .option("uri","mongodb://127.0.0.1/ds_open_subwayPassengersDb.db_open_subwayTable")\
    .load()
print df.printSchema()
```

    root
     |-- CardSubwayStatisticsService: struct (nullable = true)
     |    |-- row: array (nullable = true)
     |    |    |-- element: struct (containsNull = true)
     |    |    |    |-- COMMT: string (nullable = true)
     |    |    |    |-- RIDE_PASGR_NUM: double (nullable = true)
     |    |    |    |-- WORK_DT: string (nullable = true)
     |    |    |    |-- LINE_NUM: string (nullable = true)
     |    |    |    |-- SUB_STA_NM: string (nullable = true)
     |    |    |    |-- ALIGHT_PASGR_NUM: double (nullable = true)
     |    |    |    |-- USE_MON: string (nullable = true)
     |    |-- RESULT: struct (nullable = true)
     |    |    |-- MESSAGE: string (nullable = true)
     |    |    |-- CODE: string (nullable = true)
     |    |-- list_total_count: integer (nullable = true)
     |-- _id: struct (nullable = true)
     |    |-- oid: string (nullable = true)
    
    None



```python
df.registerTempTable("mySubway")
myTab = spark.sql("SELECT CardSubwayStatisticsService.row.RIDE_PASGR_NUM FROM mySubway")
#print type(myTab)
print myTab.show()
print myTab.first()
print myTab.head()
```

    +--------------------+
    |      RIDE_PASGR_NUM|
    +--------------------+
    |[111275.0, 11495....|
    |[30281.0, 10832.0...|
    |[75553.0, 189783....|
    |[62789.0, 220544....|
    |[74486.0, 152381....|
    |[43418.0, 413533....|
    |[301856.0, 37978....|
    |[146909.0, 227066...|
    |[152275.0, 285263...|
    |[161298.0, 117720...|
    |[95996.0, 39150.0...|
    |[59900.0, 201035....|
    |[228737.0, 108938...|
    |[164574.0, 481998...|
    |[748205.0, 817657...|
    |[206631.0, 188076...|
    |[112991.0, 111791...|
    |[225105.0, 938296...|
    |[175909.0, 271844...|
    |[45047.0, 126837....|
    +--------------------+
    
    None
    Row(RIDE_PASGR_NUM=[111275.0, 11495.0, 118103.0, 10590.0, 26304.0])
    Row(RIDE_PASGR_NUM=[111275.0, 11495.0, 118103.0, 10590.0, 26304.0])



```python
%%writefile src/ds_spark_subway.py
#!/usr/bin/env python
# -*- coding: UTF-8 -*-
import pyspark
def doIt():
    print "---------read-----------"
    df=spark.read.format("com.mongodb.spark.sql.DefaultSource").load()
    #df = sqlContext.read.format("com.mongodb.spark.sql.DefaultSource").load()
    print df.printSchema()
    df.registerTempTable("mySubway")
    myTab = spark.sql("SELECT CardSubwayStatisticsService.row.RIDE_PASGR_NUM FROM mySubway")
    #print type(myTab)
    print myTab.show()
if __name__ == "__main__":
    myConf=pyspark.SparkConf()
    spark = pyspark.sql.SparkSession.builder\
        .master("local")\
        .appName("myApp")\
        .config("spark.mongodb.input.uri", "mongodb://127.0.0.1/ds_open_subwayPassengersDb.db_open_subwayTable") \
        .getOrCreate()
    doIt()
    spark.stop()

```

    Overwriting src/ds_spark_subway.py



```python
!/home/jsl/Downloads/spark-1.6.0-bin-hadoop2.6/bin/spark-submit src/ds_spark_subway.py
```

    Ivy Default Cache set to: /home/jsl/.ivy2/cache
    The jars for the packages stored in: /home/jsl/.ivy2/jars
    :: loading settings :: url = jar:file:/home/jsl/Downloads/spark-2.0.0-bin-hadoop2.7/jars/ivy-2.4.0.jar!/org/apache/ivy/core/settings/ivysettings.xml
    graphframes#graphframes added as a dependency
    org.mongodb.spark#mongo-spark-connector_2.10 added as a dependency
    com.databricks#spark-csv_2.11 added as a dependency
    :: resolving dependencies :: org.apache.spark#spark-submit-parent;1.0
    	confs: [default]
    	found graphframes#graphframes;0.4.0-spark2.0-s_2.11 in spark-packages
    	found com.typesafe.scala-logging#scala-logging-api_2.11;2.1.2 in central
    	found com.typesafe.scala-logging#scala-logging-slf4j_2.11;2.1.2 in central
    	found org.scala-lang#scala-reflect;2.11.0 in central
    	found org.slf4j#slf4j-api;1.7.7 in central
    	found org.mongodb.spark#mongo-spark-connector_2.10;2.0.0 in central
    	found org.mongodb#mongo-java-driver;3.2.2 in central
    	found com.databricks#spark-csv_2.11;1.5.0 in central
    	found org.apache.commons#commons-csv;1.1 in central
    	found com.univocity#univocity-parsers;1.5.1 in central
    :: resolution report :: resolve 244ms :: artifacts dl 7ms
    	:: modules in use:
    	com.databricks#spark-csv_2.11;1.5.0 from central in [default]
    	com.typesafe.scala-logging#scala-logging-api_2.11;2.1.2 from central in [default]
    	com.typesafe.scala-logging#scala-logging-slf4j_2.11;2.1.2 from central in [default]
    	com.univocity#univocity-parsers;1.5.1 from central in [default]
    	graphframes#graphframes;0.4.0-spark2.0-s_2.11 from spark-packages in [default]
    	org.apache.commons#commons-csv;1.1 from central in [default]
    	org.mongodb#mongo-java-driver;3.2.2 from central in [default]
    	org.mongodb.spark#mongo-spark-connector_2.10;2.0.0 from central in [default]
    	org.scala-lang#scala-reflect;2.11.0 from central in [default]
    	org.slf4j#slf4j-api;1.7.7 from central in [default]
    	---------------------------------------------------------------------
    	|                  |            modules            ||   artifacts   |
    	|       conf       | number| search|dwnlded|evicted|| number|dwnlded|
    	---------------------------------------------------------------------
    	|      default     |   10  |   0   |   0   |   0   ||   10  |   0   |
    	---------------------------------------------------------------------
    :: retrieving :: org.apache.spark#spark-submit-parent
    	confs: [default]
    	0 artifacts copied, 10 already retrieved (0kB/7ms)
    ---------read-----------
    root
     |-- CardSubwayStatisticsService: struct (nullable = true)
     |    |-- row: array (nullable = true)
     |    |    |-- element: struct (containsNull = true)
     |    |    |    |-- COMMT: string (nullable = true)
     |    |    |    |-- RIDE_PASGR_NUM: double (nullable = true)
     |    |    |    |-- WORK_DT: string (nullable = true)
     |    |    |    |-- LINE_NUM: string (nullable = true)
     |    |    |    |-- SUB_STA_NM: string (nullable = true)
     |    |    |    |-- ALIGHT_PASGR_NUM: double (nullable = true)
     |    |    |    |-- USE_MON: string (nullable = true)
     |    |-- RESULT: struct (nullable = true)
     |    |    |-- MESSAGE: string (nullable = true)
     |    |    |-- CODE: string (nullable = true)
     |    |-- list_total_count: integer (nullable = true)
     |-- _id: struct (nullable = true)
     |    |-- oid: string (nullable = true)
    
    None
    +--------------------+
    |      RIDE_PASGR_NUM|
    +--------------------+
    |[111275.0, 11495....|
    |[30281.0, 10832.0...|
    |[75553.0, 189783....|
    |[62789.0, 220544....|
    |[74486.0, 152381....|
    |[43418.0, 413533....|
    |[301856.0, 37978....|
    |[146909.0, 227066...|
    |[152275.0, 285263...|
    |[161298.0, 117720...|
    |[95996.0, 39150.0...|
    |[59900.0, 201035....|
    |[228737.0, 108938...|
    |[164574.0, 481998...|
    |[748205.0, 817657...|
    |[206631.0, 188076...|
    |[112991.0, 111791...|
    |[225105.0, 938296...|
    |[175909.0, 271844...|
    |[45047.0, 126837....|
    +--------------------+
    
    None



```python
import os
print os.environ['HOME']
```

    /home/jsl



```python
filen="국내유학중인%2B외국인유학생%2B데이터%282016년%2B12월기준%29.csv"
filen="OverseasStudentsInKorea20161229.csv"
myRdd=spark.sparkContext.textFile(os.path.join(os.environ['HOME'],"Public",filen))
```


```python
data=myRdd.first()
for i in data.split(','):
    print i
print type(i)
```

    ����
    ����
    ������
    ü���ڰ�
    �б���
    ü���� �õ�
    ü���� �ñ���
    <type 'unicode'>



```python
df = spark.read.format('com.databricks.spark.csv')\
    .options(header='true', inferschema='true').load(os.path.join(os.environ['HOME'],"Public",filen))


```


```python
df.printSchema()
```

    root
     |-- ����: string (nullable = true)
     |-- ����: string (nullable = true)
     |-- ������: string (nullable = true)
     |-- ü���ڰ�: string (nullable = true)
     |-- �б���: string (nullable = true)
     |-- ü���� �õ�: string (nullable = true)
     |-- ü���� �ñ���: string (nullable = true)
    



```python

```


```python

```


```python

```


```python

```


```python
from pyspark.context import SparkContext
from pyspark.streaming import StreamingContext

```


```python
sc = SparkContext(appName="DStream_QueueStream")
ssc = StreamingContext(sc, 2)  # reading every 2 seconds
```


```python
  
rddQueue = []
for i in range(3):
    rddQueue += [ssc.sparkContext.parallelize([j for j in range(1, 21)],10)]
inputStream = ssc.queueStream(rddQueue)
mappedStream = inputStream.map(lambda x: (x % 10, 1))
reducedStream = mappedStream.reduceByKey(lambda a, b: a + b)
reducedStream.pprint()

ssc.start()
#sleep(6)
ssc.stop(stopSparkContext=True, stopGraceFully=True)
```


```python

```
