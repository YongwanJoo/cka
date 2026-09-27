# 문제 풀이 - NetworkPolicy 생성하기

> [Notion 원본](https://app.notion.com/p/3e847fbe9c2a80d09df7ee8f52e8988b)

<details>
<summary>사전 환경 구축</summary>

```bash
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/networkpolicy/namespace-backend.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/networkpolicy/namespace-frontend.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/networkpolicy/deployment-backend.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/networkpolicy/deployment-frontend.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/networkpolicy/networkpolicy.yaml
mkdir -p /home/cka0001/netpol
curl -o ~/netpol/netpol1.yaml https://raw.githubusercontent.com/kubetm/exam-c/main/networkpolicy/netpol1.yaml
curl -o ~/netpol/netpol2.yaml https://raw.githubusercontent.com/kubetm/exam-c/main/networkpolicy/netpol2.yaml
curl -o ~/netpol/netpol3.yaml https://raw.githubusercontent.com/kubetm/exam-c/main/networkpolicy/netpol3.yaml
```

</details>
<details>
<summary>리소스 정리</summary>

```bash
kubectl delete ns frontend
kubectl delete ns backend
rm -rf ~/netpol
```

</details>

## 문제 요구사항

1. `frontend`, `backend` Deployment는 서로 다른 namespace에 존재
2. frontend에서 backend로 통신할 수 있도록 NetworkPolicy 적용
3. `~/netpol`의 후보 3개 중 가장 제한적인 정책 선택
4. 기존 `deny-all` NetworkPolicy는 삭제하거나 수정하지 않음

## 풀이

### 1. Deployment의 namespace, Pod label, port 확인

```bash
kubectl describe deploy -n backend
kubectl describe deploy -n frontend
```

확인할 값:

- backend Pod label: `app=backend`
- backend container port: `8080`
- frontend Pod label: `app=frontend`
### 2. NetworkPolicy 후보 확인

```bash
cat ~/netpol/netpol1.yaml
cat ~/netpol/netpol2.yaml
cat ~/netpol/netpol3.yaml
```

후보 판단:

- `netpol1`: backend namespace의 모든 Pod를 대상으로 하고 frontend namespace의 모든 Pod를 허용하므로 범위가 넓음
- `netpol2`: `app=backend` Pod만 대상으로 하며, `frontend` namespace의 `app=frontend` Pod만 허용
- `netpol3`: 대상 Pod가 `app=database`이므로 실제 backend Pod를 선택하지 않음

따라서 가장 제한적이면서 실제 통신을 허용하는 정책은 **netpol2**.

### 3. namespace label 확인

```bash
kubectl get ns frontend --show-labels
```

정상 예시:

```text
kubernetes.io/metadata.name=frontend,name=frontend
```

`netpol2`의 `namespaceSelector`가 `name=frontend`를 사용하므로 해당 label이 있어야 함.

### 4. NetworkPolicy 적용

```bash
kubectl apply -f ~/netpol/netpol2.yaml
```

### 5. 적용 상태 확인

```bash
kubectl get netpol -n backend
kubectl describe netpol netpol2 -n backend
```

정상 상태:

- 기존 `deny-all` 유지
- `netpol2`의 PodSelector는 `app=backend`
- Source는 `name=frontend` namespace이면서 `app=frontend`인 Pod
### 6. 실제 통신 확인

```bash
kubectl get pod -n backend -o wide
kubectl exec -n frontend deploy/frontend -- curl <BACKEND_POD_IP>:8080
```

정상 결과:

```text
hello
```

### 핵심 구조

```yaml
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: frontend
        podSelector:
          matchLabels:
            app: frontend
```

- `podSelector: app=backend`: 정책을 적용할 목적지 Pod 지정
- 같은 `from` 항목 안의 `namespaceSelector`와 `podSelector`는 **AND 조건**
- 따라서 frontend namespace에 있으면서 `app=frontend` label을 가진 Pod만 허용
- `policyTypes: Ingress`이므로 backend Pod로 들어오는 트래픽만 제어하며 Egress에는 영향 없음
- `ports`가 없으므로 선택된 Source에서 backend Pod의 모든 포트로 접근 가능
- 이 문제는 주어진 YAML 중 하나를 선택하는 문제이므로 후보 중 `netpol2`가 least permissive 조건에 가장 적합

### 시험장 풀이 순서

```bash
kubectl describe deploy -n backend
kubectl describe deploy -n frontend
cat ~/netpol/netpol1.yaml
cat ~/netpol/netpol2.yaml
cat ~/netpol/netpol3.yaml
kubectl get ns frontend --show-labels
kubectl apply -f ~/netpol/netpol2.yaml
kubectl get netpol -n backend
kubectl exec -n frontend deploy/frontend -- curl <BACKEND_POD_IP>:8080
```
