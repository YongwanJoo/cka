# 문제풀이 - CRI 설치

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a808da2b7d1ce77b81de0)

<details>
<summary>리소스 정리</summary>

공용 실습 노드나 실행 중인 클러스터에서는 runtime과 네트워크 모듈을 임의로 중지하거나 해제하지 않는다. 개인 실습 노드에서 이 풀이가 새로 만든 `/etc/sysctl.d/99-cka-cri.conf`만, 다른 설정이 참조하지 않는 것을 확인한 뒤 정리한다. 기존 설정 파일을 덮어쓰거나 `br_netfilter`를 강제로 해제하지 않는다.

</details>

## 문제 요구사항

1. Docker Engine이 이미 설치된 Linux 노드에 문제에서 지정한 `~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb`를 설치한다.
2. 패키지가 제공하는 `cri-docker` 서비스와 소켓을 활성화하고 시작한다.
3. 문제 이미지의 네 가지 커널 매개변수를 지정한 값으로 설정한다. **이 문제는 kubeadm 클러스터 초기화까지 요구하지 않는다.**

## 풀이

```bash
sudo dpkg -i ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
sudo systemctl enable --now cri-docker.socket cri-docker.service

sudo modprobe br_netfilter
sudo modprobe nf_conntrack
cat <<'EOF' | sudo tee /etc/sysctl.d/99-cka-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
EOF
sudo sysctl --system
```

`sysctl` 설정 파일은 `키 = 값` 형식이다. 기존 풀이의 `to 1`은 유효한 설정 문법이 아니다. `br_netfilter`와 `nf_conntrack` 모듈을 먼저 로드해 해당 키를 읽을 수 있게 한다.

## 결과 확인

```bash
systemctl is-active cri-docker.service cri-docker.socket
ls -l /run/cri-dockerd.sock
sysctl net.bridge.bridge-nf-call-iptables net.ipv6.conf.all.forwarding net.ipv4.ip_forward net.netfilter.nf_conntrack_max
```

서비스와 소켓이 활성 상태이고, 네 값이 각각 `1`, `1`, `1`, `131072`인지 확인한다. Docker Engine 자체가 CRI를 제공하는 것은 아니므로 Kubernetes와 연결할 때 `cri-dockerd`가 필요하다. 나중에 kubeadm이 여러 runtime 소켓을 발견하면 `--cri-socket=unix:///run/cri-dockerd.sock`을 지정한다.

## 공식 참고

- [Kubernetes: kubeadm 설치와 runtime 소켓](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes: Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [cri-dockerd systemd 서비스 정의](https://github.com/Mirantis/cri-dockerd/blob/master/packaging/systemd/cri-docker.service)
