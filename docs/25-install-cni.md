# 문제풀이 - CNI 설치

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a80e284a4d49cc6d42f12)

## 문제 요구사항

1. 제시된 Flannel 또는 Calico 매니페스트 중 하나로 Pod 간 통신을 구성한다.
2. **NetworkPolicy를 실제로 집행**할 수 있어야 한다. 이 조건에는 Calico를 선택한다.
3. 매니페스트 파일로 설치하며 Helm은 사용하지 않는다.

## 풀이
Tigera Operator 매니페스트는 Operator와 CRD를 설치한다. **이 명령만으로 Calico 데이터 플레인 설치가 끝나지는 않는다.** 같은 버전의 `Installation` Custom Resource도 생성해야 한다.

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,POD_CIDR:.spec.podCIDR
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
curl -fL https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml -o ~/calico-custom-resources.yaml
# 다음 명령 전에 파일의 spec.calicoNetwork.ipPools[0].cidr 확인
kubectl create -f ~/calico-custom-resources.yaml
```

- 내려받은 파일의 기본 IP pool은 `192.168.0.0/16`이다. 클러스터의 Pod CIDR이 다르면 파일에서 `spec.calicoNetwork.ipPools[0].cidr`를 **클러스터 Pod CIDR에 맞춘 뒤** 마지막 명령을 실행한다. 노드의 `POD_CIDR`가 비어 있으면 kubeadm의 Pod network 설정도 확인한다.
- 기존 CNI가 이미 설치된 클러스터에 다른 CNI를 덧씌우지 않는다. 실습 문제의 클러스터 상태를 먼저 확인한다.

## 결과 확인

```bash
kubectl get tigerastatus
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system
kubectl get nodes
```

`tigerastatus`의 주요 구성 요소가 `AVAILABLE=True`이고 Calico Pod가 실행되며 Node가 `Ready`인지 확인한다. NetworkPolicy 리소스가 보이는 것만으로 **집행**까지 증명되지는 않는다. 필요하면 격리된 실습 네임스페이스에서 deny 정책을 적용해 통신 차단을 시험한다.

## 실수 포인트

- Flannel 매니페스트만 설치하면 이 문제의 NetworkPolicy 집행 조건을 만족하지 못한다.
- `tigera-operator.yaml`과 `custom-resources.yaml`의 버전을 **둘 다 v3.29.2**로 맞춘다.
- [Calico 공식 설치 문서](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises) · [v3.29.2 custom-resources.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml)
