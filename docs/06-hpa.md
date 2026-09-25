# 문제풀이 - HPA

> 원본 Notion 페이지에 포함된 이미지는 임시 서명 URL이라 GitHub에 안정적으로 보존하지 않았습니다.

## 요구 조건

1. HPA 설정을 할 때 타겟 CPU 사용량을 pod 당 50%로 설정
2. Autoscaling 시 pod는 1개에서 4개 사이로 설정
3. Downscale 시 replica 수가 급격히 흔들리는 것을 완화하기 위해 `scaleDown.stabilizationWindowSeconds`를 30초로 설정

<details>
<summary>사전 설치 명령어</summary>

```bash
kubectl create ns autoscale
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/hpa/deployment.yaml
```

</details>

<details>
<summary>삭제 명령어</summary>

```bash
kubectl delete ns autoscale
```

</details>

## 풀이

```bash
# autoscale namespace에 hpa 설정 추가
kubectl autoscale -n autoscale deployment apache-server --cpu=50% --min=1 --max=4

# 적용 확인
k get -n autoscale hpa -o yaml

# Downscale stabilization window 설정
k edit -n autoscale hpa apache-server

# spec 아래에 필요한 값만 추가
# behavior:
#   scaleDown:
#     stabilizationWindowSeconds: 30

# 확인
k get hpa apache-server -n autoscale -o yaml

# CPU Utilization 기반 HPA가 실제로 동작하려면 Metrics Server가 정상이어야 하고,
# 대상 Pod의 container에 CPU requests가 설정되어 있어야 CPU 사용률 계산이 가능
```

## 용어 설명

- **HPA (HorizontalPodAutoscaler)**: CPU, 메모리 또는 사용자 정의 메트릭을 기준으로 Deployment, StatefulSet 등 scale 가능한 대상의 replica 수를 자동 조정하는 리소스
- **Metrics Server**: kubelet에서 CPU, 메모리 등의 리소스 사용량을 수집해 Kubernetes Metrics API로 제공하는 컴포넌트
- **CPU request**: 컨테이너가 필요로 한다고 선언한 CPU 양. CPU utilization 기반 HPA에서는 실제 사용량을 request 대비 비율로 계산
- **stabilization window**: 최근 스케일링 권고값을 일정 시간 고려해 replica 수가 짧은 시간에 반복적으로 증가/감소하는 현상을 완화하는 설정
