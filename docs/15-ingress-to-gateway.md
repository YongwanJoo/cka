# 문제 풀이 - Ingress에서 Gateway로 전환하기

> [Notion 원본](https://app.notion.com/p/3e847fbe9c2a80edb034dcd8bc2b517e)

<details>
<summary>사전 환경 구축</summary>

```bash
cat << EOF >> /etc/hosts
192.168.56.40 ingress.web.k8s.local
192.168.56.40 gateway.web.k8s.local
EOF
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/gateway/secret.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/gateway/service.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/gateway/ingress.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/gateway/deployment.yaml
```

</details>
<details>
<summary>리소스 정리</summary>

```bash
kubectl delete deploy web
kubectl delete svc web
kubectl delete secret web-cert
kubectl delete httproutes web-route
kubectl delete gateway web-gateway
sudo sed -i '/^192\\.168\\.56\\.40 ingress\\.web\\.k8s\\.local$/d' /etc/hosts
sudo sed -i '/^192\\.168\\.56\\.40 gateway\\.web\\.k8s\\.local$/d' /etc/hosts
```

</details>
## 문제 요구 사항
1. 기존 `Ingress web`의 TLS 설정과 라우팅 규칙을 확인한다.
2. `GatewayClass nginx`를 사용하는 `Gateway web-gateway`를 생성한다.
3. Gateway의 hostname은 `gateway.web.k8s.local`로 설정하고 기존 Ingress가 사용하던 TLS Secret을 유지한다.
4. `HTTPRoute web-route`를 생성하고 기존 Ingress의 path, Service, Service port를 그대로 옮긴다.
5. `https://gateway.web.k8s.local` 요청이 성공하는지 확인한다.
6. Gateway API 전환이 정상적으로 완료된 뒤 기존 `Ingress web`을 삭제한다.
## 기존 Ingress에서 확인할 값
```bash
kubectl get ingress web -o yaml
```
이 문제는 Gateway와 HTTPRoute를 새로 설계하는 문제가 아니라 **기존 Ingress의 설정을 Gateway API 리소스로 분리해서 옮기는 문제**이다.
<table fit-page-width="true" header-row="true">
<tr>
<td>Ingress 설정</td>
<td>확인된 값</td>
<td>Gateway API에서 사용되는 위치</td>
</tr>
<tr>
<td>ingressClassName</td>
<td>nginx</td>
<td>Gateway의 gatewayClassName</td>
</tr>
<tr>
<td>tls.secretName</td>
<td>web-cert</td>
<td>Gateway listener의 certificateRefs</td>
</tr>
<tr>
<td>path</td>
<td>/</td>
<td>HTTPRoute matches.path.value</td>
</tr>
<tr>
<td>pathType</td>
<td>Prefix</td>
<td>HTTPRoute PathPrefix</td>
</tr>
<tr>
<td>backend Service</td>
<td>web</td>
<td>HTTPRoute `backendRefs.name`</td>
</tr>
<tr>
<td>Service port</td>
<td>80</td>
<td>HTTPRoute backendRefs.port</td>
</tr>
</table>
> 기존 Ingress의 hostname은 `ingress.web.k8s.local`이지만, 문제에서 새 Gateway와 HTTPRoute의 hostname을 `gateway.web.k8s.local`로 지정했으므로 hostname만 문제의 요구 사항에 맞게 변경한다.
## 풀이
### 1. 기존 Ingress 확인
```bash
kubectl get ingress web -o yaml
```
확인해야 할 핵심 값은 다음과 같다.
```text
TLS Secret : web-cert
Path       : /
Path Type  : Prefix
Service    : web
Port       : 80
```
### 2. Gateway 생성
```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: gateway.web.k8s.local
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: web-cert
    allowedRoutes:
      namespaces:
        from: Same
EOF
```
### 3. HTTPRoute 생성
```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - gateway.web.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web
      port: 80
EOF
```
### 4. Gateway와 HTTPRoute 상태 확인
```bash
kubectl get gateway web-gateway
kubectl get httproute web-route

kubectl describe gateway web-gateway
kubectl describe httproute web-route
```
정상 상태에서 확인할 값은 다음과 같다.
```text
Gateway
- Accepted: True
- Programmed: True
- Attached Routes: 1
- ResolvedRefs: True

HTTPRoute
- Accepted: True
- ResolvedRefs: True
```
`Conflicted=False`와 `Reason=NoConflicts`가 함께 표시되면 충돌이 없다는 의미이다.
### 5. HTTP 요청 테스트
먼저 hostname이 실제 Gateway 주소를 가리키는지 확인한다.
```bash
kubectl get gateway web-gateway -o wide
getent hosts gateway.web.k8s.local
```
이 실습 환경에서는 `/etc/hosts`의 `gateway.web.k8s.local`이 기존 NGINX 진입점인 `192.168.56.40`을 가리켜 404가 발생할 수 있다.
Gateway 자체가 정상인지 DNS 또는 hosts 설정과 분리해서 확인하려면 다음처럼 테스트한다.
```bash
GW_IP=$(kubectl get gateway web-gateway -o jsonpath='{.status.addresses[0].value}')

curl -k --resolve gateway.web.k8s.local:443:$GW_IP \
  https://gateway.web.k8s.local
```
정상 결과:
```text
hello
```
일반 `curl` 명령으로도 확인하려면 이 실습 환경에서는 `gateway.web.k8s.local`이 Gateway 주소를 가리키도록 `/etc/hosts`를 수정한다.
```bash
GW_IP=$(kubectl get gateway web-gateway -o jsonpath='{.status.addresses[0].value}')

sudo sed -i '/gateway\.web\.k8s\.local/d' /etc/hosts
echo "$GW_IP gateway.web.k8s.local" | sudo tee -a /etc/hosts

curl -k https://gateway.web.k8s.local
```
### 6. 기존 Ingress 삭제
Gateway API 경로로 요청이 정상 처리되는 것을 확인한 뒤 기존 Ingress를 삭제한다.
```bash
kubectl delete ingress web
```
마지막으로 다시 요청해 Gateway API만으로 서비스가 유지되는지 확인한다.
```bash
curl -k https://gateway.web.k8s.local
```
## 핵심 개념
### Gateway와 HTTPRoute의 역할 분리
Ingress에서는 외부 진입점, TLS, hostname, path, backend 설정이 하나의 리소스 안에 들어간다.
Gateway API에서는 역할이 분리된다.
```text
Client
  |
  | HTTPS :443
  v
Gateway
  - Listener
  - Hostname
  - TLS 인증서
  |
  | HTTP 요청
  v
HTTPRoute
  - Hostname
  - Path 매칭
  - Backend 선택
  |
  v
Service web:80
  |
  v
Pod
```
- **Gateway**: 트래픽을 어디에서, 어떤 프로토콜과 포트로 받을지 정의한다.
- **HTTPRoute**: Gateway로 들어온 HTTP 요청을 어떤 backend로 전달할지 정의한다.
- `parentRefs`: HTTPRoute가 사용할 Gateway를 지정한다.
- `backendRefs`: 요청을 전달할 Service를 지정한다.
### 왜 Gateway의 port는 443인데 protocol은 HTTPS인가?
`port`와 `protocol`은 서로 다른 설정이다.
```yaml
protocol: HTTPS
port: 443
```
- `port: 443`: Gateway가 요청을 받을 네트워크 포트
- `protocol: HTTPS`: 해당 포트의 트래픽을 HTTPS로 처리한다는 의미
HTTPS는 내부적으로 TCP 위에서 동작한다.
```text
HTTP
  ↓
TLS
  ↓
TCP
  ↓
IP
```
따라서 HTTPS 요청은 일반적으로 TCP 443을 사용하지만 Gateway API에서는 단순 TCP 전달이 아니라 HTTPS 요청을 해석하고 TLS를 종료해야 하므로 `protocol: HTTPS`를 사용한다.
### TLS Termination
```yaml
tls:
  mode: Terminate
  certificateRefs:
  - kind: Secret
    name: web-cert
```
`Terminate`는 클라이언트와의 TLS 연결을 Gateway에서 종료한다는 의미이다.
즉 다음 흐름이 된다.
```text
Client
  |
  | HTTPS + TLS
  v
Gateway
  | web-cert로 TLS 종료
  v
HTTPRoute
  |
  v
Service web:80
```
Service가 80 포트를 사용하더라도 외부 클라이언트는 HTTPS 443으로 접근할 수 있다.
### PathPrefix
Ingress의 다음 설정:
```yaml
path: /
pathType: Prefix
```
은 HTTPRoute에서 다음처럼 대응된다.
```yaml
matches:
- path:
    type: PathPrefix
    value: /
```
`PathPrefix: /`는 `/`, `/login`, `/api/users`처럼 `/`로 시작하는 경로를 모두 매칭한다.
## 404가 발생했을 때 확인 순서
Gateway와 HTTPRoute 상태가 모두 정상인데 NGINX의 `404 Not Found`가 나온다면 무조건 backend 문제라고 판단하면 안 된다.
이 실습에서는 다음 상태였다.
```text
Gateway ADDRESS
10.109.79.231

gateway.web.k8s.local
192.168.56.40
```
즉 hostname이 실제 Gateway가 아닌 다른 NGINX 진입점을 가리키고 있었기 때문에 요청이 HTTPRoute까지 도달하지 않았다.
다음 순서로 확인하면 원인을 빠르게 분리할 수 있다.
1. Gateway가 정상인지 확인
```bash
kubectl get gateway
kubectl describe gateway web-gateway
```
1. HTTPRoute가 Gateway에 정상 연결되었는지 확인
```bash
kubectl describe httproute web-route
```
1. Gateway 주소와 hostname 해석 결과 비교
```bash
kubectl get gateway web-gateway -o wide
getent hosts gateway.web.k8s.local
```
1. DNS 또는 `/etc/hosts`를 우회해 Gateway로 직접 테스트
```bash
curl -k --resolve gateway.web.k8s.local:443:<GATEWAY_IP> \
  https://gateway.web.k8s.local
```
`--resolve`는 요청의 hostname과 TLS SNI는 유지하면서 해당 hostname이 특정 IP로 해석되도록 강제한다. 따라서 **Gateway/Route 문제인지 DNS 또는 hosts 문제인지 분리해서 확인할 때 유용하다.**
## 시험에서 기억할 흐름
```text
기존 Ingress 확인
        ↓
TLS Secret / Path / Service / Port 추출
        ↓
Gateway 생성
- GatewayClass
- HTTPS 443 Listener
- hostname
- TLS Secret
        ↓
HTTPRoute 생성
- parentRefs
- hostname
- PathPrefix
- backendRefs
        ↓
Accepted / Programmed / ResolvedRefs 확인
        ↓
curl 테스트
        ↓
성공 후 기존 Ingress 삭제
```
