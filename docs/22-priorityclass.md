# 문제풀이 - PriorityClass 생성

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a804f9741fb0b0e05ba01)

<details>
<summary>사전 설치</summary>

```bash
kubectl create ns priority
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/priorityclass/deployment-others.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/priorityclass/deployment-busybox.yaml
```

</details>
<details>
<summary>리소스 정리</summary>

```bash
k delete priorityclass high-priority
k delete ns priority
```

</details>

## 문제 요구사항

- `high-priority`라는 PriorityClass를 생성
- PriorityClass의 `value`는 **기존 user-defined PriorityClass 중 가장 높은 값보다 1 작게** 설정
- `priority` namespace의 `busybox-logger` Deployment만 수정하여 `high-priority`를 사용
- 변경 후 `busybox-logger` Deployment가 정상적으로 rollout되어야 함
- 자원이 부족할 경우 같은 namespace의 다른 Deployment Pod가 Preemption으로 축출될 수 있음
- **다른 Deployment 자체는 수정하지 않음**
> `system-cluster-critical`, `system-node-critical`은 Kubernetes 시스템 PriorityClass이므로 문제에서 말하는 user-defined PriorityClass의 최댓값을 구할 때 제외한다.

## 풀이

### 1. 기존 PriorityClass 확인

```bash
kubectl get priorityclass
kubectl get priorityclass -o custom-columns=NAME:.metadata.name,VALUE:.value
```

기존 **user-defined PriorityClass** 중 가장 높은 `VALUE`를 확인하고, 그 값에서 1을 뺀 값을 사용한다.
예를 들어 기존 user-defined 최고값이 `1000001`이라면 새 PriorityClass의 값은 `1000000`이다.

### 2. high-priority PriorityClass 생성

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000   # 실제 시험에서는 기존 user-defined 최고값 - 1
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "High priority for user workloads"
```

```bash
kubectl apply -f priorityclass.yaml
```

또는 stdin으로 바로 생성할 수 있다.

```bash
kubectl apply -f - <<EOF
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
EOF
```

> PriorityClass는 namespace에 속하지 않는 **cluster-scoped resource**이므로 `metadata.namespace`를 작성하지 않는다.

### 3. busybox-logger Deployment만 수정

```bash
kubectl edit deployment busybox-logger -n priority
```

Pod template의 `spec` 아래에 다음 값을 추가한다.

```yaml
spec:
  template:
    spec:
      priorityClassName: high-priority
      containers:
      - name: busybox
```

시험에서는 patch로 바로 처리해도 된다.

```bash
kubectl patch deployment busybox-logger -n priority \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'
```

> `kubectl edit -n priority busybox-logger`처럼 리소스 종류를 생략하면 안 된다. `busybox-logger`는 Deployment의 이름이므로 `kubectl edit deploy -n priority busybox-logger`처럼 실행한다.

### 4. rollout 성공 확인

```bash
kubectl rollout status deployment/busybox-logger -n priority
kubectl get deploy -n priority
kubectl get pods -n priority
```

이번 실습의 최종 결과:

```text
NAME             READY   UP-TO-DATE   AVAILABLE
busybox-logger   3/3     3            3
others           0/3     3            0
```

`busybox-logger`의 Pod 3개가 모두 실행되었고, 자원이 부족해 우선순위가 낮은 `others` Pod가 축출되었다. 이는 문제에서 기대한 Preemption 동작이다.

### 5. 최종 설정 검증

```bash
kubectl get priorityclass high-priority

kubectl get deployment busybox-logger -n priority \
  -o jsonpath='{.spec.template.spec.priorityClassName}{"\n"}'

kubectl rollout status deployment/busybox-logger -n priority
```

`priorityClassName` 출력이 다음과 같으면 정상이다.

```text
high-priority
```

## 시험에서 기억할 핵심

1. 먼저 `kubectl get priorityclass`로 기존 값을 확인한다.
2. `system-*` PriorityClass는 제외하고 user-defined 중 최고값을 찾는다.
3. 새 값은 **최고값 - 1**로 설정한다.
4. PriorityClass는 cluster-scoped이므로 namespace를 지정하지 않는다.
5. 수정 대상은 `busybox-logger` Deployment 하나뿐이다.
6. `priorityClassName` 변경은 Pod template 변경이므로 새 ReplicaSet과 Pod가 생성된다.
7. 높은 Priority Pod가 스케줄링될 자원이 없으면 낮은 Priority Pod가 Preemption으로 축출될 수 있다.
8. 마지막에 반드시 `kubectl rollout status`로 성공 여부를 확인한다.

### 공식 문서

- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
