# 파일스토리지

> 원본 Notion 페이지에 포함된 이미지는 임시 서명 URL이라 GitHub에 안정적으로 보존하지 않았습니다.

**핵심 흐름**: Pod -> PVC -> PV -> 실제 Storage

## PV / PVC

- **PV (PersistentVolume)**: 실제 스토리지 자원을 Kubernetes에서 표현한 리소스
  - Static Provisioning: 관리자가 PV를 미리 생성
  - Dynamic Provisioning: PVC 요청에 따라 StorageClass와 provisioner가 PV를 자동 생성
- **PVC (PersistentVolumeClaim)**: Pod가 필요한 스토리지를 요청하는 리소스
  - 용량, AccessMode, StorageClass 등의 조건을 지정
  - 조건에 맞는 PV와 Binding되며, Pod는 PVC를 참조하여 스토리지를 사용

## Static Provisioning

- 미리 생성된 PV와 PVC를 Binding하여 사용
- `volumeName`: 특정 PV를 pre-bind할 때 사용. 지정한 PV가 존재해야 하며 storageClass, accessModes, 요청 용량 등 기본 조건은 여전히 확인
- `selector`: label 조건에 맞는 PV를 선택. **비어 있지 않은 selector를 지정한 PVC는 Dynamic Provisioning 대상이 되지 않음**

## Dynamic Provisioning

- PVC에 `storageClassName`을 지정
- 해당 StorageClass의 설정에 따라 provisioner가 필요한 PV를 자동 생성하고 PVC와 Binding
- 흐름: PVC -> StorageClass -> PV 자동 생성 -> PVC/PV Binding

## AccessMode

- `ReadWriteOnce (RWO)`: 하나의 Node에서 read-write mount. 같은 Node의 여러 Pod가 사용할 수 있으므로 'Pod 하나만 사용'이라는 뜻은 아님
- `ReadOnlyMany (ROX)`: 여러 Node에서 read-only mount
- `ReadWriteMany (RWX)`: 여러 Node에서 read-write mount
- `ReadWriteOncePod (RWOP)`: 클러스터 전체에서 하나의 Pod만 read-write mount. CSI Volume에서 지원

## CKA 핵심 포인트

- Pod는 일반적으로 PV를 직접 참조하지 않고 PVC를 참조
- PVC와 PV는 용량, AccessMode, StorageClass 등의 조건이 맞아야 Binding됨
- StorageClass는 Dynamic Provisioning에서 사용할 provisioner와 스토리지 정책을 정의
- PVC와 PV Binding은 1:1 관계

## 용어 설명

- **provisioner**: PVC 요청을 바탕으로 실제 스토리지와 PV를 생성하는 주체. StorageClass의 `provisioner` 필드로 어떤 구현을 사용할지 지정
- **Binding**: 조건이 맞는 PVC와 PV를 서로 연결하는 과정
- **pre-bind**: PVC의 `volumeName` 등을 사용해 특정 PV와의 연결을 미리 지정하는 방식
- **CSI Volume**: CSI 드라이버를 통해 Kubernetes와 연동되는 외부 스토리지 볼륨
