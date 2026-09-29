# 문제풀이 - Core Components 장애 해결하기

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a800a8d12de4ae151d17c)

<details>
<summary>장애 재현 명령어 (실습용)</summary>

```bash
sudo sed -i 's|--etcd-servers=https://127.0.0.1:2379|--etcd-servers=https://192.168.56.41:2379|' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i 's/cpu: 100m/cpu: 4/' /etc/kubernetes/manifests/kube-scheduler.yaml
```

</details>

## 요구 사항

1. single-node cluster를 고치기
2. 현재 정상 기능을 못하는 components를 조사하고 망가진 원인 밝히기
3. 문제 설명의 etcd 연결 정보를 확인하고, 실제 control-plane 설정과 비교하기
4. 필요한 서비스들 전부 재시작하기
이 실습 노드에는 `etcd.yaml`이 있고, 정상 API Server 설정의 etcd 주소는 `127.0.0.1:2379`였다. 문제 설명의 “외부 etcd” 문구만 따르지 말고 실제 매니페스트를 기준으로 복구한다.

## 풀이

### 1. API Server 접속 상태 확인

```bash
kubectl get nodes -o wide
# The connection to the server 192.168.56.40:6443 was refused
```

`6443` 연결 거부가 발생했으므로 먼저 control-plane의 API Server 설정을 확인한다.

### 2. control-plane manifest 확인

```bash
hostname
sudo ls -al /etc/kubernetes/manifests
```

확인된 파일

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

### 3. kube-apiserver의 etcd 주소 복구

사전 장애 주입 명령에서 etcd 주소가 아래와 같이 변경되어 있었다.

```text
https://127.0.0.1:2379
→ https://192.168.56.41:2379
```

`kube-apiserver.yaml`을 열어 etcd 주소를 원래 값으로 복구한다.

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

```yaml
- --etcd-servers=https://127.0.0.1:2379
```

이후 다시 확인한다.

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

API Server가 `1/1 Running`으로 복구되면서 `kubectl` 명령도 다시 동작한다.

### 4. kube-scheduler 추가 장애 확인

API Server 복구 후 `kube-scheduler`가 다음 상태로 확인됐다.

```text
kube-scheduler-k8s-master   0/1   UnexpectedAdmissionError
```

노드 자원을 확인한다.

```bash
kubectl describe node k8s-master
```

확인 결과 노드의 CPU Capacity와 Allocatable은 `4`였고, 기존 Pod들이 이미 약 `950m`의 CPU request를 사용하고 있었다.
사전 장애 주입 명령으로 scheduler의 CPU request가 `100m`에서 `4`로 변경되어 있었으므로, scheduler가 노드에 올라갈 수 없는 상태였다.

### 5. kube-scheduler CPU request 복구

```bash
sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
```

CPU request를 원래 값으로 복구한다.

```yaml
resources:
  requests:
    cpu: 100m
```

복구 확인

```bash
kubectl get pods -A
```

```text
kube-apiserver-k8s-master   1/1   Running
kube-scheduler-k8s-master   1/1   Running
```

### 6. kubelet 재시작 및 최종 확인

```bash
sudo systemctl restart kubelet
kubectl get pods -A
```

kubelet 재시작 직후 로그에서는 control-plane Pod의 READY가 일시적으로 `0/1`로 확인됐다.
Static Pod 매니페스트 수정은 kubelet이 감지한다. 이 단계의 재시작은 문제 요구사항에 맞춰 수행한 것이며, 매니페스트 변경 때마다 필수인 절차는 아니다.

## 연관 개념

- **API Server 6443 포트**: 이번 장애에서는 `192.168.56.40:6443` 연결 거부가 control-plane 이상을 확인하는 첫 단서였다.
- **etcd 2379 포트**: `kube-apiserver`가 etcd에 연결할 endpoint가 잘못 설정되어 API Server가 정상 동작하지 못했다.
- **control-plane manifest 경로**: 이 실습 환경에서는 `/etc/kubernetes/manifests`에서 `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` 설정을 확인했다.
- **Resource Request**: 노드 CPU가 `4`인데 scheduler가 CPU `4`를 요청하도록 설정되면서 기존 Pod의 request와 함께 수용할 수 없어 `UnexpectedAdmissionError`가 발생했다.

### 시험용 장애 해결 순서

```text
kubectl 연결 실패
→ control-plane 노드인지 확인
→ /etc/kubernetes/manifests 확인
→ kube-apiserver 설정 확인
→ API Server 복구
→ kubectl get pods -A
→ 비정상 Core Component 확인
→ kubectl describe node로 자원 확인
→ 해당 manifest 수정
→ kubelet 재시작
→ kubectl get pods -A로 검증
```

### 공식 문서

- [Static Pod 생성과 동작](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
