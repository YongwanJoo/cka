# 문제 풀이 - Service 생성하기

> [Notion 원본](https://app.notion.com/p/3e847fbe9c2a80a7ac2df7dce90a02b6)

<details>
<summary>사전 설치 명령어</summary>

```bash
kubectl create ns sp-culator
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/service/deployment.yaml
```

</details>
<details>
<summary>리소스 정리</summary>

```bash
kubectl delete ns sp-culator
```

</details>

## 문제 요구사항

1. `sp-culator` Namespace의 기존 Deployment `front-end`에서 **nginx 컨테이너의 80/TCP 포트를 명시**
2. `front-end-svc`라는 Service를 생성해 **Service port 80 → Pod targetPort 80**으로 연결
3. Service 타입을 **NodePort**로 설정해 Node의 포트를 통해 접근 가능하게 구성

> **주의:** NodePort가 Pod마다 하나씩 생성되는 것은 아님. 하나의 NodePort Service가 모든 Node의 동일한 NodePort를 열고, 들어온 트래픽을 selector에 매칭되는 Ready Pod 중 하나로 전달함.

## 핵심 개념

### containerPort와 targetPort는 다른 개념

Deployment의 컨테이너에는 다음을 추가해야 함.

```yaml
containers:
- name: nginx
  ports:
  - containerPort: 80
    protocol: TCP
```

- `containerPort: 80`: 해당 컨테이너가 사용하는 포트를 **Pod 명세에 선언**
- `targetPort: 80`: Service가 트래픽을 전달할 **Pod의 포트**
- `port: 80`: 클라이언트가 Service에 접근할 때 사용하는 포트
- `nodePort`: NodePort 타입에서 각 Node에 열리는 외부 접근용 포트

> `containerPort` 자체가 nginx 프로세스의 포트를 열어주는 것은 아님. 애플리케이션이 실제로 해당 포트에서 listen하고 있어야 함. 또한 Service는 숫자 `targetPort`를 사용한다면 `containerPort` 선언이 없어도 전달할 수 있음. 하지만 **이 문제에서는 Deployment에 80/TCP를 노출하라고 명시했으므로 containerPort를 추가해야 함.**

## 풀이

### 1. 기존 Deployment와 label 확인

```bash
kubectl get deployment front-end -n sp-culator
kubectl get deployment front-end -n sp-culator -o yaml
kubectl get pods -n sp-culator --show-labels
```

먼저 다음을 확인해야 함.

- 컨테이너 이름이 `nginx`인지
- Deployment의 `spec.selector.matchLabels`
- Pod template의 label
- 기존에 `ports` 항목이 존재하는지

### 2. Deployment에 containerPort 80/TCP 추가

```bash
kubectl edit deployment front-end -n sp-culator
```

`spec.template.spec.containers`에서 `name: nginx`인 컨테이너에 다음을 추가.

```yaml
ports:
- containerPort: 80
  protocol: TCP
```

예시 구조:

```yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

저장하면 Deployment의 Pod template이 변경되므로 기존 Pod가 새로운 ReplicaSet의 Pod로 교체됨.

### 3. NodePort Service 생성

CKA에서는 Deployment를 그대로 노출시키는 명령이 가장 빠름.

```bash
kubectl expose deployment front-end   -n sp-culator   --name=front-end-svc   --type=NodePort   --port=80   --target-port=80
```

장점:

- Deployment의 selector를 자동으로 사용하므로 **Service selector를 직접 잘못 입력할 가능성을 줄일 수 있음**
- `nodePort`를 직접 지정하지 않으면 Kubernetes가 사용 가능한 포트를 자동 할당함

### YAML로 생성하는 경우

```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-end-svc
  namespace: sp-culator
spec:
  type: NodePort
  selector:
    app: front-end
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

> `selector.app: front-end`는 무조건 외워서 쓰는 값이 아님. 반드시 기존 Deployment의 `spec.selector.matchLabels` 또는 Pod label을 확인한 뒤 동일하게 작성해야 함.

## 검증

### 1. Deployment의 containerPort 확인

```bash
kubectl get deployment front-end -n sp-culator -o yaml
```

또는 필요한 부분만 확인:

```bash
kubectl get deployment front-end -n sp-culator   -o jsonpath='{.spec.template.spec.containers[?(@.name=="nginx")].ports}'
```

### 2. Service 확인

```bash
kubectl get svc front-end-svc -n sp-culator
```

예시:

```text
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
front-end-svc   NodePort   10.102.11.106   <none>        80:32071/TCP
```

여기서:

- `80`: Service port
- `32071`: 자동 할당된 NodePort

기본 NodePort 범위는 `30000-32767`.

### 3. EndpointSlice 확인

현재 Kubernetes에서는 기존 Endpoints보다 EndpointSlice를 확인하는 것이 적절함.

```bash
kubectl get endpointslice -n sp-culator   -l kubernetes.io/service-name=front-end-svc -o wide
```

EndpointSlice에 `front-end` Pod들의 IP와 80번 포트가 등록되어 있으면 selector와 targetPort 연결이 정상임.

### 4. NodePort로 실제 접근 확인

먼저 Node IP 확인:

```bash
kubectl get nodes -o wide
```

그 후:

```bash
curl <Node-IP>:<NodePort>
```

예:

```bash
curl 192.168.64.10:32071
```

nginx Welcome 페이지가 반환되면 정상.

> 실습 환경에서는 `curl localhost:32071`도 동작했지만, NodePort의 본래 접근 방식은 **NodeIP:NodePort**이므로 시험 노트에서는 Node IP를 이용한 검증을 기준으로 기억하는 것이 안전함.

## 시험장에서의 최소 풀이 순서

```bash
# 1. 리소스 확인
k get deploy front-end -n sp-culator -o yaml
k get po -n sp-culator --show-labels

# 2. Deployment 수정
k edit deploy front-end -n sp-culator
# nginx 컨테이너에 containerPort: 80, protocol: TCP 추가

# 3. Service 생성
k expose deploy front-end -n sp-culator   --name=front-end-svc   --type=NodePort   --port=80   --target-port=80

# 4. 검증
k get deploy front-end -n sp-culator
k get svc front-end-svc -n sp-culator
k get endpointslice -n sp-culator   -l kubernetes.io/service-name=front-end-svc
```

> **이 문제에서 기억할 패턴:** Deployment에 containerPort 추가 → `kubectl expose deployment`로 NodePort Service 생성 → Service와 EndpointSlice 검증
