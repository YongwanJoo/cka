# 문제 풀이 - Ingress 생성하기

> [Notion 원본](https://app.notion.com/p/3e847fbe9c2a8023b51cd159af03e4da)

<details>
<summary>사전 환경 구축</summary>

```bash
cat << EOF >> /etc/hosts
192.168.56.40 [example.org](http://example.org/)
EOF
kubectl create ns echo-sound
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/ingress/deployment.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/ingress/service.yaml
```

</details>
<details>
<summary>리소스 정리</summary>

```bash
kubectl delete ns echo-sound
sudo sed -i '/^192\\.168\\.56\\.40 example\\.org$/d' /etc/hosts
```

</details>
## 문제 요구사항
1. echo-sound라는 namespace에 새로운 echo라는 이름의 Ingress 리소스 생성
2. Service `echoserver-service`를 `http://example.org/echo`로 노출하고 Service port `8080` 사용
3. curl로 200이 나오면 통과
## 풀이
1. IngressClass 확인
```bash
kubectl get ingressclass
```
1. Ingress 생성
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: echo-sound
spec:
  ingressClassName: nginx
  rules:
  - host: example.org
    http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echoserver-service
            port:
              number: 8080
EOF
```
1. 생성 확인
```bash
kubectl get ingress -n echo-sound
```
1. 요청 확인
```bash
curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```
정상 결과: `200`
### 핵심 구조
- `host`: `example.org`
- `path`: `/echo`
- `pathType`: `Prefix`
- Backend Service: `echoserver-service`
- Service port: `8080`
- `ingressClassName`은 `kubectl get ingressclass`로 확인
