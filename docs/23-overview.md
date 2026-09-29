# 전체 개요

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a80149802c204f3713351)

## 한눈에 보는 흐름
Kubernetes는 사용자가 선언한 **원하는 상태**를 실제 클러스터 상태와 계속 맞춘다. Pod는 컨테이너 자체가 아니라 컨테이너 실행 방법을 기술한 **API 객체**다.

1. 사용자가 `kubectl` 등으로 Deployment/Pod를 제출하면 `kube-apiserver`가 요청을 처리하고 상태를 `etcd`에 저장한다.
2. Deployment를 사용했다면 관련 Controller가 ReplicaSet과 Pod를 만들고 원하는 복제 수를 유지한다. 아직 Node가 정해지지 않은 Pod는 `kube-scheduler`가 배치한다.
3. 선택된 Node의 `kubelet`이 Pod 명세를 보고 **CRI**를 통해 container runtime에 Pod sandbox와 컨테이너 실행을 요청한다.
4. runtime은 **CNI 플러그인**을 이용해 Pod 네트워크를 구성한다. 볼륨이 필요하면 Kubernetes의 볼륨 기능과 **CSI 드라이버**가 스토리지 연결·마운트에 관여한다.
5. 이후 kubelet과 Controller가 상태를 계속 관찰하고 장애가 나면 재시작 또는 대체 Pod 생성 등으로 원하는 상태에 가깝게 만든다.

> 암기 순서: API Server/etcd → Controller → Scheduler → kubelet → CRI/runtime → CNI(네트워크)·CSI(스토리지). `kubectl`은 API Server에 요청하는 **클라이언트**이며, Pod를 직접 실행하지 않는다.

## CRI·CNI·CSI: 무엇을 연결하는가

- **CRI (Container Runtime Interface):** kubelet과 container runtime 사이의 표준 API. 실제로 설치·설정하는 것은 CRI 자체가 아니라 **CRI를 지원하는 runtime**(예: containerd, CRI-O)이다. Docker Engine을 사용할 때는 `cri-dockerd` 어댑터가 필요하다. Podman을 이 목록의 기본 CRI runtime으로 묶어 외우지 않는다.
- **CNI (Container Network Interface):** Pod의 네트워크 인터페이스와 IP 연결을 구성하기 위한 플러그인 규격. Calico, Cilium, Flannel 등은 구체적인 네트워크 구현이며, 지원 기능은 구현별로 다르다. 현재 Kubernetes에서는 일반적으로 CRI runtime이 CNI 플러그인을 로드·호출한다.
- **CSI (Container Storage Interface):** 외부 스토리지를 Kubernetes 볼륨으로 연결하기 위한 표준 인터페이스. CSI driver가 설치되면 StorageClass, PV/PVC를 통해 볼륨을 생성하고 Pod에 연결할 수 있다.
**구분:** CRI는 *컨테이너 실행*, CNI는 *Pod 네트워크*, CSI는 *볼륨·스토리지*를 담당한다.

## 네트워킹: Pod Network와 Service를 분리해서 보기

### Pod Network

- Pod는 클러스터 내부에서 Pod IP를 갖고, 서로 다른 Node의 Pod도 네트워크 구현이 허용하는 범위에서 통신한다. Pod IP는 재생성 시 바뀔 수 있다.
- CNI 기반 네트워크 플러그인이 Pod 간 연결을 구현한다. **Pod CIDR**은 Pod IP 할당에 사용하는 주소 범위다.

### Service Network

- Service는 변할 수 있는 Pod 집합 앞에 안정적인 접점을 제공한다. `ClusterIP` Service를 만들면 Control Plane이 **Service CIDR**에서 가상 IP를 할당한다.
- Service의 `selector`와 일치하는 Pod는 **EndpointSlice**에 백엔드로 반영된다. 실제 패킷 전달 규칙은 보통 각 Node의 `kube-proxy` 또는 이를 대체하는 네트워크 구현이 만든다. EndpointSlice 자체가 패킷을 전달하는 것은 아니다.
- **CoreDNS**는 Service 이름을 IP로 찾게 해준다. 일반적인 흐름은 `Service DNS 이름 → ClusterIP → Service 프록시 규칙 → EndpointSlice에 반영된 백엔드 Pod`로 이해하면 된다. DNS가 정상이어도 Service selector나 백엔드가 틀리면 접속은 실패할 수 있다.

### NetworkPolicy

- 어떤 Pod의 인바운드·아웃바운드 연결을 허용할지 선언하는 리소스다. 정책이 없다면 기본적으로 해당 방향의 통신은 허용된다.
- 정책을 적용한 Pod의 통신은 허용 규칙을 만족해야 한다. 출발 Pod의 egress와 도착 Pod의 ingress가 모두 격리되어 있다면 **양쪽 규칙이 모두 허용**해야 연결된다.
- 정책의 실제 적용에는 **NetworkPolicy를 지원하는 네트워크 플러그인**이 필요하다. egress를 전부 차단하면 DNS 요청도 막힐 수 있으므로 필요 시 DNS 예외를 추가한다.
확인 명령: `kubectl get pod -o wide`, `kubectl get svc`, `kubectl get endpointslice`, `kubectl get networkpolicy -A`.
더 자세한 내용: [Service & Networking](https://app.notion.com/p/3e847fbe9c2a80c5a5aec3c53284e9c6)

## Helm: 여러 Kubernetes 리소스를 하나의 설치 단위로 관리

- **Chart**는 애플리케이션 설치에 필요한 Kubernetes 매니페스트 템플릿 묶음이다. 보통 `Chart.yaml`(차트 정보), `values.yaml`(기본 설정값), `templates/`(매니페스트 템플릿)으로 구성된다.
- `helm install`은 Chart와 설정값을 렌더링해 Kubernetes 리소스를 설치하고 **Release**라는 설치 이력을 만든다. `helm upgrade`로 설정을 바꾸고 `helm rollback`으로 이전 Release로 돌아갈 수 있다.
- `values.yaml`이나 `--set`을 이용하면 차트의 설정값을 바꿀 수 있다. Helm의 장점은 매니페스트를 무조건 적게 쓰는 데 있지 않고, **여러 리소스의 설정을 재사용하고 설치·업데이트를 일관되게 관리**하는 데 있다.
- Artifact Hub는 공개 Chart를 찾는 한 경로다. 가져온 Chart도 `helm show values`와 `helm template`으로 설정과 렌더링 결과를 확인한 뒤 설치한다.
- Helm은 Kubernetes의 필수 구성 요소가 아니다. Argo CD를 Helm으로 설치하는 것은 **Helm 활용 예시**이며, CKA 공식 범위에는 Helm과 함께 **Kustomize**도 포함된다.
핵심 명령: `helm show values <chart>`, `helm template <release> <chart> -f values.yaml`, `helm install <release> <chart>`, `helm list -A`, `helm upgrade <release> <chart>`, `helm rollback <release> <revision>`.

## Kustomize: 기본 매니페스트를 환경별로 조합

- Kustomize는 `kustomization.yaml`에서 기존 YAML 파일을 `resources`로 모으고, 이름·레이블·이미지·패치 등을 적용해 최종 매니페스트를 만든다. 원본 매니페스트를 복사해 환경마다 따로 수정할 필요가 없다.
- `kubectl kustomize <디렉터리>`는 결과를 **출력만** 하고, `kubectl apply -k <디렉터리>`는 결과를 클러스터에 **적용**한다. 적용 전에는 `kubectl diff -k <디렉터리>`로 차이를 확인할 수 있다.
- Helm은 Chart의 템플릿과 values로 릴리스를 설치·관리한다. Kustomize는 기존 매니페스트를 조합·변형한다. CKA 공식 범위에는 둘 다 포함되므로 문제에서 지정한 도구와 결과 파일·적용 범위를 먼저 확인한다.
공식 문서: [Kubernetes Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)

## CRD와 Operator: Kubernetes API 확장

- **CRD (CustomResourceDefinition):** Kubernetes에 새로운 리소스 종류의 API group, kind, schema 등을 등록한다. 예를 들어 CRD가 등록되면 사용자는 그 종류의 **Custom Resource(CR)** 객체를 만들 수 있다.
- **CR:** 새로 정의된 종류의 실제 객체. CRD가 “종류의 정의”라면 CR은 “그 종류의 인스턴스”다.
- **Operator:** 보통 CR을 감시하는 Controller와 관련 CRD/RBAC 등을 함께 제공해, CR에 적힌 원하는 상태를 실제 리소스에 반영한다. **CRD만 설치하면 API에 객체를 저장·조회할 수 있지만, 그 객체를 보고 자동으로 운영 작업을 할 Controller가 생기는 것은 아니다.**
- `kubectl get crd`는 등록된 종류를 보는 명령이다. CKA 범위인 “Operator 설치·설정”까지 대비하려면 CRD 확인에 더해 Operator 배포 상태, CR 생성 결과, Controller 로그를 검증해야 한다.
확인 명령: `kubectl get crd`, `kubectl api-resources`, `kubectl describe crd <이름>`, `kubectl get deploy -A`, `kubectl get <custom-resource-kind> -A`.

## 이 페이지에서 기억할 연결

- Pod 실행 문제: **API 객체 → Scheduler → kubelet → CRI/runtime** 순서로 원인을 좁힌다.
- Pod 간 연결 문제: **Pod IP/CNI → Service selector·EndpointSlice → Service 프록시 → DNS·NetworkPolicy** 순서로 확인한다.
- 볼륨 문제: **StorageClass → PVC/PV → CSI driver → Pod mount**를 확인한다.
- Helm/Operator 문제: **Chart 또는 설치 매니페스트 → CRD/RBAC/Controller → CR의 상태**를 구분해서 확인한다.

## 공식 참고 문서

- [CKA 공식 시험 범위](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)
- [Kubernetes Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Kubernetes Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Helm Charts](https://helm.sh/docs/topics/charts/)
