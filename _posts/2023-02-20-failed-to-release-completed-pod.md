---
layout: post
title: "Amazon MWAA에서 Kubernetes Pod 회수 실패 문제"
date: 2023-02-10
categories: [Airflow, Kubernetes, AWS]
tags: [mwaa, airflow, kubernetes, pod, troubleshooting, celery, eks]
author: K3N
description: "Amazon MWAA에서 KubernetesOperator Pod가 작업 완료 후 회수되지 않는 문제 해결 과정. CeleryWorker 축소와 xcom 컨테이너 생명주기 관리 문제를 분석합니다."
keywords: "mwaa, airflow, kubernetes, pod, celery, eks, troubleshooting, xcom, container lifecycle"
---

운영 환경에서 Amazon MWAA(Managed Workflows for Apache Airflow)를 사용하던 중 심각한 문제가 발생했다. `KubernetesOperator`를 통해 생성된 Pod가 작업 완료 후에도 회수되지 않고 계속 실행 중인 상태로 남아있어 클러스터 리소스를 불필요하게 점유하는 문제였다.

<!-- more -->

![Not Ready Pod](https://github.com/user-attachments/assets/799617d7-2f20-43c7-ad25-a65760b5a22e)
*kubectl describe로 확인한 Pod 상태 - NotReady 상태로 표시됨*

명령어를 통해 Pod 상태를 확인해보니 `kubectl`에서는 NotReady 상태로 표시되고 있었지만, 실제로는 Pod 내부의 컨테이너가 여전히 구동 중이었다. 더욱 이상한 점은 Amazon EKS 콘솔에서 확인한 Pod 상태가 kubectl 결과와 다르게 나타났다는 것이었다.

![EKS Console](https://github.com/user-attachments/assets/c69eacf8-62e1-44cc-b0c8-c35e35a2160c)
*EKS 콘솔에서 확인한 같은 Pod의 상태 - Running으로 표시*

동일한 Pod에 대해 `kubectl`과 Amazon EKS 콘솔에서 서로 다른 상태로 확인되었지만, 결국 Pod가 회수되지 않고 계속 구동 중이라는 사실은 변하지 않았다. 이는 Pod 내부 컨테이너가 여전히 실행 중이기 때문이었다.

## 원인 분석

### Airflow Worker 로그 분석

문제의 근본 원인을 파악하기 위해 해당 Pod가 남긴 Airflow Worker 로그를 상세히 분석했다.

![Stuck Logs](https://github.com/user-attachments/assets/b6372c4d-1c83-4dcd-8a82-4e5740addbca)
*Airflow Worker 로그가 갑자기 끊어진 상황*

로그를 확인하던 중 갑작스럽게 끊어진 부분을 발견했다. 로그를 출력하는 프로세스가 예기치 않게 종료된 것으로 추정되었다. 이때 EKS 콘솔에서 매우 중요한 단서를 발견했다.

![xcom container running](https://github.com/user-attachments/assets/e12b1559-92b2-46ac-8bc5-3ba0c1f2e4ef)
*앱 컨테이너는 종료되었지만 xcom 사이드카 컨테이너가 계속 실행 중인 상태*

애플리케이션의 base 컨테이너는 정상적으로 종료되었지만, 사이드카로 구동 중인 xcom 컨테이너가 계속 실행되고 있었다. 이는 xcom 컨테이너가 busy wait 상태로 대기하고 있으며, 어떤 프로세스도 이를 적절히 종료시켜주지 않고 있다는 것을 의미했다.

### Airflow Pod 생명주기 분석

문제를 정확히 이해하기 위해 Airflow Worker가 생성하는 Pod의 정상적인 생명주기를 분석했다.

![Pod Lifecycle](https://github.com/user-attachments/assets/0448d0a8-c3d7-4615-ad34-bcbb012aef02)

*정상적인 Airflow Pod 생명주기 흐름도*

정상적인 Pod 생명주기는 다음과 같다:

1. **작업 할당**: CeleryWorker가 CeleryExecutor로부터 작업을 받는다
2. **Base 컨테이너 생성**: `pod_manager`가 base 컨테이너를 생성한다(1번 화살표).
3. **Xcom 컨테이너 생성**: base 컨테이너의 작업이 xcom 커뮤니케이션이 필요하다면, `pod_manager`가 xcom 컨테이너를 생성한다(2번 화살표).
4. **Pod 실행**: Pod가 RUNNING 상태로 변경된다
5. **애플리케이션 실행**: base 컨테이너는 애플리케이션이므로 이 컨테이너의 생명주기는 애플리케이션의 책임이다
6. **Xcom 컨테이너 종료**: base 컨테이너가 종료되고 xcom 커뮤니케이션이 완료되면, `pod_manager`는 xcom 컨테이너를 종료한다 (xcom 컨테이너의 생명주기는 `pod_manager`의 책임)
7. **Pod 회수**: Pod이 회수되면서 COMPLETED 상태로 변경된다

문제는 어떤 이유로 CeleryWorker가 회수되면서 6번 이후 과정이 생략된다는 점이었다. 그렇다면 왜 CeleryWorker가 회수되었는지, 그 시점의 Airflow 지표와 CeleryWorker 로그를 면밀히 조사해야 했다.

### CeleryWorker 로그 상세 분석

![Celery Worker Logs](https://github.com/user-attachments/assets/76add46b-475d-433c-8dd9-82ddac96f5ba)
*CeleryWorker의 비정상적인 종료 과정을 보여주는 로그*

문제가 발생한 CeleryWorker의 로그를 시간순으로 분석한 결과:

1. **(Celery 로그)** `Task airflow.executors.celery_executor.execute_command received`
   - MainProcess에게 작업이 할당된다 (task_id=c0994a20-db82-4f40-b8e9-287faa712d71)

2. **(Celery 로그)** `Scaling up 1 processes`
   - 할당된 작업을 수행하기 위해 MainProcess는 프로세스를 스케일링한다

3. **(Airflow 로그)** `Execution command in Celery`
   - Celery 워커(ForkPoolWorker-26)가 브로커에서 메시지를 받고 명령을 실행한다

4. **(Celery 로그)** `worker: Warm shutdown (MainProcess)`
   - **Warm Shutdown 시그널을 받고 종료된다**

정상적인 상황이라면 3번 로그 다음에 브로커로부터 작업을 수신(ack)했다는 다음과 같은 로그를 남기는 것이 일반적이다:
```text
YYYY-MM-DD HH:mm:ss.sss: INFO/ForkPoolWorker-XX Task airflow.executors.celery_executor.execute_commandabcdef-ghij-1234-5678-abcdefe succeeded in 4.067282755999258s: None
```

하지만 문제의 워커는 이 로그를 출력하지 않고 shutdown되었다. 이는 작업이 완료되기 전에 워커가 강제로 종료되었음을 의미했다.

### MWAA 지표 분석

![metrics1](https://github.com/user-attachments/assets/002292bf-27cc-4bfb-bf98-feb33a8b5221)
![metrics2](https://github.com/user-attachments/assets/1d606bef-3787-48e8-8e22-f3020b2ef191)
*2022-02-17 04:30:00 주변 시간대의 MWAA 지표*

문제 발생 시점인 2022-02-17 04:30:00 주변의 MWAA 지표를 확인한 결과, 매우 흥미로운 패턴을 발견할 수 있었다.

![metrics3](https://github.com/user-attachments/assets/171469c5-cb5d-441d-8cbc-b880f886f735)
*04:25부터 워커 수가 4개에서 2개로 감소하는 과정*

04:25에 진행 중인 작업이 줄어들면서(1.25개 → 0.183개) 워커의 개수도 줄어들다가, 결국 워커 수가 4개에서 2개로 감소했다. 보라색 그래프가 끊어진 것은 해당 프로세스가 종료되면서 지표 수집도 실패한 것으로 분석되었다.

이 시간대에는 작업량이 줄어들기 시작했고, Amazon MWAA에서는 불필요한 워커를 자동으로 축소(scale-in)하는 과정을 진행했다. **문제는 축소 대상이 된 워커가 바로 작업을 한창 진행 중이었고, 그 워커가 회수되면서 Kubernetes Pod와의 논리적인 연결이 끊어진 것이었다.**

### 핵심 문제점

중요한 사실은 `KubernetesOperator`로 생성된 Pod는 Airflow 워커와 특별한 연결 장치가 없다는 점이다. Airflow 워커는 단순히 Kubernetes API를 통해 생성한 Pod의 상태를 확인하고, Airflow 워커가 사라진다고 하더라도 서로는 언제나 독립적인 서버-클라이언트 관계이기 때문에 Pod는 자동으로 사라지지 않는다.

**결국 Pod의 생명주기(정확히는 xcom 컨테이너)를 담당하는 워커가 사라지면서, xcom 컨테이너의 생명주기를 관리해주지 못하여 계속 RUNNING 상태로 남게 된 것이다.**

## 해결 방안

### 임시 해결책

작업 중인 워커가 축소 대상이 되는 것이 의아하여, 우선 최소 워커 개수를 늘려서 이런 상황을 방지하는 임시 조치를 취했다.

![MWAA Configuration](https://github.com/user-attachments/assets/a552f344-0db3-49bd-ace1-cfacaec99f9f)
*Amazon MWAA 설정에서 min-workers 값을 증가시켜 스케일다운 방지*

### 근본적 해결책: Amazon MWAA 공식 권장사항

Amazon MWAA 공식 문서에서 이 문제에 대한 명확한 설명과 해결책을 발견했다.

Amazon MWAA의 자동 스케일링 메커니즘에 대한 공식 설명:

> Amazon MWAA uses Apache Airflow metrics to determine when additional Celery Executor workers are needed, and as required increases the number of Fargate workers up to the value specified by max-workers. When that number is zero, Amazon MWAA removes additional workers, downscaling back to the min-workers value.

즉, Amazon MWAA는 Apache Airflow 메트릭을 사용하여 추가 CeleryExecutor 워커가 필요한 시기를 결정하고, 필요에 따라 max-workers로 지정한 값까지 Fargate 워커 개수를 늘린다. 작업이 0개가 되면 Amazon MWAA는 추가 워커를 제거하고 min-workers 값으로 다시 축소한다.

**핵심적인 문제점에 대한 AWS의 설명:**

> When downscaling occurs, it is possible for new tasks to be scheduled. Furthermore, it's possible for workers that are set for deletion to pick up those tasks before the worker containers are removed. This period can last between two to five minutes, due to a combination of factors: the time it takes for the Apache Airflow metrics to be sent, the time to detect a steady state of zero tasks, and the time it takes the Fargate workers to be removed.

다운스케일링이 발생할 때, 새로운 작업이 할당될 가능성이 있으며, 삭제 대상이 된 워커가 워커 컨테이너가 제거되기 전에 해당 작업들을 받을 수도 있다. 이 기간은 Apache Airflow 메트릭 전송 시간, 무작업 상태 감지 시간, Fargate 워커 제거 시간의 조합으로 인해 2~5분간 지속될 수 있다.

### AWS 권장 해결방안

Amazon MWAA 문서에서는 다음 두 가지 해결방안을 권장한다:

1. **정적 워커 설정**: `min-workers`를 `max-workers`와 동일하게 설정하여 평균 워크로드를 충족할 수 있는 충분한 용량을 확보한다. 이는 24시간 중 대부분의 시간에 이런 패턴이 지속되는 경우에 적합하며, 이런 상황에서는 자동 스케일링의 가치가 제한적이다.

2. **지속적 작업 유지**: DateTimeSensor와 같은 최소 하나의 작업을 하나의 DAG에서 간헐적 활동 기간 동안 실행시켜 원치 않는 다운스케일링을 방지한다.

## 결론과 교훈

### 학습한 교훈

1. **Amazon MWAA의 자동 스케일링 특성 이해**: MWAA의 자동 스케일링은 메트릭 기반으로 동작하며, 다운스케일링 과정에서 타이밍 이슈가 발생할 수 있다는 점을 인지해야 한다.

2. **Pod 생명주기와 워커 생명주기의 독립성**: Kubernetes Pod와 Airflow Worker는 독립적인 생명주기를 가지므로, 워커가 종료되어도 Pod는 자동으로 정리되지 않는다.

3. **모니터링과 로그 분석의 중요성**: 복잡한 분산 시스템에서는 여러 계층의 로그와 메트릭을 종합적으로 분석해야 근본 원인을 파악할 수 있다.

### 운영 환경에서의 권장사항

1. **안정적인 워크로드 패턴**: 간헐적인 워크로드가 있는 환경에서는 min-workers를 적절히 설정하여 불필요한 스케일링을 방지한다.

2. **리소스 정리 모니터링**: Pod나 컨테이너가 제대로 정리되지 않는 상황을 감지할 수 있는 모니터링 시스템을 구축한다.

3. **테스트 환경에서의 검증**: 프로덕션 환경에 적용하기 전에 다양한 워크로드 패턴에서 충분한 테스트를 수행한다.

이번 경험을 통해 Amazon MWAA와 Kubernetes의 복잡한 상호작용을 깊이 이해하게 되었으며, 유사한 문제를 사전에 예방할 수 있는 운영 노하우를 축적할 수 있었다.


