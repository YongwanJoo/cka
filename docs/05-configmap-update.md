# 문제 풀이 - ConfigMap 수정

> 원본 Notion 페이지에 포함된 이미지는 임시 서명 URL이라 GitHub에 안정적으로 보존하지 않았습니다.

요약: TLSv1.2도 가능하게 nginx-config를 수정하기

<details>
<summary>문제 환경 구축</summary>

```bash
kubectl create ns nginx-static
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/configmap/configmap.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/configmap/secret.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/configmap/service.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/configmap/deployment.yaml
echo "192.168.56.40 web.k8s.local" | sudo tee -a /etc/hosts
```

</details>

<details>
<summary>리소스 정리</summary>

```bash
kubectl delete ns nginx-static
sudo sed -i '/^192\.168\.56\.40 web\.k8s\.local$/d' /etc/hosts
```

</details>

## 풀이

이 문제는 ConfigMap의 Nginx 설정을 수정한 뒤 **Nginx 프로세스가 새 설정을 다시 읽도록 Pod를 재생성**하면 된다. Deployment가 관리하는 Pod이므로 `rollout restart`를 사용하면 Pod 이름을 직접 찾고 삭제할 필요가 없다.

```bash
kubectl edit -n nginx-static configmap nginx-config

# 기존 TLSv1.3 설정에 TLSv1.2 허용 범위를 추가한 후
kubectl rollout restart deployment/nginx-static -n nginx-static
kubectl rollout status deployment/nginx-static -n nginx-static

# 결과 확인
curl -k --tls-max 1.2 https://web.k8s.local:30007

# Pod 에러 발생 시
kubectl get pod -n nginx-static
kubectl describe pod -n nginx-static <pod-name>

# ImagePullBackOff처럼 컨테이너가 시작 전이면 logs보다 describe의 Events를 먼저 확인
```

## 용어 설명

- **rollout restart**: Deployment 등의 Pod template에 재시작 트리거를 반영해 관리되는 Pod를 순차적으로 다시 생성하는 명령
- **ImagePullBackOff**: kubelet이 컨테이너 이미지를 가져오는 데 반복적으로 실패해 재시도 간격을 늘리고 있는 Pod 상태
- **Events**: 스케줄링, 이미지 pull, 볼륨 mount 등 리소스에서 발생한 주요 상태 변화를 기록한 정보
