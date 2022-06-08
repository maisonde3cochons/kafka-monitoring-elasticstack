## [Elastic Stack을 이용한 Apache Kafka Monitoring 방법]

<br>

### [Usage & Advantage]

<br>

> #### Elastic Stack을 통해서 다양한 유형의 데이터를 별도 코딩 없이 수집/저장/검색/시각화까지 빠르고 안정적으로 구현 가능

- 다양한 데이터 수집 제어 : LogStash, File Beats 등 데이터 유형별 수집 기술, Beats와 LogStash간 연결을 통한 데이터 파이프라인
- 빠른 검색 서비스 : 검색엔진인 elasticsearch를 활용한 빠른 검색, 분산 구조로 데이터 확장 및 유실방지
- 풍부한 실시간 차트 : 코딩 없이 실시간 차트 생성(bar, pie, area, line, map...), 업무 목적별 dashboard 구성

<br>

### [환경 구성도]

![image](https://user-images.githubusercontent.com/30817824/172549387-5834d324-4abd-4d17-ab08-9693103b016f.png)

(refer: https://github.com/freepsw/kafka-metrics-monitoring)

<br><br>

----------------------------------------------------

<br>

> ##  STG.01 GCP VM Instance 생성

<br>

### 사전에 GCP 환경에 VM Instance를 생성하고 Java/LogStash 등을 설치한다

<br>

> #### 아래 내용 참고하여 GCP VM Instance 구성( + Java + LogStash 설치)
- https://github.com/maisonde3cochons/kafka-monitoring-jconsole
- https://github.com/maisonde3cochons/kafka-monitoring-producer-jconsole
- https://github.com/maisonde3cochons/kafka-monitoring-consumer-jconsole
 
 
 ##### Kafka producer(LogStash)를 실행하여 tracks.csv의 data를 전송하고 있으며, 동일 서버 내의 Consumer(Logstash)가 해당 data를 consume 하고 있는 상태이다
 
<br><br>

> ## STG.02. Install & Configure

<br>

### 01. Kafka-monitoring 서버 접속 및 limit 설정 (Java 11 installed)

### Elasticsearch를 실행하기 위해서 필요한 OS 설정이 충족되지 못하여 발생하는 오류 (이를 해결하기 위한 설정 변경)
#### 오류1) File Descriptor 오류 해결
- [1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
- file descriptor 갯수를 증가시켜야 한다.
- 에러 : [1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
- https://www.elastic.co/guide/en/elasticsearch/reference/current/setting-system-settings.html#limits.conf

```
> sudo vi /etc/security/limits.conf
# 아래 내용 추가 
* hard nofile 70000
* soft nofile 70000
root hard nofile 70000
root soft nofile 70000

# 적용을 위해 콘솔을 닫고 다시 연결한다. (console 재접속)
# 적용되었는지 확인
> ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 59450
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 70000  #--> 정상적으로 적용됨을 확인함
```

#### 오류2) virtual memory error 해결
- 시스템의 nmap count를 증가기켜야 한다.
- 에러 : [2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
- https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html
```
# 0) 현재 설정 값 확인
> cat /proc/sys/vm/max_map_count
65530

# 아래 3가지 방법 중 1가지를 선택하여 적용 가능
# 1-1) 현재 서버상태에서만 적용하는 방식
> sudo sysctl -w vm.max_map_count=262144

# 1-2) 영구적으로 적용 (서버 재부팅시 자동 적용)
> sudo vi /etc/sysctl.conf
# 아래 내용 추가
vm.max_map_count = 262144

# 1-3) 또는 아래 명령어 실행 
> echo vm.max_map_count=262144 | sudo tee -a /etc/sysctl.conf

# 3) 시스템에 적용하여 변경된 값을 확인
> sudo sysctl -p
vm.max_map_count = 262144
```

<br>

### 02. install & configure elasticsearch
#### Download the elasticsearch binary file
```
curl -LO https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.10.2-linux-x86_64.tar.gz
tar -xzf elasticsearch-oss-7.10.2-linux-x86_64.tar.gz
rm -rf elasticsearch-oss-7.10.2-linux-x86_64.tar.gz
```

#### Configure elasticsearch (master host 설정)
- conf/elasticsearch.yml
- master host 설정 (cluster.initial_master_nodes) : Master Node의 후보를 명시하여, Master Node 다운시 새로운 Master로 선출한다.
- 설정하지 않을 경우, elasticsearch 실행 시 아래 오류 발생
```
[3]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```
- 따라서 아래와 같이 설정 필요
```
> cd ~/elasticsearch-7.10.2
> vi config/elasticsearch.yml

## Master Node의 후보 서버 목록을 적어준다. (여기서는 1대 이므로 본인의 서버 정보만)
cluster.initial_master_nodes: ["kafka-monitoring"]

## 위의 설정에서 IP를 입력하면, 아래 오류 발생
    - skipping cluster bootstrapping as local node does not match bootstrap requirements: [34.64.85.xx]
    - master not discovered yet, this node has not previously joined a bootstrapped (v7+) cluster, and [cluster.initial_master_nodes] is empty on this node
```

#### Run the elasticsearch 
```
cd ~/elasticsearch-7.10.2/

## 아래 2가지 방식으로 실행 가능 

### 1) foreground로 실행시 명령어
./bin/elasticsearch 

### 2) background(daemon)으로 실행시 명령어 (실행된 프로세스의 pid 값을 elastic_pid 파일에 기록)
./bin/elasticsearch -d -p elastic_pid

### daemon으로 실행시 서비스 종료하려면 아래 명령어 실행 (기록된 pid 값을 읽어와서 프로세스 종료)
pkill -F elastic_pid
```

#### elasticsearch 정상 동작 확인
```
curl -X GET "localhost:9200/?pretty"
```

<br>

### 03. Install the kibana
#### Download the kibana binary file
```
cd ~
curl -OL https://artifacts.elastic.co/downloads/kibana/kibana-oss-7.10.2-linux-x86_64.tar.gz
tar xvf kibana-oss-7.10.2-linux-x86_64.tar.gz 
rm -rf kibana-oss-7.10.2-linux-x86_64.tar.gz
cd kibana-7.10.2-linux-x86_64/

## 외부에서 접속 할 수 있도록 설정한다. 
> vi config/kibana.yml

## 아래 내용 추가 (0.0.0.0은 모든 ip에서 접근이 가능한 설정. 운영환경에서는 특정 IP로 제한 필요)
server.host: "0.0.0.0"
```

#### Run the kibana service
```
cd ~/kibana-7.10.2-linux-x86_64

## foreground 실행
bin/kibana

## background 실행
nohup bin/kibana &

```

#### [ ERROR 유형 1 ] Port 5601 is already in use. Another instance of Kibana may be running!
- kibana가 현재 실행중. (설정이나 재시작이 필욯지 않다면 현재 실해 중인 kibana 사용)
- 재시작이 필요하다면, 아래와 같이 process id를 찾아서 해당 process를 종료한 후, kibana 재시작
```
## 마지막 칼럼의 16998이 process id (process id는 개인별로 다르게 표시됨)
netstat -anp | grep 5601
tcp        0      0 0.0.0.0:5601            0.0.0.0:*               LISTEN      16998/bin/../node/b

## process 종료
kill -9 16998
```

#### [ ERROR 유형 2 ] Unable to connect to Elasticsearch. Error: Request Timeout
- ElasticSearch가 현재 실행중이지만, 위에서 설정한 Master Node의 후보 서버 목록에 오타가 있을 수 있다

```
cluster.initial_master_nodes: ["kakfa-monitoring"]
=>
cluster.initial_master_nodes: ["kafka-monitoring"]
```

<br>

### 04. GCP VPC Network Firewall 설정 (5601, 9000 port 허용)

<br>

![image](https://user-images.githubusercontent.com/30817824/172518925-0a960d8a-8a56-4d92-9af6-bf24231fcf53.png)

![image](https://user-images.githubusercontent.com/30817824/172541307-5011100c-2428-4d12-aa07-93d52feeb89d.png)

<br>

#### Kibana 접속 확인

![image](https://user-images.githubusercontent.com/30817824/172572049-ba31a01c-239c-42d5-8006-e966c95544aa.png)


<br>

### 05. Install the logstash
#### Install the logstash and run
```
cd ~
curl -OL https://artifacts.elastic.co/downloads/logstash/logstash-oss-7.10.2-linux-x86_64.tar.gz
tar xvf logstash-oss-7.10.2-linux-x86_64.tar.gz
rm -rf logstash-oss-7.10.2-linux-x86_64.tar.gz
cd ~/logstash-7.10.2

## install jmx plugin
bin/logstash-plugin install logstash-input-jmx
```

<br>

### 06. JMX metric 설정 (kafka broker 용)
```
mkdir ~/jmx_conf
```

#### vi ~/jmx_conf/broker01.conf
```
{
  "host" : "broker-01", 
  "port" : 9999,
  "alias" : "broker01",
  "queries" : [
  {
    "object_name" : "kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec",
    "attributes" : [ "OneMinuteRate" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec",
    "attributes" : [ "OneMinuteRate" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec",
    "attributes" : [ "OneMinuteRate" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions",
    "attributes" : [ "Value" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce",
    "attributes" : [ "Mean" ],
    "object_alias" : "${type}.${name}"
  },
  {
    "object_name" : "kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer",
    "attributes" : [ "Mean" ],
    "object_alias" : "${type}.${name}"
  }
 ]
}
```



<br>

### 07. logstash config 설정 
- broker01.conf에 정의된 metric 들을 수집하여, 
- elasticsearch로 전송하는 logstash 설정
```
mkdir ~/logstash_conf
```

#### vi ~/logstash_conf/logstash_jmx.conf

```
input {
 jmx {
  path => "/home/${USER}/jmx_conf"
  polling_frequency => 1
 }
}

output{
 stdout {
  codec => rubydebug
 }
 elasticsearch {
   hosts => "localhost:9200"
   index => "kafka_mon"
 }
}
```

```
cd ~/logstash-7.10.2/
bin/logstash -f ~/logstash_conf/logstash_jmx.conf
```
#### [ ERROR 유형 ] Connection Refused localhost:9999 java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.ServiceUnavailableException
- kafka 기동 시 앞에 JMX_PORT 설정 안 했을 수 있다

![image](https://user-images.githubusercontent.com/30817824/172591060-2a5e3063-0c84-4606-94af-11a16c30a1ae.png)

<br>


#### 생성된 index (kakfa_mon) 확인 
```
> curl -X GET "localhost:9200/_cat/indices/"
yellow open kafka_mon 6ULkg30FT7aQuxAI7ua7yg 1 1 18 0   32kb   32kb
```
<br>


### 08. Web Browser에서 접속 확인

```
http://<External IP>:9000
```
