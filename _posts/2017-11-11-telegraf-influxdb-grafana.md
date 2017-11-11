---
layout: default
title: Legacy System 의 Event Log 및 시스템 상태 모니터링 하기
category: post
author: Jae
comments: true
---

# Legacy System 의 Event Log 및 시스템 상태 모니터링 하기

> 본 포스트의 내용은 개인의 공부 및 정리를 목적으로 작성한 것 입니다.
> 혹시 사실과 다르거나 잘못된 내용이 있으면 댓글로 남겨주시면 수정하겠습니다.

## 나는 왜 이 작업을 시작하였나.

팀에서 운영중인 서비스 중 Legacy System은 다음과 같은 방법으로 모니터링을 하고있었다.

1. 모니터링 하고자하는 Log를 log4j의 `SyslogAppender`를 사용하여 Flume에 전송
2. Flume을 통해서 Log 데이터를 JSON 형태로 변환
3. 정제된 데이터를 ElasticSearch에 저장
4. 저장된 데이터는 ElasticSearch에 의해서 Indexing 되고 Kibana를 통해 검색하거나 DashBoard를 구성하는데 이용

하지만 신규 서비스들이 추가되고 Legacy System의 Log 양도 점점 늘어나다보니 Log의 목적이나 데이터의 성격에 맞게 Log를 나누어 저장해야 할 필요성이 생겼다.

그 중 주기적으로 누적되는 Status 성 Log들은 요즘 Time-Series Database 진영에서 가장 핫하다고 할 수 있는 InfluxDB에 저장하기로 했다. (정확히는 이미 팀에서 InfluxDB를 운영중이었고, 나는 숟가락을 얹기로 했다.)


## Telegraf - InfluxDB - Grafana

Telegraf - InfluxDB - Grafana 조합은 Time-Series Database로 InfluxDB를 사용하는 많은 사람들이 주로 사용하는 조합이다.

> Telegraf와 InfluxDB는 Influxdata 에서 주도적으로 개발 중이고, 같은 회사에서 개발한 Chronograf라는 Grafana와 비슷한 놈이 있는데 사람들은 Grafana를 주로 이용하는 것 같다.. Grafana가 더 좋은가..?

각 컴포넌트에 대해서 조금 구체적으로 알아보면..

#### Telegraf

###### What is Telegraf ?

InfluxDB와 마찬가지로 Go언어로 개발되었다.
다양한 종류의 Plugin이 존재하며 설정 파일의 변경 만으로 쉽게 사용할 수 있다는 장점이 있다. 

Plugin은 크게 Input/Output Plugin으로 구분된다. 
Input Plugin은 Cpu 및 Disk 사용량과 같은 데이터 Source를 수집하는 역할을 한다. Output Plugin은 수집한 데이터를 저장하는 역할을 한다. 저장 매체로는 InfluxDB, Graphite 같은 Time-Series Database 뿐만아니라 일반 파일이나 Kafka같은 메시징 시스템을 사용할 수 있다. 

###### Why telegraf ?

Telegraf는 다음과 같은 기능들을 자동으로 제공한다.

* `host` tag와 `timestamp`가 자동으로 추가된다. (Option을 통해서 Drop 시키는것도 가능하다.)
* Output stream의 저장에 실패하는 경우, Telegraf Agent가 자동으로 재시도한다.
* 로그 전송에 Interval을 설정 할 수 있다. (버퍼링을 통해서 네트워크 트레픽을 줄일 수 있다.)
* 다양한 Input/Output Plugin 존재한다. (2017년 11월 현재도 다양한 Plugin들이 추가되고 있다!!)

###### Push or Pull !!

각 서버의 모니터링 데이터의 수집을 위해서는 push 방식과 pull 방식을 고려할 수 있다.

* Push: 각 서버에 agent로서 Telegraf나 FluentD를 설치하거나 또는 자체 개발한 프로그램 등을 백그라운드로 실행하여 모니터링 서버로 데이터를 전송한다.
* Pull: 모니터링 서버(influxdb가 설치된)에서 각 서버로 접속하여 모니터링 데이터를 가져온다.

Push 방식의 구성에서는 서버가 추가 될 때마다 매번 해당 서버에 Agent를 설치해줘야 하는 번거로움이 있다. 하지만 모니터링 서버를 통해서 개별 서버에 접근할 수 없기 때문에 보안상 안전하고 모니터링 서버에 수집 부하가 집중되는 문제를 피할 수 있다.
반대로, Pull 방식으로 구성할 경우 모니터링 대상 서버가 추가될 때마다 agent용 프로그램들을 설치할 필요 없이 간단히 설정만 추가하면 되는 편리함은 있다. 그러나 모니터링 서버가 보안 위험에 노출될 경우 다른 서버들로 접속이 가능한 위험이 있다. 특히 모니터링 대상 서버가 DB서버라면 심각한 보안 위험 요소가 될 수 있다. 또한 모니터링 서버에 수집 부하가 집중되는 문제도 있을 수 있다.

운영중인 System의 구성이 하나의 InfluxDB가 전체 서비스의 Metric을 수집하고 있기 때문에 Pull 방식을 사용할 경우, 모니터링 서버에 트레픽이 집중되는 문제가 발생 할 우려가 있다.
따라서 대부분의 서비스가 각 Source에 Telegraf를 Agent로 설치하여 InfluxDB로 Push 하는 방식으로 모니터링 시스템을 구성하였고 나 역시 Push 방식을 선택하였다.

#### InfluxDB

###### What is Influxdb ?

대표적인 Time-Series Database 이다.

 > Time-Series Data란 시간의 흐름에 따라 변하는 값들을 의미하며, 예로 OS 및 서비스들의 Status 정보 등을 들 수 있다.

###### Why InfluxDB ?

다양한 Time-Series DataBase가 존재하지만 기존에 많이 사용되던 RRDTool이나 Graphite를 제치고 당당히 업계 1, 2위를 다투고 있다. (사실 서로 지들이 좋다고 우기는 것 같다.. 포스팅 하는 2017년 11월 기준 google에 "time series database benchmark"로 검색하면 요즘은 [DalmatinerDB](https://dalmatiner.io/)가 제일 핫 한 모양이다.)
Graphite와 간단히 비교하면 Graphite는 Python으로 개발되었고 메트릭 하나당 파일 한개씩 생성하는 구조로 동시 I/O가 많은 상황 취약하다. 즉, N개 메트릭 저장시 N개의 파일 I/O를 일으켜야 하는 방식이라 동시에 많은 수의 메트릭을 저장한다면 클러스터를 증설해야한다.
이에 비해서 InfluxDB는 Go언어로 개발되었고 LSM(Log Structured Merge) Tree를 개량하여 시계열 데이터 저장에 최적화되도록 자체 개발한 TSM(Time Structured Merge) Tree를 스토리지 엔진으로 사용한다. 따라서 동시성 및 I/O처리 성능이 뛰어나며 압축 알고리즘도 적용하여 저장 용량 효율면에서 뛰어나다.

###### RP(Retention Policy) & CQ(Countiuous Query)
[TODO]

#### Grafana

###### What is Grafana ?

Graphite, InfluxDB, Elasticsearch 등 데이터베이스에 저장된 데이터를 시각화, 모니터링 하기위한 툴이다.

###### Why Grafana ?

Data Source와의 연동 후 기본적인 Panel 몇 가지와 Template 기능 정도만 익히면 손쉽게 멋진 스타일의 모니터링 툴을 디자인 할 수 있다.


## Service Metric 의 수집

Telegraf는 다양한 [Input plugin](https://github.com/influxdata/telegraf#input-plugins)을 제공하는데, Metric 수집을 위해서는 다음과 같은 Service plugin 을 제공한다.

* [kafka_consumer](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/kafka_consumer)
* [logparser](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/logparser)
* [socket_listener](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/socket_listener)
* [zipkin](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/zipkin)

이번에 작업을 진행한 Service는 log4j의 `SyslogAppender`를 통해서 Flume으로 Log를 전송하는것 이외에 추가로 `DailyRollingFileAppender`를 통해서 Local에도 Log를 기록하고 있었기 때문에 logparser plugin을 이용해서 Metric을 추출, InfluxDB에 저장하기로 하였다.

#### Telegraf logparser input plugin

[Telegraf logparser plugin](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/logparser)은 telegraf 설정 파일에 다음과 같이 plugin 설정을 추가함으로써 사용할 수 있다.

```bash
[[inputs.logparser]]
	files = [“/var/log/apache/access.log”]
	from_beginning = false
	
	[inputs.logparser.grok]
		patterns = [“%{COMBINED_LOG_FORMAT}”, “%{MY_CUSTOM_PATTERN}”]
		measurement = “apache_access_log”
		custom_pattern_files = []
		custom_patterns = ‘’’
			MY_CUSTOM_PATTERN (?:%{DATA})
		‘’’
		timezone = Local #UTC
```

* files: parsing 할 log file의 경로
* from_beginning: 이미 존재하는 파일을 처음부터 parsing 하고 싶을 때 true로 설정한다.
* patterns: parsing에 이용할 패턴을 추가한다.
	* for문을 이용해서 모든 패턴을 검사하도록 구현되어 있기때문에 패턴을 추가하면 processing time이 증가한다. O(N)
	* logparser당 하나의 패턴을 가지는 것을 권장한다.
* measurement: meaurement의 이름을 지정한다.
	* logparser 마다 하나의 measurement 만 설정 가능하기 때문에 패턴마다 measurement를 다르게 설정할 수 없다. 패턴마다 measurement를 다르게 설정하고 싶다면 logparser plugin을 여러개 설정한다.
* custom_pattern_files: 사용자 정의 패턴을 파일을 이용해서 추가할 수 있다.
* custom_patterns: 사용자 정의 패턴을 문자열로 추가할 수 있다.
* timezone: timezone을 설정할 수 있다.

>  Grok
>
> grok은 비정형 log 데이터를 파싱하여 정형 데이터로 만들어 주는 라이브러리이다.
> 기본 문법은 `%{SYNTAX:SEMANTIC}`으로 SYNTAX에는 텍스트와 일치하는 패턴이, SEMANTIC에는 텍스트를 식별할 key의 이름을 넣는다.
> SYNTAX는 Regex를 지원하기 때문에 Regex를 이용하여 log를 파싱할 수 있다.

#### Troubleshooting for logparser

> logparser의 안정성이 아직 믿을만한 수준은 아닌듯 하니 사용시 주의가 필요하다. 

서비스 배포 후 이슈가 발생했다.
log4j 의 `DailyRollingFileAppender`에 의해서 Log가 rolling 되면, logparser 중 일부가 랜덤으로 Drop 되는 현상이 발견되었다.
Github Issue를 찾아보면 관련해서 꽤 많은 Issue 들이 존재하는 것을 알 수 있는데 대충 보면 대부분 Close되어 있다. (속았다... 젠장..)
다른 이슈로 판단해서 꽤 시간을 날리긴 했는데.. (하나의 Log 파일을 다수의 logparser plugin이 사용하는 경우 문제가 있는것으로 의심했으나 테스트 및 소스코트 분석 결과 정상 동작함을 확인했다.) 
결론은 Log 파일의 감시를 위해서 `inotify`를 사용하는 logparser의 구현에 다소 문제가 있는듯 하다..
정리하면 다음과 같다.

* 현상: logrotate 후, logparser plugin 중 일부가 정상적으로 동작하지 않음
	* github issue: https://github.com/influxdata/telegraf/issues/2847
* 해결 방법: Telegraf 1.5 버전 이후 polling 방식으로 log의 변경을 감시할 수 있는 기능 추가
	* github issue: https://github.com/influxdata/telegraf/pull/3213


## 수집된 Metric 저장

#### Telegraf influxdb output plugin

수집된 Metric은 influxdb output plugin을 사용해서 InfluxDB에 저장할 수 있다.
influxdb output plugin의 설정은 다음과 같다.

```bash
[[outputs.influxdb]]
	urls = [“http://<influxdb’s IP address>:8086”]
	database = “SystemMetric”
	retention_policy = “one_year”
```

* urls: influxdb instance에 접근 가능한 주소
* database: metric을 저장할 database
* retention_policy: retention poicy의 이름
	* retention policy는 사전에 정의되어 있어야 사용할 수 있다.
	* 비워놓는 경우 database에 default로 설정된 retention policy 정책을 사용한다.

#### Telegraf Output Data Format

Telegraf는 3가지 [output data format](https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_OUTPUT.md#telegraf-output-data-formats)을 지원한다.

* InfluxDB Line Protocol
* JSON
* Graphite

Default 설정은 InfluxDB Line Protocol 이며, 기본 형태는 다음과 같다.

```bash
measurement_name[,tag1=val1,...]  field1=val1[,field2=val2,...]  [timestamp]
```

InfluxDB Line Protocol 형태의 Output Data는 influxdb output plugin에 설정된 InfluxDB로 전송되어 저장된다.

> Data format을 변경하기위해서는 `[[outputs.file]]` plugin의 `data_format` option에 원하는 data foramt의 이름을 명시해주면 된다.


## InfluxDB에 저장된 데이터 확인하기

#### InfluxDB CLI/Shell

InfluxDB는 [CLI/Shell](https://docs.influxdata.com/influxdb/v1.3/tools/shell/) `influx`를 제공한다.
`influx`를 이용하면 InfluxDB에 데이터를 쓰거나, Query를 사용해서 데이터를 읽는 등의 동작이 가능하다.

다음은 간단한 사용 예를 보여준다. 각 명령어에 대한 결과는 생략 하였다.

```bash
$ influx
> SHOW DATABASES
> SHOW MEASUREMENTS
> CREATE RETENTION POLICY one_year ON mydatabase DURATION 365d REPLICATION 1 DEFAULT
> SHOW RETENTION POLICY ON mydatabase
> USE mydatabase
> select * from mymeasurement limit 10;
```

#### Grafana

Grafana는 위에서 설명한 것 처럼 Graphite, InfluxDB, Elasticsearch 등 데이터베이스에 저장된 데이터를 시각화, 모니터링 하기위한 툴이다.
Grafana에 데이터 소스로서 InfluxDB를 추가한 후 적절한 query를 사용해서 데이터 받아 그래프, 테이블 등으로 출력할 수 있다.


## 더 해볼 것

* Grafana Alerting 기능 이용해 보기
* Kapacitor를 이용해서 InfluxDB로 들어오는 데이터를 실시간으로 모델링 해보기
* logparser polling 기능 적용하기


## References

* http://www.popit.kr/influxdb_telegraf_grafana_1
* http://www.popit.kr/influxdb_telegraf_grafana_2
* http://www.popit.kr/influxdb_telegraf_grafana_3
* https://docs.influxdata.com/influxdb/latest
* https://docs.influxdata.com/telegraf/latest
* http://docs.grafana.org/