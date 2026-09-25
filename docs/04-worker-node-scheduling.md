# 워커 노드, 스케줄링

> 원본 Notion 페이지에 포함된 이미지는 임시 서명 URL이라 GitHub에 안정적으로 보존하지 않았습니다.

## 쿠버네티스 클러스터 기본 구조

- 일반적으로 Control Plane과 Worker Node로 구성
- **최소 2개의 VM이 필수인 것은 아님**. 단일 노드 클러스터도 구성 가능
- Control Plane: API Server, Scheduler, Controller Manager, etcd 등 클러스터 제어 컴포넌트 실행
- Worker Node: kubelet, container runtime 등을 통해 실제 워크로드 Pod 실행
- Control Plane 노드도 taint를 제거하거나 스케줄링 조건을 만족시키면 일반 Pod 실행 가능

## kubeadm

- Kubernetes 클러스터를 bootstrap하기 위한 도구
- `kubeadm init`: Control Plane 초기화
- `kubeadm join`: Worker 또는 추가 Control Plane 노드를 기존 클러스터에 참여시킴

## kubectl

- Kubernetes API Server에 요청을 보내 API Object를 조회, 생성, 수정, 삭제하는 CLI
- `kubectl`이 etcd에 직접 저장하는 것이 아니라 **API Server가 클러스터 상태를 etcd에 저장**
- 리소스 간 관계는 물리적 연결이 아니라 label/selector, ownerReference, 이름 참조 등의 **논리적 관계**로 구성

## 주요 리소스

- Pod: 컨테이너를 실행하는 최소 배포 단위
- Deployment: ReplicaSet을 관리하며 stateless Pod의 배포와 업데이트를 관리
  - `Recreate`: 기존 Pod를 종료한 뒤 새 버전 Pod 생성
  - `RollingUpdate`: 기존 Pod와 새 Pod를 점진적으로 교체
- ReplicaSet: 지정한 수의 Pod replica가 유지되도록 관리
- HPA: 메트릭을 기준으로 Deployment/StatefulSet 등 scale 가능한 대상의 replica 수를 조정
- Service: 선택한 Pod에 안정적인 접근 지점을 제공. `port`와 `targetPort`를 다르게 지정 가능
- ConfigMap: 비민감 설정값, 환경 변수, 설정 파일 데이터 저장
- Secret: 비밀번호, 토큰, 인증서 등 민감한 데이터 저장. base64 인코딩 자체는 암호화가 아님
- PV/PVC: 영구 스토리지를 클러스터에 표현하고, 워크로드가 필요한 스토리지를 요청하여 사용

## Control Plane 컴포넌트

- kube-controller-manager: 여러 controller의 control loop를 실행하여 현재 상태가 사용자가 선언한 원하는 상태에 가까워지도록 조정. 일반적으로 API Server를 통해 리소스를 조회/수정함
- kube-apiserver: Kubernetes API의 중심 진입점. 요청을 인증/인가하고 admission을 거쳐 API Object를 처리하며 상태를 etcd에 저장
- etcd: Kubernetes 클러스터의 API 데이터와 상태를 저장하는 분산 key-value store
- kube-scheduler: 아직 Node가 정해지지 않은 Pod를 대상으로 resource request, taint/toleration, affinity, nodeSelector 등의 조건을 고려해 적절한 Node를 선택
  - 스케줄링된 Node의 kubelet이 PodSpec을 확인하고 CRI를 통해 container runtime에 컨테이너 실행을 요청

## Namespace

Namespace는 namespaced 리소스를 논리적으로 구분하는 범위입니다. 같은 클러스터 안에서 이름, 권한, 정책, quota 등을 구분하는 데 활용하며 Node/PV 등 일부 리소스는 namespace에 속하지 않습니다.

## 용어 설명

- **StatefulSet**: 각 Pod의 안정적인 이름, 네트워크 식별자, 스토리지 연결을 유지하며 stateful 애플리케이션을 관리하는 컨트롤러
- **taint / toleration**: taint는 Node가 특정 Pod의 스케줄링을 거부하도록 표시하고, toleration은 Pod가 해당 taint를 허용하도록 설정하는 방식
- **affinity**: Pod를 특정 Node 또는 다른 Pod와 가깝게 혹은 떨어지게 배치하도록 스케줄링 선호/제약 조건을 정의하는 기능
- **nodeSelector**: Node label을 기준으로 Pod가 배치될 Node를 제한하는 간단한 스케줄링 조건
- **CRI (Container Runtime Interface)**: kubelet과 container runtime 사이의 표준 인터페이스
- **ownerReference**: 어떤 Kubernetes 리소스가 다른 리소스의 소유자인지를 기록하는 메타데이터로, 종속 리소스의 생명주기 관리에 사용
- **ResourceQuota**: namespace 단위로 CPU, 메모리, 리소스 개수 등의 사용량 상한을 제한하는 리소스
