# 문제 풀이 - StorageClass 생성

> 원본 Notion 페이지에 포함된 이미지는 임시 서명 URL이라 GitHub에 안정적으로 보존하지 않았습니다.

<details>
<summary>리소스 정리</summary>

```bash
kubectl delete sc low-latency
```

</details>

## 요구 사항

1. `low-latency`라는 이름의 StorageClass를 `rancher.io/local-path`를 provisioner로 하여 생성
2. VolumeBindingMode는 `WaitForFirstConsumer`로 지정
3. Cluster 내의 default StorageClass로 지정하기

## 풀이

```bash
cat <<EOF > sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: low-latency
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl apply -f sc.yaml

kubectl get storageclass
```

## 용어 설명

- **StorageClass**: 동적 프로비저닝에 사용할 provisioner와 볼륨 생성 정책을 정의하는 Kubernetes 리소스
- **provisioner**: PVC 요청을 바탕으로 실제 스토리지와 PV를 생성하는 구현체
- **WaitForFirstConsumer**: PVC 생성 즉시 볼륨을 만들지 않고, 해당 PVC를 사용하는 첫 Pod가 스케줄링될 때까지 바인딩/프로비저닝을 지연하는 `volumeBindingMode`
- **Default StorageClass**: PVC가 StorageClass를 명시하지 않았을 때 기본으로 사용되는 StorageClass
