# 문제풀이 - CPU & memory 재설정

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a807d8577facfbe314baf)

<details>
<summary>사전 설치</summary>

```bash
kubectl create ns relative-fawn
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/resource/deployment-mariadb.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/resource/deployment-wordpress.yaml
```

</details>
<details>
<summary>리소스 정리</summary>

```bash
kubectl delete ns relative-fawn
```

</details>

## 문제 요구 사항

- namespace: `relative-fawn`
- WordPress Deployment의 replica는 최종적으로 **3개를 유지**해야 한다.
- 문제에서 주어진 가용 자원은 CPU `1`, Memory `1024Mi`이다.
- 3개의 Pod가 CPU와 Memory를 공평하게 요청하도록 `resources.requests`를 설정한다.
- 노드가 안정적으로 동작할 수 있도록 자원을 정확히 3등분하지 않고 일부 여유 자원을 남긴다.
- 일반 Container와 Init Container가 있다면 **동일한 request 값**을 적용한다.
- 기존 `resources.limits`는 변경할 필요가 없다.
- 작업 후 WordPress Pod 3개가 모두 `Running`, `Ready`인지 확인한다.

## 자원 계산

단순히 3등분하면 다음과 같다.

```text
CPU    : 1000m / 3 ≈ 333m
Memory : 1024Mi / 3 ≈ 341Mi
```

문제에서 노드 안정성을 위한 overhead를 남기라고 했으므로 각 Pod의 request를 다음과 같이 설정했다.

```text
Pod 1: CPU 300m / Memory 300Mi
Pod 2: CPU 300m / Memory 300Mi
Pod 3: CPU 300m / Memory 300Mi

총 요청량: CPU 900m / Memory 900Mi
남는 자원: CPU 100m / Memory 124Mi
```

## 풀이

### 1. Deployment 상태 확인

```bash
kubectl get deploy -n relative-fawn
kubectl get pods -n relative-fawn
```

초기 상태에서는 WordPress replica 3개 중 1개가 `Pending` 상태였다.

### 2. WordPress Deployment를 임시로 0 replica로 축소

문제에서 필요하면 업데이트 중 WordPress를 0 replica로 줄여도 된다고 안내하고 있다.

```bash
kubectl scale deploy wordpress -n relative-fawn --replicas=0
kubectl get deploy -n relative-fawn
```

확인 결과:

```text
wordpress   0/0   0   0
```

### 3. resource requests 수정

```bash
kubectl edit deploy wordpress -n relative-fawn
```

WordPress Container의 `resources.requests`를 다음과 같이 설정한다.

```yaml
resources:
  limits:
    cpu: "1"
    memory: 1000Mi
  requests:
    cpu: 300m
    memory: 300Mi
```

`limits`는 기존 값을 그대로 유지하고 `requests`만 변경한다.
이번 실습의 WordPress Deployment에는 `initContainers`가 없으므로 추가로 수정할 대상이 없었다. 실제 시험 문제에서 Init Container가 존재한다면 일반 Container와 동일하게 다음 request를 설정한다.

```yaml
resources:
  requests:
    cpu: 300m
    memory: 300Mi
```

### 4. WordPress를 다시 3 replica로 복구

```bash
kubectl scale deploy wordpress -n relative-fawn --replicas=3
```

### 5. 최종 검증

```bash
kubectl get pods -n relative-fawn
kubectl get deploy -n relative-fawn
```

실습 결과:

```text
wordpress-86c948c57b-6mqrr   1/1   Running
wordpress-86c948c57b-kxnfs   1/1   Running
wordpress-86c948c57b-t8nft   1/1   Running

NAME        READY   UP-TO-DATE   AVAILABLE
wordpress   3/3     3            3
```

따라서 **3 replicas 유지**, **모든 Pod Running**, **모든 Pod Ready** 조건을 만족한다.

## 시험장에서 기억할 흐름

```text
자원 계산
→ scale 0
→ Deployment의 requests 수정
→ scale 3
→ Pod Running/Ready 확인
```

> `kubectl edit deploy deployment`처럼 리소스 종류를 이름으로 입력하면 안 된다. 실제 Deployment 이름인 `wordpress`를 사용해야 한다.

## 관련 개념

- **requests**: Kubernetes Scheduler가 Pod를 어느 Node에 배치할지 판단할 때 사용하는 최소 요청 자원이다.
- **limits**: Container가 사용할 수 있는 최대 자원이다. 이 문제에서는 수정 대상이 아니다.
- **CPU 단위**: `1000m = 1 CPU`. 따라서 `300m = 0.3 CPU`이다.
- **Init Container**: 애플리케이션 Container보다 먼저 실행되는 Container다. 문제에서 동일한 request를 요구하면 Init Container에도 같은 값을 설정해야 한다.
- Deployment의 `status:` 영역은 Kubernetes Controller가 관리하므로 `kubectl edit`에서 직접 수정하지 않는다.
