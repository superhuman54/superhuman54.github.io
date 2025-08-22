---
layout: post
title: "Amazon EMR 6.12와 AWS Java SDK 충돌 문제"
date: 2023-11-24
categories: [AWS, Spark, EMR]
tags: [aws, emr, spark, java-sdk, dependency, troubleshooting]
---

운영 중인 Spark 애플리케이션의 성능 향상과 보안 패치를 위해 Amazon EMR 클러스터를 6.2.0에서 6.12.0으로 업그레이드했다. 하지만 업그레이드 직후 예상치 못한 문제가 발생했다. 기존에 정상적으로 동작하던 Spark 애플리케이션이 Amazon Glue 카탈로그에서 메타데이터를 읽어오는 과정에서 `NoSuchMethodError`를 발생시키며 실패하기 시작한 것이다.

<!-- more -->

![Stacktrace](https://github.com/user-attachments/assets/95fffa4d-8a09-44c8-a728-f5a930af4dd4)


## 문제 발생

문제가 발생한 Spark 애플리케이션은 다음과 같이 제출되고 있었다:
```shell
spark-submit 
-–master yarn 
$MASTER_URL 
–-name any_spark_app 
–-conf spark.sql.session.timeZone=“Asia/Seoul” 
–-conf spark.driver.memory=$DRIVER_MEM 
–-conf spark.executor.memory=$EXECUTOR_MEM 
–-conf spark.executor.cores=$EXCUTOR_CORES 
–-conf spark.dynamicAllocation.enabled=true 
–-conf spark.yarn.maxAppAttempts=1 
–-conf spark.sql.broadcastTimeout=2000 
–-packages com.amazonaws:aws-java-sdk-core:1.11.975 \ # 문제의 SDK 
–-conf spark.pyspark.python=${PYSPARK_PYTHON} 
–-archives env.zip#env${RES_OPTIONS} 
${SRC_OPTIONS} 
${PYTHON_ENTRY_PATH} ${PYTHON_ARGS}
```

업그레이드 후 Spark Analyzer가 parsed logical plan을 resolve하는 과정에서 Amazon Glue 카탈로그의 데이터베이스 정보를 JSON으로부터 역직렬화하려 할 때 메소드를 찾을 수 없다는 에러가 발생했다. 이는 Spark SQL이 Hive 메타스토어 대신 Glue 카탈로그를 사용하는 테이블에 접근할 때마다 반복적으로 발생하는 치명적인 문제였다.

## 원인 분석

### 의존성 충돌 진단

`NoSuchMethodError`는 대부분 의존성 버전 충돌로 인해 발생한다. 문제를 정확히 파악하기 위해 애플리케이션이 의존하는 `com.amazonaws:aws-java-sdk-core:1.11.975` 버전의 `JsonUnmarshallerContext.getCurrentToken()` 메소드 시그니처를 확인해보았다.

```java
package com.amazonaws.transform;

import java.io.IOException;
import java.util.Collections;
import java.util.Map;

import com.amazonaws.http.HttpResponse;
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.core.JsonToken; 

public abstract class JsonUnmarshallerContext {

    public enum UnmarshallerType {
        JSON_VALUE
    }
    
    // 중략

    public JsonToken getCurrentToken() {}
        return null;
    }
```


여기서 중요한 발견을 했다. 위 코드에서 `JsonToken` 클래스의 패키지는 `com.fasterxml.jackson.core`인데, 에러 스택트레이스에서는 `com.amazonaws.thirdparty.jackson.core` 패키지를 참조하고 있었다. 동일한 Jackson 라이브러리지만 패키지명에 `com.amazonaws.thirdparty`가 추가되어 있었던 것이다.

### Spark SQL 호출 체인 분석

Amazon Glue Catalog를 사용하는 Spark SQL이 쿼리를 수행할 때의 호출 순서는 다음과 같다:

| 순서 | 클래스 | ArtifactId |
|------|--------|------------|
| 1 | `org.apache.spark.sql.hive.HiveExternalCatalog` | spark-hive |
| 2 | `com.amazonaws.glue.catalog.metastore.AWSCatalogMetastoreClient` | aws-glue-datacatalog-hive-client |
| 3 | `com.amazonaws.services.glue.AWSGlueClient` | aws-java-sdk-glue |
| 4 | `com.amazonaws.transform.JsonUnmarshallerContext` | aws-java-sdk-core |

이 호출 체인에서 문제가 발생한 지점은 마지막 단계였다. `AWSGlueClient`는 EMR 버전 업그레이드와 함께 새로운 버전으로 교체되었고, 이 클라이언트는 자신과 호환되는 `aws-java-sdk-core`를 기대하고 있었다. 하지만 우리 애플리케이션은 명시적으로 낮은 버전의 SDK를 의존하고 있었기 때문에 충돌이 발생한 것이다.

### EMR 버전별 SDK 패키징 전략의 변화

문제의 근본 원인을 이해하기 위해 EMR 6.2와 6.12 버전 사이의 AWS SDK 패키징 전략 변화를 분석했다.

**EMR 6.2의 AWS SDK 구조:**
- 서비스별로 개별 JAR 파일 제공 (예: `aws-java-sdk-glue-1.11.880.jar`)
- 써드파티 라이브러리의 패키지명 변경 없음
- `aws-java-sdk-core-1.11.880.jar`에 Jackson 라이브러리 원본 패키지명 유지

**EMR 6.12의 AWS SDK 구조:**
- 모든 서비스를 하나로 통합한 번들 JAR 사용 (`aws-java-sdk-bundle-1.12.490.jar`)
- 써드파티 라이브러리 패키지명 재배치(relocation) 적용
- Jackson 라이브러리가 `com.amazonaws.thirdparty.jackson.core`로 이동

![aws java sdk](https://github.com/user-attachments/assets/1f5e1d68-addb-4c5b-9748-f5f65506486a)

그리고 이 버전의 `JsonUnmarshallerContext`의 메소드를 확인해보았다.
![JsonUnmarshallerContext](https://github.com/user-attachments/assets/cbd0aae6-8fe5-45c8-8a16-a92b5760cc5f)
*패키지명 재배치가 적용된 문제의 메소드*

이러한 패키징 전략의 변화는 AWS SDK 1.11.84 버전부터 도입된 것으로, 써드파티 라이브러리 간의 버전 충돌을 방지하기 위한 조치였다. 하지만 기존 방식으로 개발된 애플리케이션에서는 예상치 못한 문제를 야기할 수 있었다. 

### 충돌 발생 메커니즘

문제 상황을 정리하면 다음과 같다:

1. **EMR 6.12 환경**: `aws-java-sdk-bundle-1.12.490.jar`가 설치되어 있음 (패키지 재배치 적용)
2. **애플리케이션**: `--packages` 옵션으로 `aws-java-sdk-core-1.11.975.jar` 추가 (패키지 재배치 없음)
3. **런타임 충돌**: EMR의 `GetDatabaseResultJsonUnmarshaller`는 재배치된 Jackson 패키지를 기대하지만, 애플리케이션이 추가한 SDK는 원본 패키지명을 사용

클래스로더의 우선순위에 따라 애플리케이션이 런타임에 추가한 의존성이 더 높은 우선순위를 가지게 되면서, EMR의 Glue 클라이언트가 기대하지 않은 클래스 구조를 만나게 된 것이다.

## 해결 방안

해결책은 의외로 간단했다. **불필요한 의존성 제거**였다.

다음 문제의 패키지를 제거하여, EMR에 이미 설치된 SDK를 사용하도록 한다.
```text
–-packages com.amazonaws:aws-java-sdk-core:1.11.975 // 제거
```

EMR 6.12에는 이미 최적화된 AWS SDK가 번들 형태로 포함되어 있기 때문에, 별도로 SDK를 추가할 필요가 없었다. 오히려 명시적으로 추가한 낮은 버전의 SDK가 EMR에 설치된 최신 버전과 충돌을 일으키는 원인이 되었던 것이다.

## 교훈과 인사이트

### AWS SDK 패키징 전략의 진화

이번 문제를 통해 AWS SDK의 패키징 전략이 어떻게 진화해왔는지 깊이 이해할 수 있었다. Maven Shade Plugin을 사용한 패키지 재배치(relocation)는 의존성 충돌을 방지하는 강력한 도구이지만, 기존 애플리케이션과의 호환성 문제를 야기할 수 있다는 양면성을 가지고 있다.

### 의존성 관리의 중요성

"필요할 것 같다"는 추측으로 의존성을 추가하는 것이 얼마나 위험한지 다시 한번 깨닫게 되었다. 특히 매니지드 서비스 환경에서는 플랫폼에서 제공하는 라이브러리들이 이미 최적화되어 있는 경우가 많기 때문에, 불필요한 의존성 추가는 오히려 독이 될 수 있다.

### EMR 업그레이드 시 고려사항

EMR 버전 업그레이드는 단순히 Spark나 Hadoop 버전의 변경뿐만 아니라, 하위 수준의 의존성까지 광범위하게 영향을 미칠 수 있다. 업그레이드 전에는 반드시 다음 사항들을 점검해야 한다:

1. **기본 제공 라이브러리 버전 확인**: 새로운 EMR 버전에서 제공하는 기본 라이브러리들의 버전과 패키징 방식
2. **애플리케이션 의존성 검토**: 명시적으로 추가한 의존성들이 정말 필요한지, 그리고 새 버전과 충돌하지 않는지
3. **충분한 테스트**: 다양한 워크로드에서의 호환성 검증

## 마무리
이 문제는 표면적으로는 복잡한 의존성 충돌 문제로 보였지만, 실제로는 불필요한 의존성 추가라는 단순한 실수에서 비롯되었다. 하지만 이 과정에서 AWS SDK의 내부 구조와 패키징 전략, 그리고 EMR의 의존성 관리 방식에 대해 깊이 있게 이해할 수 있는 귀중한 경험이었다.
특히 흥미로웠던 점은 AWS가 써드파티 라이브러리 충돌을 방지하기 위해 패키지 재배치(relocation) 기법을 적용했다는 것이다. 이는 Maven의 표준적인 접근 방식으로, Jackson과 같은 인기 있는 라이브러리들이 서로 다른 버전으로 충돌하는 것을 방지하기 위해 `com.amazonaws.thirdparty.jackson.core`와 같이 패키지명을 변경하는 것이다.
Maven Shade Plugin을 통한 패키지 재배치는 의존성 지옥(dependency hell)을 해결하는 강력한 도구이지만, 기존 애플리케이션과의 호환성 문제를 야기할 수 있다는 양면성을 가지고 있다. AWS SDK Bundle JAR의 도입도 이런 맥락에서 이해할 수 있으며, 장기적으로는 의존성 관리를 단순화하고 안정성을 높이는 방향으로 발전하고 있다.
앞으로는 매니지드 서비스를 사용할 때 플랫폼에서 제공하는 최적화된 구성을 최대한 활용하고, 정말 필요한 경우에만 최소한의 의존성을 추가하는 원칙을 지켜야겠다. “Less is More”라는 격언이 의존성 관리에서도 여전히 유효하다는 것을 다시 한번 확인한 사례였다.

## 참고
[Guide to relocation – Maven Apache](https://maven.apache.org/guides/mini/guide-relocation.html)




