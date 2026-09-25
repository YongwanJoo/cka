# Storage Class

> 원본 Notion 페이지에 포함된 이미지는 임시 서명 URL이라 GitHub에 안정적으로 보존하지 않았습니다.

## Default StorageClass

- StorageClassName 생략 시 default가 사용됨
- `""` -> StorageClass 지정 안할 시

## Provisioner

- 특정 볼륨 솔루션의 이름
- StorageClass 만들 때 Provisioner에 넣고 PVC 연결 가능
- PV와 Volume 동적 생성 가능

## volumeBindingMode

- `Immediate`: PVC 생성 시 PV와 Volume이 만들어짐
- `WaitForFirstConsumer`: Pod 생성 시 Volume이 만들어짐

## 용어 설명

- **StorageClass**: 동적 프로비저닝에 사용할 provisioner와 볼륨 생성 정책을 정의하는 Kubernetes 리소스
- **Default StorageClass**: PVC에서 `storageClassName`을 생략했을 때 기본적으로 선택되는 StorageClass
