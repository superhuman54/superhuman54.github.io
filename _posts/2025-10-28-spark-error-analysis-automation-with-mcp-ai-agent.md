---
layout: post
title: "Spark History MCP + AI Agent로 Spark 분석 자동화하기"
date: 2025-10-28 12:00:00 +0900
categories: [Spark, AI, Automation]
tags: [spark, mcp, ai-agent, n8n, openai, sparklistener, debugging, automation, emr, troubleshooting]
author: K4N
description: "SparkListener와 MCP(Model Context Protocol)를 활용하여 Spark 작업 실패를 자동으로 감지하고 AI가 원인을 분석하여 해결책을 제시하는 시스템을 구축한 경험을 공유합니다."
keywords: "spark, mcp, model context protocol, ai agent, n8n, openai, sparklistener, debugging, automation, emr, spark history server, troubleshooting"
published: false
---

우리 팀은 매일 수십 개의 Spark 배치 작업을 운영하고 있다. 추천 모델 학습, 사용자 행동 데이터 집계, 데이터마트 생성, 사용자 추천 데이터 추출 등 다양한 ETL 파이프라인이 Amazon EMR 클러스터에서 돌아가고 있다. 그런데 이 Spark 작업들이 실패하면... 정말 머리가 아프다.

"어? 새벽 3시에 돌린 작업이 실패했네? 로그 보러 가야겠다..."

"Stage 47이 실패했는데 왜 실패했지? Executor 로그를 뒤져봐야 하나?"

"Data Skew인가? OOM인가? 아니면 설정 문제인가?"

<!-- more -->

이런 상황에서 Spark UI, CloudWatch Logs, Amazon EMR 콘솔을 오가며 몇 시간씩 디버깅하는 건 일상다반사였다. 특히 주말이나 휴일 새벽에 장애가 발생하면... (더 이상 말하지 않겠다)

## 문제: Spark 디버깅의 고통

### 1. 복잡한 데이터 파이프라인

우리 팀에서 운영하는 Spark 작업은 다음과 같은 특징이 있다:

- 하루 수십 GB의 사용자 음원 청취 로그 처리
- 긴 Stage들로 구성된 DAG
- 다양한 데이터 소스 Join (S3, RDS, MongoDB..)

### 2. 빈번한 에러 발생

운영 중 자주 마주치는 에러들:

- **OOM (Out of Memory)**: Executor 메모리 부족으로 작업 실패
- **Data Skew**: 특정 파티션에 데이터 집중으로 인한 타임아웃
- **Shuffle Spill**: 과도한 Shuffle로 인한 성능 저하
- **네트워크 이슈**: S3 read timeout, connection refused

### 3. 긴 트러블슈팅 프로세스

문제 발생 후 트러블슈팅 과정을 구분해서 나열하자면 다음의 순서로 진행한다.

```
[문제 발생] MSP의 알림으로 장애 인지
↓
[1단계] (VPN 접속 후) Airflow Task 확인
↓
[2단계] Spark History UI에서 실패한 Stage 찾기
↓
[3단계] Driver, Executor 로그 검색
↓
[4단계] Stage 메트릭 분석 (Shuffle, GC, Memory), 필요하다면 코드와 비교
↓
[5단계] 원인 추정 및 해결책 도출
↓
[6단계] 설정 변경 후 재실행
```

이 과정이 **평균 1시간**, 복잡한 경우 **그 이상** 소요되었다.

더 큰 문제는 **Spark 내부 동작에 대한 깊은 이해**가 필요하다는 점이었다. Lazy Evaluation, Physical Plan, Catalyst Optimizer 등을 이해하지 못하면 근본 원인을 찾기 어려웠다. 더군다나, 팀 내부에서 MSP 장애 알림 수신 담당자를 주기적으로 교체하는데 그 담당자의 직무가 데이터 엔지니어가 아니거나, 신규 입사자 일경우 파악하기가 쉽지 않았다.

## AI가 대신 분석해주면 어떨까?

어느 날 내가 내 옆 동료한테 이런 아이디어를 제안했다.

> "우리가 매번 Spark UI 보고, 로그 뒤지고, 메트릭 분석하는데... GPT한테 물어보면 안 될까?"

처음엔 반신반의했지만, AWS에서 [Spark History Server MCP](https://aws.amazon.com/blogs/big-data/introducing-mcp-server-for-apache-spark-history-server-for-ai-powered-debugging-and-optimization/)를 오픈소스로 공개했다는 소식을 접하고 본격적으로 프로젝트를 시작했다.

### 내가 만들고 싶었던 것

1. **Spark 작업이 실패하면 자동으로 감지**
2. **AI가 Spark History Server 데이터를 조회하여 원인 분석**
3. **분석 결과를 Slack으로 전송**
4. **해결책까지 구체적으로 제시**

이렇게 하면 새벽에 에러가 발생해도 아침에 출근해서 Slack만 보면 어느정도 원인과 해결책을 알 수 있지 않을까?

## 해결 방안: SparkListener + n8n AI Agent

나는 다음과 같은 아키텍처를 설계했다.

![Spark Error Analysis Architecture](/assets/images/spark-error-analysis/ai_agent_arch.png)

### EMR 영역: Spark 에러를 실시간으로 감지

왼쪽의 파란색 EMR 영역은 실제 Spark 작업이 실행되는 공간이다. 이 영역의 핵심은 **Spark와 EventListener의 결합**이다. 이 리스너는 Spark 작업이 실행되는 동안 모든 이벤트를 실시간으로 모니터링한다(SparkListener에 대해 이론적으로만 알고있었던터라 실제 구현에서 애를 좀 먹었다..).

EMR 영역 아래쪽에는 **Spark History Server**가 있고, 나중에 AI Agent가 MCP 서버를 통해 필요한 데이터를 이 서버에서 가져와 분석한다.

### AI Workflow 영역: 에러를 분석하고 해결책을 제시

오른쪽의 초록색 AI Workflow 영역은 [n8n](https://n8n.io/)으로 구성된 자동화 파이프라인이다. Spark의 EventListener가 Stage 실패를 감지하면 이 주소로 JSON 데이터를 전송한다.

나는 OpenAI의 GPT-4o-mini 모델을 사용했는데, 이 모델은 빠른 응답 속도와 저렴한 비용이 장점이면서도 Spark 에러 분석에는 충분한 품질을 제공한다. AI 에이전트는 *"j-2AXXXXXXGAPLF 클러스터의 application_1761710205960_0056 애플리케이션 에러 원인을 분석해주고 해결책을 자세히 제시해줘"*(물론 실제 프롬프트는 훨씬 복잡하다..)라는 자연어 프롬프트를 받아서 분석을 시작한다.

AI 에이전트의 아래는 **Spark History MCP Server**가 연결되어 있다. AWS에서 오픈소스로 공개한 이 MCP 서버는 Spark History Server의 REST API를 AI가 자연어로 호출할 수 있게 해주는 브릿지 역할을 한다.

```
[EMR 영역]                      [AI Workflow 영역]
SparkListener                   HTTP Webhook
    ↓                               ↓
에러 감지 & 전송  ────────────→     AI Agent (GPT-4o-mini)
    ↓                               ↓
Spark History Server  ←────────  MCP Server
                                    ↓
                                Slack 알림
```

## 구현 과정

### 1. SparkListener 개발

가장 먼저 내가 해결해야 했던 문제는 "어떻게 Python으로 Java 인터페이스를 구현할 것인가?"였다. 개인적으로는 Scala로 작성하고 싶었지만, 팀의 주언어가 Python이라 유지/보수를 위해 Python으로 작성할 수밖에 없었다. Spark는 Scala와 Java로 작성되어 있고, SparkListener도 엄연히 Java 인터페이스다. 그런데 PySpark는 내부적으로 **Py4J**라는 라이브러리를 사용해서 Python과 JVM을 연결한다. 이 Py4J가 제공하는 특별한 패턴을 사용하면 Python 클래스를 Java 인터페이스처럼 보이게 만들 수 있다.

핵심은 **Java 클래스 선언**을 Python 안에 만드는 것이다. Python 클래스 안에 `class Java`라는 내부 클래스를 만들고, 그 안에 `implements = ["org.apache.spark.scheduler.SparkListenerInterface"]`라고 선언하면 된다. 이렇게 하면 Py4J가 이 Python 객체를 Java 쪽에서 SparkListener로 인식하게 된다.

```python
class JSparkListener:
    """Py4J를 통해 Java SparkListenerInterface를 구현"""
    def __init__(self, py_listener: SparkListener):
        self.py_listener = py_listener
    
    def onStageCompleted(self, jevent):
        """Java 이벤트를 Python으로 전달"""
        self.py_listener.onStageCompleted(jevent)
    
    # ... 생략 ...
    
    class Java:
        """Py4J 인터페이스 구현 선언"""
        implements = ["org.apache.spark.scheduler.SparkListenerInterface"]
```

### 2. SparkContext에 리스너 등록

리스너를 구현했으면 이제 Spark에 등록해야 한다. 여기서도 Py4J의 도움이 필요하다. PySpark의 `SparkContext`는 Python 객체처럼 보이지만 실제로는 내부에 Java의 SparkContext를 가지고 있다. 이 Java SparkContext에 접근하려면 `spark.sparkContext._jsc.sc()`라는 경로를 타고 들어가야 한다.

```python
from pyspark.sql import SparkSession
from spark_error_listener import ErrorInterceptorListener

# Spark Session 생성
spark = SparkSession.builder.appName("MyJob").getOrCreate()

# 리스너 등록
listener = ErrorInterceptorListener()
spark.sparkContext._jsc.sc().addSparkListener(listener._jlistener)
```

### 3. ZIP 파일로 패키징하고 배포

리스너를 구현하고 등록하는 코드를 작성했으면 이제 배포 단계다. PySpark의 `--py-files` 옵션은 Python 패키지를 배포할 수 있게 해주는데, 여기서 중요한 점은 **반드시 ZIP 파일이어야 한다**는 것이다. Wheel(`.whl`) 파일은 지원하지 않는다. 내가 만든 SparkListener ZIP 파일은 S3에 업로드했고, 다음과 같이 사용하게 된다.

```bash
# spark-submit으로 실행
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --py-files s3://spark-libs/spark_error_listener.zip \
  my_spark_app.py
```

이렇게 세 단계만 거치면 끝이다. Python으로 SparkListener를 구현하고, SparkContext에 등록하고, ZIP 파일로 배포하면 Spark 작업이 실행되는 동안 자동으로 에러를 감지해서 n8n으로 전송한다. 처음엔 복잡해 보였지만 막상 해보니 PySpark의 내부 구조를 이해하는 좋은 기회였다. 

### 4. Spark History MCP 서버

AWS의 Spark History Server MCP는 **항상 구동(Running) 중인 EMR 클러스터**에서만 통신이 가능하도록 구현되어 있었다. 우리 팀은 비용 절감을 위해 Transient EMR 클러스터를 사용하는데, 작업이 끝나면 클러스터가 자동으로 종료된다. AI가 분석하는 동안 EMR이 종료될 수 있다는 게 문제였다.

다행히 Amazon EMR은 **Persistent Spark History Server**를 제공한다. EMR 클러스터가 종료되어도 Spark 이벤트 로그를 S3에 저장해두고, 별도의 영구적인 History Server를 통해 조회할 수 있는 기능이다. 하지만 오픈소스 MCP 서버는 이 Persistent History Server를 지원하지 않았다.

결국 [오픈소스 코드](https://github.com/kubeflow/mcp-apache-spark-history-server)를 직접 수정하여 Persistent History Server와 통신할 수 있도록 개선할 수밖에 없었다. 이제 EMR 클러스터가 종료된 후에도 AI는 S3에 저장된 이벤트 로그를 통해 모든 메트릭과 로그를 분석할 수 있게 되었다. 

## 실제 사용 예시

워크플로우가 Data Skew를 발생하고, Executor가 특정 큰 파티션을 처리하던 중 OOM이 발생했다.

### 실패한 Spark 작업의 설정값

```bash
--conf spark.dynamicAllocation.enabled=true
--conf spark.sql.adaptive.enabled=true
--conf spark.sql.adaptive.coalescePartitions.enabled=true
--conf spark.sql.adaptive.coalescePartitions.minPartitionSize=128mb
--conf spark.sql.adaptive.advisoryPartitionSizeInBytes=256mb
--conf spark.executor.cores=2
--conf spark.executor.memory=6g
```


### AI Agent가 제시해준 단기 전략

- `spark.executor.memory`: 6g → 12g
- `spark.executor.memoryOverhead`: 2g
- `spark.sql.shuffle.partitions`: 1000 → 1500

### AI Agent가 제시해준 Spark 작업의 설정값 적용하여 재시도

```bash
--conf spark.dynamicAllocation.enabled=true
--conf spark.sql.adaptive.enabled=true
--conf spark.sql.adaptive.coalescePartitions.enabled=true
--conf spark.sql.adaptive.coalescePartitions.minPartitionSize=128mb
--conf spark.sql.adaptive.advisoryPartitionSizeInBytes=256mb
--conf spark.sql.shuffle.partitions=1500  # shuffle 파티션 확장
--conf spark.executor.cores=2             # executor 코어수 유지
--conf spark.executor.memory=12g          # executor 메모리 확장
--conf spark.executor.memoryOverhead=2g   # 추가
```

**코드 수정, EMR 클러스터 스펙 조정 없이** AI Agent가 알려준 설정대로 적용 → 파티션 크기가 작아짐(69MB → 47MB)으로써 Spill이 발생하지 않고, 모두 정상 처리되었다.

## 개선 효과

내 트러블슈팅 프로세스를 다시 살펴보자. 프로젝트를 시작하기 전, 이렇게 6단계로 트러블슈팅을 진행했었다.

**이전: 6단계 프로세스 (평균 소요 시간: 60분)**

```
[문제 발생] MSP의 알림으로 장애 인지
↓
[1단계] (VPN 접속 후) Airflow Task 확인 (5분)
↓
[2단계] Spark History UI 혹은 Yarn Resource Manager에서 실패한 Job, Stage 찾기 (10분)
↓
[3단계] Driver, Executor 로그, Spark 설정 검색 (15분)
↓
[4단계] Stage 메트릭 분석 (Shuffle, GC, Memory) (15분)
↓
[5단계] 원인 추정 및 해결책 도출 (10분)
↓
[6단계] 설정 변경 후 재실행 (5분)
```

이 과정이 이제는 다음과 같이 개선되었다.

**개선 후: 3단계 프로세스 (평균 소요 시간: 10분, 83% 단축)**

```
[문제 발생] MSP의 알림으로 장애 인지
↓
[1단계] AI Analysis가 자동 분석/레포트 (AI가 자동으로 수행, 약 2분)
  - 기존 1~5단계를 사람 개입 없이 자동 처리
  - Spark History Server 메트릭 자동 조회
  - 에러 패턴 분석 및 원인 추정
  - 해결책 도출 및 레포트 생성
↓
[2단계] Slack 채널에서 AI 분석 보고서 확인 (3분)
  • 어떤 에러인지
  • 왜 발생했는지
  • 어떻게 해결하는지
↓
[3단계] AI 제안대로 설정 변경 후 재실행 (5분) ✅
```

### 정량적 개선 지표

물론 이런 개선 효과를 정량적으로 측정하기는 쉽지 않다. 에러의 복잡도에 따라 분석 시간이 천차만별이고, 사람마다 Spark에 대한 이해도도 다르기 때문이다. 하지만 나는 엔지니어고, 엔지니어라면 체감한 개선 효과를 수치로 표현해봐야 하지 않겠는가.

- **트러블슈팅 단계**: 6단계 → 3단계 (50% 감소)
- **사람의 수동 분석 시간**: 50분 → 8분 (84% 감소)
- **VPN 접속 및 여러 UI 탐색 필요**: 필수 → 불필요

이런 성과들을 보면서 나는 확신하게 되었다. **"AI는 반복적이고 패턴이 있는 작업을 정말 잘한다"**는 것을. Spark 디버깅이 바로 그런 작업이었다. 이제 AI에게 디버깅을 맡기고, 나는 더 창의적이고 가치 있는 일에 집중할 수 있게 되었다.

## 결론

솔직히 처음엔 반신반의했다. "AI가 Spark 에러를 제대로 분석할 수 있을까?" 하는 의구심이 있었다. 하지만 막상 사용해보니 **AI가 제시해준 솔루션의 정확도에 놀랐다**.

실제 사용 예시에는 기술하지 않았지만, 어떤 케이스에서는 OOM이 발생한 근본적인 이유가 Spark UDF였다. 메모리 설정을 아무리 늘려도 해결되지 않는 문제였는데, AI는 Stage 메트릭과 Task 분포를 분석한 후 "이 UDF가 문제의 원인"이라고 정확히 짚어냈다. 심지어 "Spark-Native 함수로 변경하거나 Pandas UDF를 사용하라"는 구체적인 해결책까지 제시했다. 결국 Spark-Native 코드로 변경하니 문제가 완전히 해결되었다.

이런 경험을 하고 나니 AI가 단순히 로그를 요약하는 수준이 아니라, **실제로 Spark 내부 동작을 이해하고 있다**는 확신이 들었다. 

그리고 이 프로젝트는 단순한 자동화를 넘어서 **Spark의 Observability를 확보**한 것이라고 할 수 있다. 사실 그동안 Spark 작업이 실패하면 사후에 로그를 뒤지는 방식으로만 대응해왔다. 실시간 모니터링도, 구조화된 메트릭 수집도, 체계적인 분석 프로세스도 없었다. 하지만 이제는 SparkListener를 통해 모든 이벤트를 실시간으로 캡처하고, Spark History Server의 메트릭을 구조화된 방식으로 조회하며, AI를 통해 자동으로 분석하는 시스템을 갖추게 되었다. 비로소 "우리 Spark 작업이 어떻게 돌아가고 있는지"를 제대로 볼 수 있게 된 것이다.
