# 실습 환경

## Host / UTM

- Host: **MacBook Pro M3 Pro (Apple Silicon)**
- Hypervisor: **UTM**
- VM Architecture: **ARM64 (aarch64)**
- VM CPU core 수, RAM, Disk 용량은 현재 기록에서 확인되지 않아 임의로 적지 않음
- Network: `192.168.56.0/24` 대역의 Host-Only 계열 실습 네트워크
  - 게스트 NIC: `enp0s1`
  - VM IP: `192.168.56.40/24`
  - UTM UI에 표시되는 정확한 Network Mode 이름은 현재 기록에서 미확인

## OS / Node

- OS: **Rocky Linux 9.6 ARM64**
- Hostname: `k8s-master`
- Node 구성: **단일 Control Plane Node**
- 별도 Worker Node 없음
- Control Plane taint를 제거하여 실습 Pod도 `k8s-master`에 스케줄링 가능
- `/etc/hosts`: `192.168.56.40 k8s-master`

## Kubernetes

- Kubernetes: **v1.34.3**
  - kubeadm: 1.34.3
  - kubelet: 1.34.3
  - kubectl: 1.34.3
- 설치 방식: **kubeadm**
- API Server advertise address: `192.168.56.40`
- Pod CIDR: `20.96.0.0/12`
- Service CIDR: kubeadm 기본값 `10.96.0.0/12`
- 현재 실제 CKA 시험은 v1.35 기준이므로 실습 클러스터와 minor version 차이가 있음

```bash
kubeadm init --pod-network-cidr=20.96.0.0/12 \
  --apiserver-advertise-address=192.168.56.40
```

## Container Runtime / CNI

- containerd: **1.7.29**
- cgroup driver: **systemd**
- `/etc/containerd/config.toml`: `SystemdCgroup = true`
- CNI: **Calico 3.31.1**

## 추가 구성

- Helm: **v3.19.2 ARM64**
- Metrics Server: **0.8.1**
- ingress-nginx Helm Chart: **4.13.4**
- ingress-nginx Controller: **1.13.4**
- Gateway API CRD: **1.4.0**
- NGINX Gateway Fabric: **2.2.1**
- cert-manager: **1.19.1**

## OS 사전 설정

- Timezone: `Asia/Seoul`
- NTP: chronyd 사용
- SELinux: permissive
- firewalld: disabled
- swap: disabled
- IPv4 forwarding: enabled
- 실습 사용자: `cka0001`
  - passwordless sudo
  - `/home/cka0001/.kube/config`에 admin kubeconfig 복사
  - `k=kubectl` alias 및 bash completion 설정

## VM 시간 동기화 주의

UTM VM을 오래 꺼둔 뒤 부팅했을 때 VM 시간이 과거로 남아 TLS 인증서 검증이 실패하면서 image pull이 `ImagePullBackOff`가 된 적이 있음.

확인:

```bash
date
timedatectl
chronyc tracking
chronyc sources -v
```

시간이 크게 틀린 경우:

```bash
sudo chronyc makestep
```

정상 상태 확인 포인트:

- `chronyc sources -v`에서 현재 선택된 NTP source가 `^*`
- `chronyc tracking`에서 `Leap status: Normal`

## 용어 설명

- **Control Plane**: API Server, Scheduler, Controller Manager, etcd 등을 통해 클러스터 전체 상태를 관리하는 제어 영역
- **taint**: 특정 Pod가 toleration을 가지지 않으면 해당 Node에 스케줄링되지 않도록 제한하는 Node 속성
- **Pod CIDR**: Pod에 할당할 IP 주소 범위
- **Service CIDR**: ClusterIP 타입 Service 등에 할당할 가상 IP 주소 범위
- **Container Runtime**: kubelet의 요청을 받아 실제 컨테이너를 생성하고 실행하는 소프트웨어. 이 환경에서는 containerd 사용
- **CNI (Container Network Interface)**: Kubernetes Pod 네트워크를 구성하기 위한 표준 인터페이스. 이 환경에서는 Calico 사용
- **cgroup driver**: Linux cgroup을 사용해 컨테이너의 CPU, 메모리 등 자원을 관리하는 방식
- **CRD (CustomResourceDefinition)**: Kubernetes API에 사용자가 새로운 리소스 종류를 추가할 수 있게 하는 확장 기능
