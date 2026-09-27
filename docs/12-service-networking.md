# Service & Networking

> [Notion 원본](https://app.notion.com/p/3e847fbe9c2a80c5a5aec3c53284e9c6)

> **CKA Services & Networking (20%) 핵심 범위:** Pod 간 통신, NetworkPolicy, ClusterIP/NodePort/LoadBalancer, EndpointSlice, Gateway API, Ingress, CoreDNS를 실습 가능한 수준으로 숙지

## 1. Kubernetes 네트워크 기본

- 각 Pod는 클러스터 내부에서 고유한 IP를 가지며, 같은 Pod 안의 컨테이너끼리는 `localhost`로 통신할 수 있음
- CNI가 정상 구성되어 있다면 서로 다른 Node의 Pod끼리도 Pod IP를 이용해 직접 통신할 수 있음
- 다만 Pod는 삭제 및 재생성될 수 있는 일시적인 리소스이므로 **Pod IP를 서비스 주소로 직접 사용하면 안 됨**
- 여러 Pod에 안정적으로 접근하기 위해 **Service**를 사용함

> **핵심 흐름:** Client → Service의 안정적인 IP/DNS → EndpointSlice에 등록된 정상 Pod → targetPort

## 2. Service

Service는 여러 Pod를 하나의 안정적인 네트워크 엔드포인트로 묶는 L4 네트워크 추상화임.

### Selector와 EndpointSlice

- 일반적인 Service는 spec.selector의 label 조건으로 대상 Pod를 선택함
- Control Plane은 Service의 selector와 일치하는 Pod를 추적해 **EndpointSlice**를 자동으로 생성 및 갱신함
- Pod가 생성, 삭제되거나 readiness 상태가 변하면 EndpointSlice의 사용 가능한 endpoint도 바뀜
- kube-proxy 또는 이를 대체하는 네트워크 구현이 Service와 EndpointSlice를 감시하고 실제 트래픽 전달 규칙을 구성함
- 따라서 RollingUpdate 중에도 사용자는 Service 주소를 그대로 사용하고, 백엔드 Pod 목록만 자동으로 갱신됨
- 이를 별도의 "서비스 레지스트리" 기능이라고 부르기보다는 **Service + EndpointSlice 기반의 서비스 디스커버리 및 트래픽 전달**로 이해하는 것이 정확함

> Kubernetes v1.33부터 기존 Endpoints API는 deprecated 상태이며 현재는 **EndpointSlice**를 기준으로 이해하는 것이 좋음.

### Service Port

- port: 클라이언트가 Service에 접근할 때 사용하는 포트
- targetPort: Service가 실제 Pod로 전달할 때 사용하는 포트
- nodePort: NodePort 타입에서 각 Node에 열리는 포트

```yaml
ports:
- port: 80
  targetPort: 8080
```

위 설정은 Service:80 → Pod:8080으로 전달됨.

### Service Type

- **ClusterIP**
  - 기본값
  - 클러스터 내부에서만 접근 가능한 가상 IP를 제공
  - Pod 간 내부 서비스 통신에서 가장 일반적으로 사용
- **NodePort**
  - 모든 Node의 NodeIP:nodePort를 통해 Service에 접근
  - nodePort를 생략하면 자동 할당됨
  - 기본 할당 범위는 30000-32767
  - Node가 외부에서 접근 가능한 네트워크에 있어야 실제 외부 접근이 가능함
- **LoadBalancer**
  - 지원되는 클라우드 또는 LoadBalancer 구현과 연동해 외부 LoadBalancer를 프로비저닝
  - 일반적으로 NodePort/ClusterIP 기능을 바탕으로 외부 진입점을 추가함
- **ExternalName**
  - Pod selector 대신 외부 DNS 이름을 Service 이름으로 매핑
  - CKA에서는 ClusterIP, NodePort, LoadBalancer를 우선적으로 숙지

### Service Discovery와 CoreDNS

Kubernetes는 보통 **CoreDNS**를 이용해 Service 이름을 DNS로 해석함.

- 같은 Namespace: service-name
- 다른 Namespace: service-name.namespace
- FQDN: service-name.namespace.svc.cluster.local

예를 들어 prod Namespace의 api Service는 다른 Namespace에서 다음과 같이 접근할 수 있음.

```text
api.prod
api.prod.svc.cluster.local
```

즉, 다른 Namespace의 Service를 호출할 때는 Service 이름 앞에 다른 값을 붙이는 것이 아니라 **Service 이름 뒤에 Namespace를 붙임**.

### Headless Service

clusterIP: None으로 설정하면 Service의 가상 IP를 만들지 않고 DNS가 백엔드 Pod의 IP들을 직접 반환함. StatefulSet처럼 개별 Pod를 직접 식별해야 하는 경우에 주로 사용함.

### Service 확인 명령어

```bash
kubectl get svc
kubectl describe svc <service-name>
kubectl get pods --show-labels
kubectl get pods -l app=<label> -o wide

# 특정 Service의 EndpointSlice 확인
kubectl get endpointslice -l kubernetes.io/service-name=<service-name>

# Service 빠르게 생성
kubectl expose deployment web --name=web-svc --port=80 --target-port=8080 --type=ClusterIP
```

## 3. Ingress

Ingress는 **HTTP/HTTPS 트래픽을 host와 path 규칙에 따라 클러스터 내부 Service로 라우팅**하는 API 리소스임.

- Ingress 리소스만 생성해서는 동작하지 않으며 반드시 **Ingress Controller**가 필요함
- Ingress Controller가 Ingress 리소스를 감시하고 실제 reverse proxy 또는 LoadBalancer 설정을 적용함
- 구현체에 따라 NGINX 기반 Controller, 클라우드 제공 Controller 등이 존재함
- spec.ingressClassName으로 어떤 Ingress Controller가 해당 Ingress를 처리할지 지정할 수 있음
- TLS 인증서는 일반적으로 Secret에 저장하고 spec.tls에서 참조함
- Ingress는 주로 L7의 HTTP/HTTPS 라우팅을 담당하며 임의의 TCP/UDP 포트를 노출하는 범용 리소스는 아님

> Kubernetes 프로젝트는 현재 **Gateway API 사용을 권장**하고 있으며 Ingress API는 frozen 상태임. 다만 제거된 것은 아니며 CKA 범위에도 Ingress Controller와 Ingress 리소스가 포함됨.

### 최소 Ingress 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

트래픽 흐름은 보통 다음과 같음.

```text
Client
  ↓
Ingress Controller
  ↓
Ingress의 host/path 규칙
  ↓
Service
  ↓
Pod
```

## 4. Gateway API

Gateway API는 Ingress의 후속 API로, 인프라와 애플리케이션의 트래픽 설정을 역할별 리소스로 분리하고 다양한 프로토콜 및 고급 라우팅을 표준화함.

- Kubernetes 기본 내장 API가 아니라 **Gateway API CRD와 이를 구현하는 Gateway Controller가 필요함**
- 핵심 구조는 다음과 같음

```text
GatewayClass
    ↓
Gateway
    ↓
HTTPRoute / GRPCRoute
    ↓
Service
    ↓
Pod
```

- **GatewayClass**: 어떤 Gateway Controller 구현체를 사용할지 정의
- **Gateway**: 실제 트래픽을 수신할 listener와 진입점을 정의
- **HTTPRoute**: host/path 등 HTTP 라우팅 규칙과 backend Service를 정의
- **Gateway Controller**: Gateway API 리소스를 실제 네트워크 인프라 설정으로 구현

> Gateway를 생성할 때 매번 Pod, Deployment, Service가 반드시 새로 만들어지는 것은 아님. 실제 구현 방식은 사용하는 Gateway Controller에 따라 달라짐.

### Gateway + HTTPRoute 예시

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: example
  listeners:
  - name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app
    backendRefs:
    - name: web-svc
      port: 80
```

CKA 문제에서는 GatewayClass나 Controller가 이미 제공되는 경우가 있으므로 **문제에서 주어진 GatewayClass 이름과 listener 조건을 먼저 확인**하는 것이 중요함.

## 5. NetworkPolicy

NetworkPolicy는 Pod의 **Ingress와 Egress 트래픽을 L3/L4 수준에서 제어**하는 리소스임.

- NetworkPolicy가 하나도 적용되지 않은 Pod는 기본적으로 Ingress와 Egress가 모두 허용됨
- 특정 방향의 NetworkPolicy가 Pod를 선택하면 해당 방향은 격리되고, 정책에서 허용한 트래픽만 통과함
- 여러 NetworkPolicy가 적용되면 허용 규칙은 합쳐져서 적용됨
- 실제 차단 기능은 NetworkPolicy를 지원하는 CNI가 구현해야 함
- 주요 selector
  - podSelector: 대상 Pod 또는 상대 Pod 선택
  - namespaceSelector: Namespace label로 상대 Namespace 선택
  - ipBlock: CIDR 기반 IP 범위 선택
- `ingress.from`, `egress.to`, `ports`를 조합해 허용 범위를 지정

### Namespace 전체 Default Deny

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 확인 명령어

```bash
kubectl get networkpolicy
kubectl describe networkpolicy <name>
kubectl get pods --show-labels
kubectl get ns --show-labels
```

## 6. CoreDNS

CoreDNS는 클러스터 내부 DNS를 제공하며 Service와 Pod 이름을 해석하는 핵심 컴포넌트임.
Service 접근 장애가 발생하면 Service 자체뿐 아니라 DNS도 함께 확인해야 함.

```bash
# CoreDNS 상태
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system get svc kube-dns

# Pod 내부 DNS 설정
kubectl exec <pod> -- cat /etc/resolv.conf

# DNS 확인
kubectl exec <pod> -- nslookup <service-name>
kubectl exec <pod> -- nslookup <service-name>.<namespace>
```

## 7. Service & Networking 트러블슈팅 순서

1. **Pod가 정상인지 확인**
  - kubectl get pod -o wide
  - readiness 실패 여부 확인
2. **Service selector와 Pod label 확인**
  - kubectl describe svc <name>
  - kubectl get pods --show-labels
3. **EndpointSlice에 Pod가 등록됐는지 확인**
  - kubectl get endpointslice -l `kubernetes.io/service-name=<name>`
4. **port와 targetPort 확인**
  - Service의 port, targetPort
  - 컨테이너가 실제로 listen하는 포트
5. **클러스터 내부에서 직접 호출**
  - curl <service-name>:<port>
6. **DNS 문제 확인**
  - nslookup <service-name>
  - CoreDNS Pod 상태 및 로그 확인
7. **NetworkPolicy 확인**
  - 정책에 의해 Ingress/Egress가 차단되는지 확인
8. **Ingress/Gateway 확인**
  - Controller 존재 여부
  - IngressClass 또는 GatewayClass
  - Route가 올바른 Service와 Port를 참조하는지 확인

## CKA에서 우선 숙지할 범위

- Pod 간 통신 구조
- Service의 ClusterIP, NodePort, LoadBalancer
- selector와 EndpointSlice
- port, targetPort, nodePort
- CoreDNS 기반 Service Discovery
- NetworkPolicy 작성 및 문제 해결
- Ingress Controller와 Ingress 리소스
- GatewayClass, Gateway, HTTPRoute 관계
- Service/Networking 장애 진단 순서
