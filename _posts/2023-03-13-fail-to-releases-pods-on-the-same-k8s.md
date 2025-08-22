---
layout: post
title: "공유 Kubernetes 클러스터에서 발생한 Airflow Pod 중복 생성 및 회수 실패 해결기"
date: 2023-03-13
categories: [Airflow, Kubernetes]
tags: [airflow, kubernetes, pod, duplicate, cleanup, troubleshooting]
---

소속팀은 2개의 독립적인 Airflow 클러스터가 1개의 K8s 클러스터를 "공유"한다(불필요한 과금 축소를 위해). 2개의 Airflow 클러스터는 다음의 환경에서 각각 운영된다.

<!-- more -->

- **alpha**: 상용 배포 전, 상용 데이터를 사용하는 테스트 환경  
- **production**: 상용 환경

개발자가 alpha에서 마지막 테스트를 진행하고 DAG을 비활성화를 잊어버리고, 그대로 production에 DAG을 동일한 시간에 등록한다. 문제는 각 환경의 DAG_RUN이 동일한 시간에 k8s에 Pod를 생성할때, Pod 회수가 실패하여 찌꺼기 Pod가 남는것이다.

## 문제 발생 과정

### Alpha 환경에서 Pod 생성

alpha에서 한국시각 18시 13분 경 다음의 Pod 생성 요청을 한다.

```text
2023-03-08T09:13:29.836+0000 {% raw %}{{kubernetes_pod.py:817}}{% endraw %} INFO - Building pod export-action-log-4devg7t1 with labels: {'dag_id': 'foo-bar-dag-id', 'task_id': 'foo-bar-task-id', 'run_id': 'scheduled__2023-03-08T0130000000-77fa57adf', 'kubernetes_pod_operator': 'True', 'try_number': '1'}
```
이때 Airflow는 Pod에 다음의 레이블을 추가한다:
```JSON
{
 "dag_id": "foo-bar-dag-id", 
 "task_id": "foo-bar-task-id", 
 "run_id": "scheduled__2023-03-08T0130000000-77fa57adf", 
 "kubernetes_pod_operator": "True", 
 "try_number": "1"
}
```

alpha에서 이 Pod는 RUNNING 상태로 전환된다. 그 뒤로 production 환경에서 18시 14분에 Pod 생성 요청을 한다.

### Production 환경에서 Pod 생성

```text
2023-03-08T09:14:46.342+0000 {% raw %}{{kubernetes_pod.py:817}}{% endraw %} INFO - Building pod export-action-log-rzwroznv with labels: {'dag_id': 'foo-bar-dag-id', 'task_id': 'foo-bar-task-id', 'run_id': 'scheduled__2023-03-08T0130000000-77fa57adf', 'kubernetes_pod_operator': 'True', 'try_number': '1'}
```
이때 production의 Pod의 레이블은 다음과 같다:
```JSON
{
 "dag_id": "foo-bar-dag-id", 
 "task_id": "foo-bar-task-id", 
 "run_id": "scheduled__2023-03-08T0130000000-77fa57adf", 
 "kubernetes_pod_operator": "True", 
 "try_number": "1"
}
```

apache-provider-cncf-kubernetes의 K8s 클라이언트는 Pod을 생성할때(`reattach` 옵션 False일 경우), 생성할 Pod의 레이블로 Pod 존재 여부를 확인하지 않기 때문에 충분히 중복 생성이 가능하다. 그러므로 2개의 Pod가 아무 문제 없이 생성되지만 아래의 에러가 발생한다.

## 에러 발생

```text
[2023-03-08T09:14:48.148+0000] {% raw %}{{kubernetes_pod.py:872}}{% endraw %} ERROR - 'NoneType' object has no attribute 'metadata'
Traceback (most recent call last):
  File "/usr/local/airflow/.local/lib/python3.10/site-packages/airflow/providers/cncf/kubernetes/operators/kubernetes_pod.py", line 528, in execute_sync
    self.remote_pod = self.find_pod(self.pod.metadata.namespace, context=context)
  File "/usr/local/airflow/.local/lib/python3.10/site-packages/airflow/providers/cncf/kubernetes/operators/kubernetes_pod.py", line 475, in find_pod
    raise AirflowException(f"More than one pod running with labels {label_selector}")
airflow.exceptions.AirflowException: More than one pod running with labels dag_id=export-to-skt-searchcell,kubernetes_pod_operator=True,run_id=scheduled__2023-03-08T0130000000-77fa57adf,task_id=export_action_log,already_checked!=True,!airflow-worker

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/airflow/.local/lib/python3.10/site-packages/airflow/providers/cncf/kubernetes/operators/kubernetes_pod.py", line 715, in patch_already_checked
    name=pod.metadata.name,
AttributeError: 'NoneType' object has no attribute 'metadata'
```
*실제 발생한 에러 메시지와 스택트레이스*

## 코드 레벨 분석

Pod 생성 이후부터 KubernetesOperator는 Pod 생명주기를 관리(예: Pod 회수)하고, Pod 풀에서 관리할 Pod를 탐색할때, 아래와 같이 레이블로 검색한다.

```python
def execute_sync(self, context: Context):
    try:
        self.pod_request_obj = self.build_pod_request_obj(context)
        # Pod을 생성한다.
        self.pod = self.get_or_create_pod(  # must set `self.pod` for `on_kill`
            pod_request_obj=self.pod_request_obj,
            context=context,
        )
        # 완료된 Pod를 회수하기 위해 원격의 Pod를 미리 가져온다.
        self.remote_pod = self.find_pod(self.pod.metadata.namespace, context=context)
        self.await_pod_start(pod=self.pod)

 ... # 생략
    

  finally:
		  # 에러가 나면 무조건 Pod를 정리한다.
      self.cleanup(
          pod=self.pod or self.pod_request_obj,
          remote_pod=remote_pod,
      )

... # 생략
```
*KubernetesOperator의 Pod 생명주기 관리, `providers.cncf.operators.kubernetes_pod.py`*

```python

 def find_pod(self, namespace: str, context: Context, *, exclude_checked: bool = True) -> k8s.V1Pod | None:
    """Returns an already-running pod for this task instance if one exists."""
    label_selector = self._build_find_pod_label_selector(context, exclude_checked=exclude_checked)
    # Pod 레이블로 Pod를 조회한다.
    pod_list = self.client.list_namespaced_pod(
        namespace=namespace,
        label_selector=label_selector,
    ).items
    
    pod = None
    num_pods = len(pod_list)
    if num_pods > 1: # <- Pod가 1개 이상 조회되면 예외를 발생한다.
        raise AirflowException(f'More than one pod running with labels {label_selector}')
```
*Pod 조회 및 중복 검증 로직, `providers.cncf.operators.kubernetes_pod.py`*


production 환경에서 Operator는 `find_pod`에서 2개의 Pod 조회 결과로 인해 AirflowException이 발생한다. 예외가 발생 후 production Operator는 더 이상 정상 진행하지 않고 `cleanup` 함수를 호출하여 Pod 회수를 시도한다.

```python
def cleanup(self, pod: k8s.V1Pod, remote_pod: k8s.V1Pod):
    pod_phase = remote_pod.status.phase if hasattr(remote_pod, "status") else None
    if pod_phase != PodPhase.SUCCEEDED or not self.is_delete_operator_pod:
        self.patch_already_checked(remote_pod, reraise=False)
    if pod_phase != PodPhase.SUCCEEDED:
        if self.log_events_on_failure:
            self._read_pod_events(pod, reraise=False)
        self.process_pod_deletion(remote_pod, reraise=False)
        error_message = get_container_termination_message(remote_pod, self.base_container_name)
        error_message = "\n" + error_message if error_message else ""
        raise AirflowException(
            f"Pod {pod and pod.metadata.name} returned a failure:\n{error_message}\n"
            f"remote_pod: {remote_pod}"
        )
    else:
        self.process_pod_deletion(remote_pod, reraise=False)
```
*Pod 정리 로직과 에러 발생 지점, `providers.cncf.operators.kubernetes_pod.py`*


여기서 `cleanup` 함수를 호출하여 Pod 회수를 시도할때, 넘어오는 `remote_pod`의 값은 None이다(예외가 발생하여 할당 실패). `patch_already_checked` 함수는 Pod에 레이블을 추가하는데 이 과정에서 AttributeError가 발생하여 회수도 실패한다. 그래서 `remote_pod`는 클라이언트가 관리하지 못하는 미아 상태의 Pod로 남게된다.

## 문제 결과

production Pod는 최종 실패로 남지만, 한국시각 18시 34분(작업 약 11분 경과) 경 태스크 종료와 함께, alpha Pod는 정상 종료된다(회수까지 완료).

```text
2023-03-08T09:34:34.236+0000 {% raw %}{{local_task_job.py:159}}{% endraw %} INFO - Task exited with return code 0
```

## 회고

이것은 **명백한 Apache Airflow Provider 패키지의 버그**였다. 공유 Kubernetes 환경에서 동일한 레이블을 가진 여러 Pod가 생성될 때 cleanup 로직이 제대로 작동하지 않는 치명적인 결함이었다.

이 버그의 핵심 문제점은 다음과 같다:

1. **중복 Pod 검증 로직의 불완전성**: `find_pod` 메소드에서 여러 Pod 발견 시 예외를 발생시키지만, 이로 인해 `remote_pod`가 `None`으로 설정되어 cleanup이 불가능해진다.

2. **에러 핸들링의 연쇄 실패**: 첫 번째 예외(`AirflowException`) 발생 후 cleanup 과정에서 또 다른 예외(`AttributeError`)가 발생하여 리소스 정리가 완전히 실패한다.

3. **미아 Pod 생성**: 결과적으로 관리되지 않는 Pod가 클러스터에 남게 되어 리소스 누수가 발생한다.

## 버그 해결 현황

다행히 이 문제는 Apache Airflow 커뮤니티에서 이미 인식하고 있던 버그였다. 이슈를 제기하려고 했으나 이미 다음과 같이 해결 작업이 진행되고 있음을 확인했다:

**해결된 버전:**
- **apache-airflow-providers-cncf-kubernetes 8.0.1rc1** 버전에서 패치 완료

**관련 링크:**
- [GitHub PR: Fix cleanup logic for duplicate pod scenarios](https://github.com/apache/airflow/pull/37671)
- [Apache Airflow Release Notes - CNCF Kubernetes Provider 8.0.1](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/8.0.1/changelog.html)

**권장사항:**
현재 `apache-airflow-providers-cncf-kubernetes` 8.0.1 이상 버전으로 업그레이드하면 이 문제가 해결된다. 하지만 근본적으로는 환경별 네임스페이스 분리를 통해 이런 상황 자체를 예방하는 것이 가장 확실한 해결책이다.

## 결론과 교훈

이번 문제를 통해 얻은 가장 큰 교훈은 “예외 상황에 대한 방어적 설계”의 중요성이었다. Airflow Provider 패키지의 개발자들은 분명히 중복 Pod 생성을 방지하려는 의도로 예외를 던지도록 설계했지만, 정작 그 예외가 발생했을 때의 후속 처리는 충분히 고려하지 못했다. 이는 우리가 시스템을 설계할 때도 마찬가지로 적용되는 교훈이다. “이런 상황은 절대 발생하지 않을 것”이라는 가정보다는 “만약 이런 상황이 발생한다면 어떻게 우아하게 처리할 것인가”를 항상 고민해야 한다는 것이다.
특히 인상 깊었던 것은 첫 번째 에러를 처리하려다가 두 번째 에러가 발생하는 연쇄 반응이었다. `find_pod`에서 `AirflowException`이 발생한 후 `cleanup` 과정에서 `AttributeError`가 추가로 발생하면서 원래 목적인 리소스 정리가 완전히 실패한 것을 보면서, 에러 핸들링 로직 자체가 또 다른 에러의 원인이 되어서는 안 된다는 점을 깨달았다. 방어 코드가 오히려 시스템을 더 취약하게 만들 수도 있다는 역설적인 상황이었다.

이번 경험을 통해 멀티 환경에서 공유 자원을 사용할 때의 위험성과 적절한 격리의 중요성을 깊이 인식하게 되었으며, 오픈소스 소프트웨어 사용 시 커뮤니티와의 적극적인 소통이 얼마나 중요한지 알 수 있었다.




