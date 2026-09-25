# 단축어

| 리소스 | Short name |
| --- | --- |
| Pod | `po` |
| Deployment | `deploy` |
| ReplicaSet | `rs` |
| StatefulSet | `sts` |
| DaemonSet | `ds` |
| Service | `svc` |
| Ingress | `ing` |
| ConfigMap | `cm` |
| Namespace | `ns` |
| Node | `no` |
| PersistentVolume | `pv` |
| PersistentVolumeClaim | `pvc` |
| ServiceAccount | `sa` |
| HorizontalPodAutoscaler | `hpa` |

## 용어 설명

- **Pod**: 하나 이상의 컨테이너를 실행하는 Kubernetes의 최소 배포 단위
- **Deployment**: 주로 stateless 애플리케이션의 Pod 배포, 복제, 롤링 업데이트를 관리하는 컨트롤러
- **ReplicaSet**: 지정한 수의 동일한 Pod replica가 유지되도록 관리하는 컨트롤러
- **StatefulSet**: 각 Pod에 안정적인 이름, 네트워크 식별자, 스토리지 연결을 유지하며 stateful 애플리케이션을 관리하는 컨트롤러
- **DaemonSet**: 조건에 맞는 각 Node마다 Pod가 하나씩 실행되도록 관리하는 컨트롤러
- **Service**: Pod 집합에 안정적인 네트워크 접근 지점을 제공하는 리소스
- **Ingress**: HTTP/HTTPS 요청을 클러스터 내부 Service로 라우팅하기 위한 리소스
- **ConfigMap**: 비민감 설정 데이터를 저장하는 리소스
- **Namespace**: 하나의 클러스터 안에서 namespaced 리소스를 논리적으로 구분하는 범위
- **PersistentVolume (PV)**: 실제 영구 스토리지 자원을 Kubernetes 리소스로 표현한 것
- **PersistentVolumeClaim (PVC)**: 워크로드가 필요한 영구 스토리지를 요청하는 리소스
- **ServiceAccount**: Pod 등 Kubernetes 워크로드에 API 접근용 신원을 제공하는 리소스
- **HorizontalPodAutoscaler (HPA)**: CPU, 메모리 등의 메트릭을 기준으로 대상 워크로드의 replica 수를 자동 조정하는 리소스
