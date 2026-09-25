# Storage

## 스토리지 유형

1. **Block Storage**
   - 데이터를 block 단위로 제공하는 스토리지. Kubernetes에서는 보통 CSI를 통해 Volume/PV로 사용
   - DB처럼 낮은 지연시간과 random I/O가 중요한 워크로드에서 많이 사용하지만 **StatefulSet 전용은 아님**
   - `ReadWriteOnce`가 흔하지만 실제 동시 연결 가능 여부는 스토리지 backend와 CSI driver의 기능에 따라 달라짐
   - 네트워크 block storage도 있으므로 물리적으로 서버에 직접 장착해야 하는 것은 아님
2. **File Storage**
   - NFS, CephFS, EFS처럼 파일 시스템을 네트워크로 공유하는 형태
   - 여러 Node/Pod에서 같은 파일을 공유해야 하는 워크로드에 적합하며 backend가 지원하면 `ReadWriteMany` 사용 가능
   - Deployment와 StatefulSet 모두에서 사용할 수 있으며 특정 workload controller에 종속되지 않음
3. **Object Storage**
   - S3와 같이 object를 API/SDK를 통해 key 기반으로 저장하고 조회
   - 일반적인 Kubernetes 워크로드에서는 애플리케이션이 API로 직접 접근하므로 PV/PVC 없이 사용하는 경우가 많음
   - Block/File Volume과 사용 방식이 다르며, 필요한 경우 별도 CSI driver나 FUSE 계층으로 mount하는 구현도 존재
   - 대규모 비정형 데이터 저장과 수평 확장에 적합

## Kubernetes에서 스토리지를 사용하는 대표 방식

1. `emptyDir`: Pod 수명 동안 사용하는 임시 저장소. Pod 삭제 시 데이터도 삭제
2. `hostPath`: 특정 Node의 경로를 직접 mount. Node 종속성이 강해 일반적인 영구 스토리지 용도로는 주의
3. **Static Provisioning**: 관리자가 PV를 미리 생성하고 PVC가 조건에 맞는 PV와 Binding
4. **Dynamic Provisioning**: PVC가 StorageClass를 요청하면 provisioner/CSI driver가 PV를 자동 생성
5. 외부 NFS, Ceph, 클라우드 Block/File Storage 등은 보통 CSI driver를 통해 Kubernetes와 연동

## 워크로드와 스토리지

- **Deployment = File Storage**, **StatefulSet = Block Storage**처럼 1:1로 연결해서 외우면 안 됨
- Deployment는 stateless workload에 주로 사용하지만 PVC를 mount할 수 있음
- StatefulSet은 Pod별 안정적인 identity와 persistent storage가 필요한 workload에 적합하며 `volumeClaimTemplates`를 자주 사용

## 용어 설명

- **CSI (Container Storage Interface)**: Kubernetes가 외부 스토리지 시스템과 연동해 볼륨을 생성, 연결, 마운트하도록 하는 표준 인터페이스
- **volumeClaimTemplates**: StatefulSet에서 각 Pod에 대응하는 PVC를 자동 생성하기 위한 템플릿
- **stateless**: 개별 Pod 자체에 지속적으로 유지해야 할 고유 상태를 두지 않는 애플리케이션 구조
- **stateful**: Pod별 데이터나 식별자처럼 재시작 또는 재스케줄링 이후에도 유지해야 할 상태가 있는 애플리케이션 구조
- **FUSE (Filesystem in Userspace)**: 사용자 공간 프로그램이 파일 시스템 인터페이스를 구현할 수 있도록 하는 Linux 메커니즘
