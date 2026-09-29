# 문제풀이 - 로그 출력 Sidecar 생성

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a80b0b944fb66c348824d)

<details>
<summary>사전 설치 명령어</summary>

```bash
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/sidecar/deployment.yaml
```

</details>
<details>
<summary>리소스 정리</summary>

```bash
kubectl delete deploy synergy-deployment
```

</details>

## 문제 요구사항

1. 기존 Deployment `synergy-deployment`의 애플리케이션 로그 파일 `/var/log/synergy-deployment.log`를 Sidecar 컨테이너에서 실시간으로 출력
2. 메인 컨테이너와 Sidecar가 같은 `/var/log` 경로를 공유하도록 볼륨 구성
3. Sidecar는 `busybox:stable` 이미지를 사용하고 `tail -n+1 -f`로 로그 파일을 지속적으로 읽도록 설정
4. 변경 후 Pod가 `2/2 Running` 상태인지 확인하고 `kubectl logs -c sidecar`로 로그 출력 검증

## 핵심 개념

### Sidecar 패턴

Sidecar는 **메인 컨테이너와 같은 Pod 안에서 보조 기능을 수행하는 컨테이너**임.
이 문제에서는 역할이 다음과 같음.
- 메인 컨테이너: `/var/log/synergy-deployment.log` 파일에 로그 기록
- Sidecar: 같은 파일을 `tail -f`로 읽어 표준 출력으로 전달
- 결과: `kubectl logs <Pod> -c sidecar`로 애플리케이션 로그 확인 가능

### emptyDir 공유 볼륨

컨테이너마다 파일 시스템은 기본적으로 분리되어 있으므로, 메인 컨테이너가 만든 로그 파일을 Sidecar가 바로 볼 수 없음.
따라서 두 컨테이너가 동일한 `emptyDir` 볼륨을 `/var/log`에 마운트함.

```yaml
volumes:
- name: data
  emptyDir: {}
```

`emptyDir`은 **Pod가 생성될 때 만들어지고 Pod가 삭제되면 함께 사라지는 임시 볼륨**임. 같은 Pod 안의 여러 컨테이너가 데이터를 공유할 때 사용할 수 있음.

### restartPolicy: Always를 사용하는 Sidecar

현재 실습 환경은 Kubernetes v1.34.3이며, `restartPolicy: Always`를 가진 Sidecar는 `initContainers` 아래에 정의하는 **native sidecar container** 방식임.
일반 init container와 달리 초기화 작업이 끝나도 종료되지 않고 메인 컨테이너와 함께 계속 실행됨.
> `restartPolicy: Always`를 일반 `containers` 항목에 넣는 것이 아니라, Sidecar를 `initContainers`에 정의할 때 사용한다고 기억하면 됨.

## 풀이

### 1. 기존 Pod와 Deployment 확인

```bash
k get pods -o wide
k get deploy synergy-deployment -o yaml
```

초기 상태에서는 Pod가 `1/1 Running`으로 실행 중이었음.

### 2. Deployment 수정

```bash
k edit deploy synergy-deployment
```

기존 메인 컨테이너에는 공유 볼륨을 `/var/log`에 마운트함.

```yaml
volumeMounts:
- name: data
  mountPath: /var/log
```

Sidecar를 추가하고 같은 볼륨을 마운트함.

```yaml
initContainers:
- name: sidecar
  image: busybox:stable
  restartPolicy: Always
  command:
  - sh
  - -c
  - tail -n+1 -f /var/log/synergy-deployment.log
  volumeMounts:
  - name: data
    mountPath: /var/log
```

Pod 수준에는 `emptyDir` 볼륨을 추가함.

```yaml
volumes:
- name: data
  emptyDir: {}
```

위 세 항목은 모두 Deployment의 `spec.template.spec` 아래에 적용한다. 기존 메인 컨테이너의 이미지와 명령은 유지한다.

### 3. 새 Pod 생성 확인

Deployment의 Pod template을 수정하면 새로운 ReplicaSet이 생성되고 기존 Pod는 교체됨.

```bash
k get pods
```

실습 결과:

```text
NAME                                   READY   STATUS
synergy-deployment-64488f46b8-mlcd9   2/2     Running
```

`2/2`가 표시되므로 메인 컨테이너와 Sidecar가 모두 정상 실행 중임.

### 4. Sidecar 로그 확인

```bash
k logs synergy-deployment-64488f46b8-mlcd9 -c sidecar
```

출력:

```text
logging
logging
logging
logging
...
```

Sidecar가 공유 볼륨의 `synergy-deployment.log`를 정상적으로 읽고 있음을 확인할 수 있음.

## 명령어 해석

```bash
tail -n+1 -f /var/log/synergy-deployment.log
```

- `-n+1`: 파일의 첫 번째 줄부터 출력
- `-f`: 이후 파일에 새 로그가 추가되면 계속 따라가며 출력
- 따라서 기존 로그와 새로 추가되는 로그를 모두 Sidecar의 stdout으로 전달

## 시험장에서의 최소 풀이 순서

```bash
# 1. 기존 Deployment 확인
k get deploy synergy-deployment -o yaml

# 2. 수정
k edit deploy synergy-deployment

# 기존 메인 컨테이너
# volumeMounts:
# - name: data
#   mountPath: /var/log

# native sidecar
# initContainers:
# - name: sidecar
#   image: busybox:stable
#   restartPolicy: Always
#   command: ['sh', '-c', 'tail -n+1 -f /var/log/synergy-deployment.log']
#   volumeMounts:
#   - name: data
#     mountPath: /var/log

# 공유 볼륨
# volumes:
# - name: data
#   emptyDir: {}

# 3. Pod 상태 확인
k get pods

# 4. Sidecar 로그 확인
k logs <Pod-Name> -c sidecar
```

> **이 문제에서 기억할 패턴:** 메인 컨테이너와 Sidecar에 동일한 `emptyDir` 마운트 → Sidecar가 `tail -f`로 로그 파일 추적 → `kubectl logs -c sidecar`로 검증

### 공식 문서

- [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
