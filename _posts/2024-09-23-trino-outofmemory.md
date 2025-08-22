---
layout: post
title: "Trino 메모리 누수: Hadoop FileSystem Cache의 함정"
date: 2024-09-23
categories: [Trino, Hadoop, Performance]
tags: [trino, hadoop, oom, memory-leak, filesystem-cache, troubleshooting]
---

Presto에서 Trino로 전환한 후 얼마 지나지 않아 예상치 못한 문제가 발생했다. 쿼리 요청이 증가하면서 워커 노드들이 메모리 부족(OOM)으로 인해 하나둘씩 셧다운되기 시작했고, 클러스터 전체가 불안정해지는 상황이 벌어진 것이다. 단순한 설정 문제일 것이라고 생각했지만, 문제의 근본 원인은 생각보다 훨씬 깊은 곳에 숨어있었다.

<!-- more -->

## 문제 발생

### 첫 번째 신호: 워커 노드들의 연쇄 셧다운

Trino 워커들이 워크로드를 처리할 때 OOM을 발생시키고 있었다. 다음은 Trino 워커의 JVM 옵션들이다.

![Trino JVM 설정](https://github.com/user-attachments/assets/2f4ba3c1-5e31-4d51-874f-b02dd0b06b81)
*Trino 워커의 JVM 옵션 설정 - jvm.config 파일*

클러스터 불안정의 원인이 명확히 OOM이었기 때문에, 워커들의 JVM 옵션에 `-XX:+HeapDumpOnOutOfMemoryError` 옵션을 활성화했다. 이제 OOM 발생 시 Trino 작업 디렉토리에 힙덤프가 생성된다.

![Heam Dumping](https://github.com/user-attachments/assets/bd7568fc-ec23-4824-9d0f-1f6e5873e08d)

힙덤프를 시작한다. 이 과정동안 이 노드는 out of service 상태가 된다.

### 힙덤프 생성의 딜레마

하지만 새로운 문제가 발생했다. 힙덤프를 생성하는 과정에서 또 다른 장애가 일어난 것이다. Trino 워킹디렉토리가 `/mnt`에 마운트된 EBS 내에 있어서, EBS 용량이 힙덤프보다 작을 경우 힙덤프 하나만으로도 디스크 사용률이 100%에 도달하게 된다. 예를 들어, 30GiB EBS 디스크가 마운트된 디렉토리에 128GiB 메모리 사양의 힙덤프가 저장되면 용량 부족으로 Trino 서버가 재시작되지 못한다.

```text
...
[29055.159592] systemd-journald[2054]: Failed to open system journal: No space left on device

[29055.161253] systemd-journald[2054]: Failed to open system journal: No space left on device

[29055.162798] systemd-journald[2054]: Failed to open system journal: No space left on device

[29055.164404] systemd-journald[2054]: Failed to open system journal: No space left on device
...
```
*30GiB EBS 디스크에 128GiB 메모리 사양의 힙덤프가 저장되면서 발생한 "No space left on device" 에러*

워커 서버의 메모리가 대부분 128GiB였기 때문에 힙덤프의 크기가 크고 생성 시간도 오래 걸렸다. 따라서 8GiB와 같이 상대적으로 작은 메모리를 가진 워커만 실험적으로 클러스터에 투입하여 힙 크기를 측정해보기로 했다.

## 추적: 문제의 정체를 찾아서

### Prometheus와 Grafana를 통한 모니터링

문제의 패턴을 파악하기 위해 각 노드의 리소스 사용량을 추적했다. Prometheus에서 지표를 수집하기 위해 다음과 같이 JVM 옵션을 추가했다:

```text
-javaagent:/usr/lib/trino/lib/jmx_prometheus_javaagent-1.0.1.jar=12345:/etc/trino/conf/jmx_exporter_config.yaml
```

`jmx_prometheus_javaagent.jar` 파일과 `jmx_exporter_config.yaml`은 EMR 부트스트랩을 통해 설치했다. 에이전트를 설치하고 Trino 서버를 실행하면 Java 기본 지표와 Trino 커스텀 지표를 수집할 수 있었다.

![Grafana Dashboard](https://github.com/user-attachments/assets/14c3111f-d01d-47b2-b294-b5862e0e37bd)
*다양한 Trino 지표들을 보여주는 Grafana 대시보드 - 노드 상태, 실행 중인 쿼리, 메모리 예약량, 실행 지연시간 등*

### 메모리 사용 패턴의 발견

Grafana를 통해 워커의 Old 영역이 매우 규칙적이고 점진적으로 점유되는 것을 관찰할 수 있었다. 결국 JVM은 OOM으로 인해 종료된다.

![G1 Old Gen](https://github.com/user-attachments/assets/c4614826-7d36-4658-ac9d-c022f85d8a8c)
*Old 영역이 계단식으로 점유되는 현상과 OOM으로 인한 JVM 종료 시점(이빨 빠진 영역은 JVM이 OOM으로 종료되고 힙덤프를 생성하는 시간)*

Old 영역이 계단식으로 점유되는 현상은 경험상 다음 두 가지 원인으로 추측할 수 있었다:
1. **메모리 누수**
2. **애플리케이션 캐시**

이제 힙덤프를 상세히 분석해본다.

### 힙덤프 분석 환경 구성

> 💡 **큰 힙덤프 분석은 MAT(Eclipse Memory Analyzer)를 사용한다.**

*힙덤프 분석을 위한 대용량 메모리 장비 설정 및 MAT 설정 방법*

- MAT 다운로드 후, 압축을 푼 후 `ParseHeapDump.sh` 스크립트 마지막에 `-vmargs -Xmx80g -XX:-UseGCOverheadLimit` 옵션을 추가(힙덤프 크기만큼 메모리 설정)
- 분석을 통해 생성된 인덱스 파일을 로컬 장비로 옮긴 후, MAT으로 열기

### MAT 분석 결과

![Leak Suspects Overview](https://github.com/user-attachments/assets/22104829-76b9-410a-a225-9220825c9295)
![MAT Analysis2](https://github.com/user-attachments/assets/ecaa29be-a15d-48bf-bbc4-1d60caa6d200)
*154,554개의 `org.apache.hadoop.conf.Configuration` 인스턴스가 메모리의 62.15%를 점유하고 있음을 보여주는 MAT 분석 결과*

MAT 분석 결과, Suspect 1을 해석하면 다음과 같다:

- **`org.apache.hadoop.conf.Configuration`** 인스턴스가 메모리 점유의 주된 원인
- **인스턴스 개수**: 154,554개
- **메모리 점유**: 16,135,726,184 bytes (약 15.4 GB, 62.15%)
- **클래스 로더**: `io.trino.server.PluginClassLoader`

근본 원인은 이 Configuration 객체들이 `java.util.HashMap$Node[]`에 의해 참조되고 있으며, 이 HashMap은 `org.apache.hadoop.fs.FileSystem$Cache` 객체가 참조하고 있다는 점이었다.

스레드 정보를 보면:
- **스레드 이름**: `20240904_071500_01656_d5p8v.1.5.0-548-61`
- **참조 객체**: `com.amazon.ws.emr.hadoop.fs.EmrFileSystem`
- **스레드 로컬 메모리**: 648,576 bytes (무시할 수 있는 크기)

정리하면, `FileSystem$Cache` 객체가 HashMap을 통해 수많은 `Configuration` 객체들을 참조하고 있어 메모리 누수가 발생하고 있었다.

## 문제의 근본 원인 추적

### 스택트레이스 분석

![](https://github.com/user-attachments/assets/051f7509-5618-4027-905c-cecb358fbf1d)
*Split을 Page로 변환하는 과정에서 발생한 OutOfMemory 스택트레이스*

스택트레이스는 Split으로부터 스트림을 열어 Page를 생성하는 단계에서 문제가 발생했고, 테이블 데이터를 읽는 과정에서 에러가 발생한것으로 추측할수있다. 주요 호출 스택은 다음과 같다:

- **93번 라인**: `io.trino.operator.ScanFilterAndProjectOperator$SplitToPages.process()` - Split을 Page로 생성
- **87번 라인**: `io.trino.plugin.hive.line.LinePageSourceFactory.createPageSource()` - 텍스트 파일을 읽어서 Reader 생성
- **83번 라인**: `io.trino.filesystem.hdfs.HdfsInputFile.newStream()` - 텍스트 파일의 InputStream 생성

`LinePageSourceFactory`를 사용한 것으로 보아 CSV나 JSON 포맷의 테이블을 읽는 중이었다(ORC 파일의 경우 `OrcPageSourceFactory`를 사용함).

### 코드 레벨 분석

이제 문제의 발생 지점을 찾았으니, Split으로 Page를 생성하는 `LinePageSourceFactory.createPageSource()` 메서드부터 자세히 살펴보자.

![LinePageSourceFactory](https://github.com/user-attachments/assets/415c0837-e3db-4974-8ab7-1e0b4aa9b346)
*LineReader를 생성하는 `LinePageSourceFactory.createLineReader()` 메서드 코드*

`LinePageSourceFactory.createPageSource()`에서는 입력파일(`inputFile`)을 받아 `lineReader`를 생성한다. `TextLineReaderFactory.createLineReader()`에서는 입력파일의 스트림을 생성한다.

![HdfsInputFile](https://github.com/user-attachments/assets/2c4b4d7f-6703-4fd6-8aa4-735b798db3b2)
*HDFS 파일시스템에서 파일을 여는 `HdfsInputFile.openFile()` 메서드*

`HdfsInputFile.openFile()`에서는 HDFS 파일시스템에서 입력파일을 연다. 파일을 열 때 파일시스템 객체가 필요하며, 이 파일시스템 객체는 `environment.getFileSystem()`에 인자로 전달하는 파일 경로(Path 객체)로부터 가져올수 있다.

![core-site.xml](https://github.com/user-attachments/assets/caed0e45-abca-49e6-9415-8d35a930ed5b)
*파일시스템과 URI 스킴 매핑을 보여주는 Hadoop core-site.xml 설정 파일*

파일 경로로 파일시스템을 식별할 수 있는 이유는, 파일 경로(URI)의 스킴(scheme)이 파일시스템과 매핑되어있는 유일한 식별자기 때문이다. 우리는 Amazon S3를 의존하기 때문에 `s3://bucket_name`으로 시작할것이고, 이는 EmrFileSystem과 매핑되어있다.

![FileSystem](https://github.com/user-attachments/assets/2c9904e0-7e6e-446a-8933-97e03ee68e2a)

### FileSystem.get() 메서드의 캐시 메커니즘

![FileSystem.get()](https://github.com/user-attachments/assets/8e584170-df58-4c77-8f13-ba49b18863a2)
*URI로부터 파일시스템을 가져오는 `FileSystem.get()` 캐시 메커니즘 구현*

인자로 전달하는 `path` 객체로부터 파일시스템을 가져온다. URI로부터 파일시스템을 가져올때 캐시를 사용하며, 캐시를 사용하는 이유는 `fs.$SCHEME.impl`의 설정에 정의된 클래스를 동적 로딩하는 과정이 꽤 비싼 연산이기 때문에 캐시를 사용하는 것으로 추측한다.

`fs.$SCHEME.impl.disable.cache`의 기본값이 `false`이므로 파일시스템을 `CACHE`로부터 조회한다. 여기서 테이블을 스캔할때, 스캔되는 파일 하나당 캐시를 한번 조회한다. 그러므로 이 `get()` 함수는 쿼리가 탐색하는 파일의 갯수만큼 호출된다.

### 캐시 키의 치명적 결함

![Cache.Key](https://github.com/user-attachments/assets/b9391faa-ff92-4ce7-ae5a-a38594faeb1b)
*캐시 조회를 위한 FileSystem Cache Key 클래스의 구현체*

`CACHE`를 조회하기 위해 키 객체를 생성한다. 이것 역시 파일 하나당 하나의 키가 생성된다. 동일한 테이블 데이터셋을 구성하는 각 파일에 대한 `Key` 객체의 멤버 변수 값을 예상해본다:

- **scheme**: `s3`
- **authority**: S3 버킷 이름 (테이블의 모든 파일이 동일한 버킷)
- **ugi**: `UserGroupInformation.getCurrentUser()`
- **unique**: `0`

`Key`의 생성자에 넘어오는 인자값 `uri`는 파일마다 다르고, `conf`는 동일하지만, 테이블의 파일들에 대한 멤버 변수 모두 동일한 값이기 때문에 동일한 키를 가질 것이라고 예상한다.

### 문제의 핵심: UserGroupInformation hashCode

이 Cache 클래스는 HashMap을 사용하여 캐시를 구현했고, HashMap은 `equals()`, `hashcode()`를 사용하여 키의 중복을 검사한다. `Key` 클래스의 `hashcode()`는 멤버 변수의 해시코드값을 조합하여 반환하도록 오버라이드되었고, 그 중에서 `ugi` 멤버 변수의 `hashcode()`는 다음과 같다.

![UserGroupInformation.hashcode()](https://github.com/user-attachments/assets/3f529f64-f48e-4622-bcde-e41ca8c4b8fe)
*`UserGroupInformation.hashcode()` 구현*

`System.identityHashCode()`는 VM에 따라 다르게 구현되어있다. Trino가 사용하는 Java Correto 17은 HotSpot VM(Open JDK)의 구현체다. HotSpot의 `System.identityHashCode()`은 호출할때마다 다른 값을 반환한다.

- [The Java System::identityHashCode method](https://www.objectos.com.br/blog/the-java-system-identity-hash-code-method.html), HotSpot VM의 해시코드 동작 설명
- [How does the default hashCode() work?](https://varoa.net/jvm/java/openjdk/biased-locking/2017/01/30/hashCode.html)

결국 파일마다 다른 키 값을 생성하는 것과 마찬가지이므로, 파일 하나당 캐시의 레코드를 하나씩 점유하게 되고 메모리 누수로 이어진다.

### 문제 발생 과정 정리

1. `TextLineReaderFactory` 객체 사용으로 JSON, CSV 같은 텍스트 포맷의 테이블 조회 시 메모리 사용량 증가
2. 파일 하나를 열기 위해 해당 파일 경로로부터 `FileSystem` 객체를 캐시에서 조회
3. 캐시 히트가 절대 발생하지 않아 파일 하나당 하나의 파일시스템 레코드가 `HashMap`에 누적

### 실제 쿼리 패턴 분석

문제를 일으키는 실제 쿼리를 확인했다:

```sql
SELECTservice_name,ip,level,count(*) AS log_count
FROM log_table -- CSV 포맷의 로그 테이블
WHERE yyyymmddhh = ?AND timestamp >= ?AND timestamp < ?AND service_name like 'server-%'
GROUP BY service_name, ip, level
ORDER BY service_name, ip, level, log_count
```

이 쿼리는:
- **5분마다 실행**
- **JSON 포맷의 1MB 이하 작은 파일들** 스캔
- 한 번에 **수백 개의 파일** 처리
- 1시간에 12번 동일한 파티션 조회

## 해결: 간단하지만 효과적인 솔루션

### Hadoop의 오래된 이슈

[key of FileSystem inner class Cache contains UGI.hascode which uses the default hascode method, leading to the memory leak with Proxy Users](https://issues.apache.org/jira/browse/HADOOP-12707)
*Hadoop 커뮤니티에서 오래전에 보고되었지만 설계상 이유로 수정되지 않은 이슈*

결국 이것은 Trino 문제가 아닌 Hadoop의 이슈였다. 오래전에 이슈로 보고되었지만 설계상의 이유로 수정되지 않았다. 많은 Hadoop 에코시스템 서비스들(Spark, Hive 등)이 여전히 메모리 누수를 일으킬 수 있다.

### 해결책: 캐시 비활성화

해결은 의외로 간단했다. HDFS 파일시스템 캐시 사용 여부 설정인 `fs.s3.impl.disable.cache`를 `true`로 설정하여 캐시를 비활성화했다.

### 결과: 극적인 개선

![G1 Old Gen After](https://github.com/user-attachments/assets/83c5b6a3-1261-46f6-b2ac-912fbc9d52cf)
![G1 Eden After](https://github.com/user-attachments/assets/a807deff-d069-49eb-924f-d576031720ff)
*캐시 비활성화 후 평화로워진 Old 영역과 바빠진 Young 영역의 모습*

캐시를 비활성화한 후:
- **Old 영역**: 평화로워짐, Major GC 미발생
- **Young 영역**: 더 바빠짐, Minor GC 빈번 발생

이는 예상된 결과로, 캐시를 사용하지 않아 매번 새로운 `FileSystem` 객체를 생성하므로 Young 영역 활동이 증가했지만, 메모리 누수는 완전히 해결되었다.

## 결론과 교훈

이번 문제 해결 과정에서 가장 큰 교훈은 문제의 근본 원인이 예상보다 훨씬 깊은 곳에 있을 수 있다는 점이었다. 단순한 Trino 설정 문제로 보였던 것이 실제로는 Hadoop 파일시스템 캐시의 설계 결함에서 비롯된 것이었다. `UserGroupInformation` 객체의 `hashCode()` 구현이 매번 다른 값을 반환하여 캐시가 전혀 작동하지 않는 미묘한 버그였다. 

운영 환경에서는 작은 파일들로 구성된 테이블 구조가 단순한 성능 문제를 넘어 메모리 누수까지 야기할 수 있으며, 상세한 모니터링과 힙덤프 분석이 문제 해결의 핵심이라는 것을 깨달았다. 앞으로는 알려진 안티패턴들을 사전에 방지하는 아키텍처를 설계하고, 간단한 설정 하나가 시스템 전체의 안정성을 좌우할 수 있다는 점을 항상 염두에 두고 운영해야겠다.







