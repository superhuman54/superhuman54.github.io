---
layout: post
title: "Airflow Provider 패키지 버전 충돌 문제"
date: 2023-08-15
categories: [Airflow, Python]
tags: [airflow, provider, dependency, kubernetes, troubleshooting, pip]
---

Airflow 버전을 업그레이드하는 과정에서 Python Airflow DAG 모듈이 패키지를 import하는 과정에서 다음과 같은 에러가 발생했다.

<!-- more -->

![Error on Airflow UI](https://github.com/user-attachments/assets/f36ad930-4ac8-4183-8fcd-f050a55a67e1)
*Kubernetes Executor는 Airflow 2.7.0 이상에서만 지원된다는 에러 메시지*

에러 내용을 해석하면 다음과 같다:
- 현재 Airflow 2.4.3 버전을 사용 중
- CNCF Provider의 Kubernetes Executor를 사용하려고 시도
- 이 조합은 Airflow 2.7.0 이상에서만 지원됨

Airflow가 필요한 패키지는 아래와 같이 기술되어 있다:
```text
apache-airflowcncf.kubernetes
kubernetes
requests
typing_extensions
```

버전 명시의 누락이 원인이지만, 그래도 트러블슈팅을 해본다.

## 분석

### Provider 패키지 의존성 분석

먼저, 의심이 가는 `apache-airflow[cncf.kubernetes]`의 버전을 "명시하지 않고" 설치를 해본 뒤, 내부적으로 그 패키지가 의존하는 의존성(이를 Transitive Dependency라고 부른다)의 버전을 확인해본다.

```shell
pip install apache-airflowcncf.kubernetes
```

위의 명령을 실행하면 KubernetesExecutor와 함께 `apache-airflow-providers-cncf-kubernetes`의 의존성도(transitive dependency) 설치된다.

`apache-airflow-providers-cncf-kubernetes`의 버전을 명시하지 않으면 설치된 Airflow의 버전과 호환 가능한 가장 최신 버전의 provider로 설치된다. 이번 provider의 최신 버전은 특정 Airflow 버전(2.7.0+)에서 호환되도록 구현되어 있고, 우리가 운영하는 Airflow 버전(2.4.3)과 충돌(conflict)로 인해 DAG 파이썬 모듈의 import 과정에서 에러가 발생했다.

### 코드 레벨 분석

```python
try:
    from airflow.cli.cli_config import (
        ARG_DAG_ID,
        ARG_EXECUTION_DATE,
        ARG_OUTPUT_PATH,
        ARG_SUBDIR,
        ARG_VERBOSE,
        ActionCommand,
        Arg,
        GroupCommand,
        lazy_load_command,
        positive_int,
    )
except ImportError:
    try:
        from airflow import __version__ as airflow_version
    except ImportError:
        from airflow.version import version as airflow_version

    import packaging.version

    from airflow.exceptions import AirflowOptionalProviderFeatureException

    base_version = packaging.version.parse(airflow_version).base_version

    if packaging.version.parse(base_version) < packaging.version.parse("2.7.0"):
        raise AirflowOptionalProviderFeatureException(
            "Kubernetes Executor from CNCF Provider should only be used with Airflow 2.7.0+.\n"
            f"This is Airflow {airflow_version} and Kubernetes and CeleryKubernetesExecutor are "
            f"available in the 'airflow.executors' package. You should not use "
            f"the provider's executors in this version of Airflow."
        )
```
*airflow.providers.cncf.kubernetes.executors.kubernetes_executor.py에서 Airflow 2.7.0+에만 가능하도록 방어된 코드*

7.4.2의 `airflow.providers.cncf.kubernetes.executors.kubernetes_executor.py`에서 Airflow 2.7.0+에만 가능하도록 방어된 코드를 확인할 수 있다:


### 버전 변화 추적

업그레이드 전후에 어떤 버전의 provider가 설치되어 있는지 비교해본다. 모두 동일한 `requirements.txt`를 사용했기 때문에, 환경은 동일하다.

**업그레이드 전 (7.3.0 설치):**

```text
Collecting apache-airflow-providers-cncf-kubernetes
Downloading apache_airflow_providers_cncf_kubernetes-7.3.0-py3-none-any.whl (60 kB) <= 7.3.0를 다운로드 받는다.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 60.1/60.1 kB 14.0 MB/s eta 0:00:00
...
Installing collected packages: ... kubernetes-asyncio, kubernetes, apache-airflow-providers-cncf-kubernetes
Successfully installed apache-airflow-providers-cncf-kubernetes-7.3.0 asgiref-3.7.2 ...
```
*Airflow 2.4.3 업그레이드 전 apache-airflow-providers-cncf-kubernetes 7.3.0 설치 로그*

**업그레이드 후 (7.4.2 설치):**

```text
Collecting apache-airflow-providers-cncf-kubernetes
Downloading apache_airflow_providers_cncf_kubernetes-7.4.2-py3-none-any.whl (108 kB) <= 7.4.2를 다운로드 받는다.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 108.8/108.8 kB 26.7 MB/s eta 0:00:00
...
Installing collected packages: ... kubernetes-asyncio, kubernetes, apache-airflow-providers-cncf-kubernetes
Successfully installed apache-airflow-providers-cncf-kubernetes-7.4.2 asgiref-3.7.2 ...

```
*업그레이드 후 apache-airflow-providers-cncf-kubernetes 7.4.2 설치 로그*


동일한 `requirements.txt`이지만 다른 날짜에 다른 버전이 설치되었고, `apache-airflow-providers-cncf-kubernetes==7.4.2`는 우리가 업그레이드하기 전에 릴리즈되어 7.4.2가 설치된 것을 PyPI에서 확인할 수 있었다.

<img width="1136" height="187" alt="image" src="https://github.com/user-attachments/assets/c4094e84-c128-453d-b402-36bbda4f33de" />

*2023-08-13에 7.4.2 버전이 출시되었다*

### Import 경로 분석

DAG 모듈을 인터프리터가 분석할 때, 다음의 `apache-airflow-providers-cncf-kubernetes` 버전에 따라 import하는 모듈이 다르다.

**apache-airflow-providers-cncf-kubernetes 7.3.0:**
1. DAG 파싱
2. `airflow.providers.cncf.kubernetes.operators.kubernetes_pod.KubernetesOperator`
   - `kubernetes_pod` 모듈은 deprecated되어 다음의 `pod` 모듈 사용
3. `airflow.providers.cncf.kubernetes.operators.pod.KubernetesOperator`
4. `airflow.kubernetes.pod_generator`
   - **Airflow Core 라이브러리의 pod_generator 사용**

**apache-airflow-providers-cncf-kubernetes 7.4.2:**
1. DAG 파싱
2. `airflow.providers.cncf.kubernetes.operators.kubernetes_pod.KubernetesOperator`
   - `kubernetes_pod` 모듈은 deprecated되어 다음의 `pod` 모듈 사용
3. `airflow.providers.cncf.kubernetes.operators.pod.KubernetesOperator`
4. `airflow.providers.cncf.kubernetes.pod_generator`
   - **Airflow provider 라이브러리의 pod_generator 사용**
5. `airflow.providers.cncf.kubernetes.executors.kubernetes_executor`
6. `airflow.cli.cli_config` ImportError 발생

Airflow Core 라이브러리는 Airflow 2.6.0부터 `cli_config` 모듈을 소개했다.

`airflow.providers.cncf.kubernetes.operators.pod` 모듈은 provider 버전에 따라 `pod_generator` 모듈을 다른 패키지에서 import한다(순서상 4번).

### 의존성 스펙 분석

그렇다면 아래의 명제에서 7.4.2는 코드상 분명히 Airflow 2.7.0+라고 방어 코드가 있지만, 우리의 Airflow 2.4.3에 왜 설치가 된 것일까?

> apache-airflow-providers-cncf-kubernetes의 버전을 명시하지 않으면 설치된 Airflow의 버전과 호환 가능한 가장 최신 버전의 provider로 설치된다.

답은 provider의 스펙에서 찾을 수 있었다.

<img width="242" height="602" alt="image" src="https://github.com/user-attachments/assets/4de7574e-9ee9-4ed5-9f4a-f131d7b1b3a6" />

*최신 7.4.2의 의존성에서 Airflow Core 의존성이 2.4.0+으로 설정된 것을 보여주는 스크린샷*

최신 7.4.2의 의존성에서 Airflow Core 의존성이 `>=2.4.0`으로 되어 있기 때문에 우리의 Airflow 버전(2.4.3)에 문제없이 설치가 되었다. 아무래도 provider의 최소 버전부터 맞추려다 보니 Airflow Core 의존성의 버전도 낮게 설정되어 있지 않았을까 추측한다.

## 해결

<img width="1910" height="164" alt="image" src="https://github.com/user-attachments/assets/6f101ed7-097f-4e86-9d6b-0bfd916b73a6" />
*Airflow 공식 문서에서 제공하는 constraints 파일 사용 방법*

pip constraints를 통해 transitive dependencies를 특정 버전으로 강제하도록 한다. 이미 Airflow에서는 Airflow Core 라이브러리 버전에 매핑된 constraints를 제공한다.

```shell
pip install apache-airflowcncf.kubernetes –constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.4.3/constraints-3.8.txt"
```

또는 requirements.txt에서 직접 버전을 명시하는 방법도 있다:

```text
apache-airflow-providers-cncf-kubernetes==7.3.0
```

## 결론과 교훈

### 학습한 교훈

1. **의존성 버전 명시의 중요성**: Provider 패키지의 버전을 명시하지 않으면 예상치 못한 호환성 문제가 발생할 수 있다.

2. **Transitive Dependency 관리**: 직접 설치하지 않는 의존성도 버전 호환성 문제를 일으킬 수 있으므로 constraints 파일을 활용해야 한다.

3. **Provider 패키지의 독립적 릴리즈**: Airflow Provider 패키지는 Core와 독립적으로 릴리즈되므로, 각각의 호환성을 별도로 확인해야 한다.

### 예방 방법

1. **Constraints 파일 사용**: 항상 Airflow 공식 constraints 파일을 사용하여 검증된 의존성 조합을 활용한다.

2. **버전 핀닝**: 중요한 패키지들은 requirements.txt에서 명시적으로 버전을 고정한다.

3. **테스트 환경에서 검증**: 업그레이드 전에 반드시 테스트 환경에서 의존성 호환성을 검증한다.

이번 경험을 통해 Python 패키지 의존성 관리의 복잡성과 Airflow Provider 시스템의 구조를 깊이 이해할 수 있었다.







