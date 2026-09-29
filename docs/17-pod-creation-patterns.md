# Pod 생성 방법과 디자인 패턴

> [Notion 원본](https://app.notion.com/p/3e947fbe9c2a80ddac18f3ec342ab8d6)

> **한눈에 보기:** Pod 생성 경로 확인 → API Server·Scheduler·kubelet의 역할 파악 → 같은 Pod에 묶을 컨테이너 선택 → Init/Sidecar 패턴 적용

## 1. Pod는 어떻게 생성되는가

- **직접 생성:** `kubectl run` 또는 Pod 매니페스트에 `kubectl apply -f` 사용. 실습에 빠르지만, 컨트롤러가 없는 Pod를 삭제하면 자동으로 대체 Pod가 생성되지 않음.
- **워크로드 컨트롤러:** Deployment·StatefulSet·DaemonSet·Job 등이 Pod를 생성하고 원하는 개수를 관리. 일반적인 서비스는 이 경로를 사용하며, Deployment는 ReplicaSet을 거쳐 Pod를 관리함.
- **Static Pod:** Node의 kubelet이 로컬 파일을 감시해 직접 실행. kubeadm 기반 Control Plane에서는 보통 `/etc/kubernetes/manifests`에 핵심 컴포넌트의 매니페스트가 있음.

### 일반 Pod의 생성 흐름

1. `kubectl` 또는 컨트롤러가 **API Server**에 Pod 생성 요청.
2. API Server가 요청을 처리하고 Pod 객체를 **etcd**에 저장.
3. **Scheduler**가 아직 Node가 지정되지 않은 Pod에 적합한 Node를 배정.
4. 해당 Node의 **kubelet**이 Pod 스펙을 확인하고 CRI를 통해 container runtime에 Pod sandbox와 컨테이너 실행을 요청.
5. 런타임이 CNI와 연동해 네트워크를 준비하고, kubelet이 Pod 상태를 API Server에 보고.
> **Static Pod는 예외:** kubelet이 로컬 매니페스트를 원본으로 사용하므로 일반 Pod의 API 생성·Scheduler 배정 흐름을 거치지 않음. API Server에 나타나는 mirror Pod를 삭제해도 원본 파일이 남아 있으면 kubelet이 계속 관리함.

### CKA에서 빠르게 만드는 방법

```bash
# 방법 A: Pod 직접 생성
kubectl run web --image=nginx:stable --port=80

# 방법 B: YAML 초안을 만든 뒤 수정·적용
kubectl run web --image=nginx:stable --port=80 --dry-run=client -o yaml > pod.yaml
kubectl apply -f pod.yaml

# 반복 실행할 앱은 Deployment로 관리
kubectl create deployment web --image=nginx:stable --replicas=2
```

방법 A와 B는 같은 Pod를 만드는 대안임. 하나를 선택해 사용하면 됨.

## 2. 컨테이너를 같은 Pod에 묶는 기준

- Pod는 **같은 Node에 함께 배치**되는 하나 이상의 컨테이너를 담음. 같은 Pod의 컨테이너는 Pod IP와 포트 공간을 공유해 `localhost`로 통신할 수 있음.
- 컨테이너의 파일시스템은 자동으로 공유되지 않음. 파일을 함께 쓰려면 `volume`을 정의하고 각 컨테이너에 마운트해야 함.
- 한 서비스만 따로 배포하거나 확장해야 한다면 별도 Pod가 적합함. 함께 시작·배치·종료해야 하는 밀접한 구성 요소를 같은 Pod에 둠.

### Pause Container: Pod의 실행 기반

- 런타임이 만드는 인프라 컨테이너로, Pod sandbox의 namespace를 유지하는 역할을 함. 사용자가 일반 애플리케이션 컨테이너처럼 Pod YAML의 `containers`에 작성하지 않음.
- Pod 네트워크와 IP 설정은 런타임이 CNI와 연동해 처리함. Pause Container 자체가 IP를 할당하는 것은 아님.

### Init Container: 시작 전에 끝내는 작업

- `spec.initContainers`에 선언하며, **순서대로 실행해 성공적으로 종료**한 뒤 앱 컨테이너가 시작됨.
- 설정 파일 생성, 공유 볼륨의 파일 준비, 권한 조정, 의존 서비스 대기 등에 사용.
- Init Container가 실패하면 주 컨테이너가 시작되지 않으므로 해당 컨테이너의 로그와 Pod Events를 확인해야 함.

### Sidecar Container: 실행 중 계속 보조

- 로그 전달, 로컬 프록시, 설정 동기화처럼 주 애플리케이션을 계속 보조함.
- **네이티브 Sidecar:** `spec.initContainers`에 `restartPolicy: Always`를 지정. 앱 컨테이너보다 먼저 시작하고 Pod 수명 동안 계속 실행됨.
- **일반 보조 컨테이너:** `spec.containers`에 앱과 함께 선언할 수도 있지만 배열의 나열 순서가 시작 순서를 보장하지는 않음.
- **Ambassador 패턴:** 외부 서비스로 향하는 요청을 대신 처리하는 Pod 내부 프록시. 앱은 `localhost`로 접속하고 프록시가 연결·TLS 등을 담당.
- **Adapter 패턴:** 앱이 만든 로그·메트릭·데이터를 외부 시스템이 원하는 형식으로 변환.
- Sidecar, Ambassador, Adapter는 **보조 역할의 디자인 패턴**이고, Pause Container는 **런타임의 인프라 구성 요소**임.

## 3. Init + 네이티브 Sidecar 실습

다음 예시에서 `prepare`는 파일을 만들고 종료함. `heartbeat`는 계속 실행되고, `web`은 두 컨테이너가 준비한 공유 볼륨을 사용함.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-patterns
spec:
  volumes:
  - name: work
    emptyDir: {}
  initContainers:
  - name: prepare
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Pod patterns' > /work/index.html"]
    volumeMounts:
    - name: work
      mountPath: /work
  - name: heartbeat
    image: busybox:1.36
    restartPolicy: Always
    command: ["sh", "-c", "while true; do date > /work/heartbeat.txt; sleep 10; done"]
    volumeMounts:
    - name: work
      mountPath: /work
  containers:
  - name: web
    image: nginx:stable
    ports:
    - containerPort: 80
    volumeMounts:
    - name: work
      mountPath: /usr/share/nginx/html
```

`emptyDir`의 데이터는 컨테이너 재시작 중에는 유지되지만 Pod가 삭제되면 사라짐. 네이티브 Sidecar는 Kubernetes v1.33부터 안정화된 기능임.

```bash
kubectl apply -f pod-patterns.yaml
kubectl wait --for=condition=Ready pod/pod-patterns --timeout=120s
kubectl describe pod pod-patterns
kubectl logs pod-patterns -c prepare
kubectl logs pod-patterns -c heartbeat
kubectl exec pod-patterns -c web -- cat /usr/share/nginx/html/index.html
```

## 4. Pod가 시작되지 않을 때

1. `kubectl get pod -o wide`로 상태와 배정 Node 확인.
2. `kubectl describe pod <name>`의 Events에서 스케줄링·이미지·볼륨 오류 확인.
3. `kubectl logs <name> -c <container>`로 컨테이너별 로그 확인. 재시작 직전 로그는 `--previous` 사용.
4. Init 단계에서 멈췄다면 `kubectl logs <name> -c <init-container>` 확인.
5. Static Pod라면 해당 Node의 `/etc/kubernetes/manifests`와 kubelet 상태 확인.

### 공식 문서

- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) · [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) · [Static Pods](https://kubernetes.io/docs/concepts/workloads/pods/static-pods/)
- [멀티 컨테이너 패턴 개요](https://kubernetes.io/blog/2025/04/22/multi-container-pods-overview/)
