# Pod가 Node에 배치되기까지 알아야할 속성들

> [Notion 원본](https://app.notion.com/p/3e947fbe9c2a80338cf9f03ed59690a6)

> **배치 흐름:** Pod 생성 요청 → Namespace 정책 검사 → 스케줄러가 배치 가능한 Node 필터링 → 적합한 Node 점수화·선택 → kubelet이 Pod 실행
이 페이지는 **① 생성 허용 ② 자원 적합성 ③ Node 선택 ④ 배치 후 보호** 순서로 읽으면 됨.

## 1. 생성 요청이 허용되는가: Namespace 정책

스케줄러가 Node를 고르기 전에 API Server의 admission 단계에서 Namespace 정책을 확인함.
- **LimitRange:** 컨테이너·Pod 등의 `requests`와 `limits`에 기본값, 최소값, 최대값 등을 적용하거나 범위를 검사함. 범위를 위반하면 생성 요청이 거부될 수 있음.
- **ResourceQuota:** Namespace 전체의 CPU·메모리 요청/제한 합계, Pod 개수 등 사용량의 상한을 둠. 새 Pod를 만들면 할당량을 초과하는 경우 생성 요청이 거부될 수 있음.
- 둘은 목적이 다름. LimitRange는 **개별 객체의 값**, ResourceQuota는 **Namespace 전체 합계**를 다룸. 여러 팀이 Namespace를 공유할 때 유용함.

## 2. Node에 자리가 있는가: requests와 limits

- **`requests`:** 스케줄러가 배치 가능 여부를 판단할 때 쓰는 예약 기준. 이미 배치된 Pod들의 요청량과 새 Pod의 요청량이 Node의 **allocatable** 자원을 넘으면 그 Node는 후보에서 제외됨. 실사용량이 낮아도 요청량 기준으로 부족하면 Pending이 될 수 있음.
- **`limits`:** 실행 중 컨테이너가 사용할 수 있는 상한. CPU 한도를 넘으면 제한(throttling)되고, 메모리 한도를 넘으면 OOM 종료가 발생할 수 있음. 스케줄러의 기본 배치 기준은 limit이 아니라 request임.
- 요청량이 없으면 해당 자원에 대한 스케줄링 예약이 없을 수 있음. 다만 **limit만 지정하면 request를 같은 값으로 채우는 동작**이나 LimitRange 기본값이 적용될 수 있으므로 최종 Pod 스펙을 확인할 것.
- Node의 모든 requests가 예약되면 **Pod 객체 생성이 막히는 것이 아니라**, 적합한 Node가 없다면 Pod가 Pending 상태로 남음. ResourceQuota 초과는 생성 단계에서 거부될 수 있어 둘을 구분해야 함.

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi
```

```bash
kubectl describe node <node-name>       # Allocatable, Allocated resources
kubectl describe pod <pod-name>         # 최종 requests/limits와 Events
kubectl get limitrange,resourcequota -n <namespace>
```

## 3. 어느 Node를 고르는가: 필터링과 점수화

스케줄러는 먼저 **하드 조건을 만족하지 못하는 Node를 제외**하고, 남은 후보에 점수를 매겨 Node를 선택함. 단순히 빈 자원이 가장 많은 Node에만 배치하는 것은 아님.

### Node의 label을 기준으로 선택

- **`nodeSelector`:** Node label의 단순 일치 조건. 지정한 label을 모두 가진 Node만 후보가 됨.
- **`nodeAffinity`:** Node label에 대한 더 표현력 있는 규칙. `requiredDuringSchedulingIgnoredDuringExecution`은 반드시 만족해야 하는 하드 조건, `preferredDuringSchedulingIgnoredDuringExecution`은 가능한 한 따르는 선호 조건임.
- **`nodeName`:** Node 이름을 직접 지정하며 스케줄러를 우회함. 일반적인 배치 요구는 label 기반 규칙을 사용하고, 이 필드는 특수한 경우에만 사용함.
- `IgnoredDuringExecution`은 배치 후 Node label이 바뀌었다고 이미 실행 중인 Pod를 자동 퇴거시키지 않는다는 뜻임.

### 다른 Pod의 위치를 기준으로 선택

- **`podAffinity`:** 특정 label의 Pod와 같은 토폴로지 영역에 배치. `topologyKey: kubernetes.io/hostname`이면 같은 Node, `topology.kubernetes.io/zone`이면 같은 Zone을 뜻함.
- **`podAntiAffinity`:** 특정 Pod와 같은 토폴로지 영역을 피함. 하드 조건과 선호 조건을 선택할 수 있음.
- **`topologySpreadConstraints`:** Node나 Zone 사이에 동일한 워크로드의 Pod 수가 지나치게 쏠리지 않도록 분산. `DoNotSchedule`은 조건 위반 시 배치하지 않고, `ScheduleAnyway`는 분산을 선호함.

### Node의 taint를 허용할지 결정

- **`taints`는 Node에, `tolerations`는 Pod에** 설정함. toleration은 해당 taint 때문에 제외되지 않도록 허용할 뿐, 그 Node에 반드시 배치하라는 뜻은 아님.
- `NoSchedule`: 새 Pod 배치를 막음. `PreferNoSchedule`: 가능하면 피함. `NoExecute`: 새 Pod 배치를 막고, toleration이 없는 기존 Pod도 퇴거시킴.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  nodeSelector:
    disk: ssd
  tolerations:
  - key: dedicated
    operator: Equal
    value: web
    effect: NoSchedule
  containers:
  - name: nginx
    image: nginx:stable
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
```

이 예시는 `disk=ssd` label이 있는 Node만 후보로 두고, `dedicated=web:NoSchedule` taint가 있어도 배치 가능하게 함. 최종 배치는 자원, 다른 taint, affinity 등 나머지 조건도 만족해야 함.

## 4. 배치 후에는 무엇이 Pod를 보호하는가

- **`priorityClassName`:** Pod의 우선순위를 지정. 높은 우선순위 Pod가 스케줄링되지 못하면 더 낮은 우선순위 Pod의 선점(preemption)을 시도할 수 있음. Node 자원 압박에 따른 퇴거 판단에도 우선순위가 고려됨.
- **PodDisruptionBudget (PDB):** `minAvailable` 또는 `maxUnavailable`로 **자발적 중단** 시 허용할 Pod 수를 제한. `kubectl drain`의 Eviction API 요청 등에 적용됨. Node 장애, Node 자원 압박에 따른 퇴거, 직접 삭제를 모두 막는 보호막은 아님.
- **`terminationGracePeriodSeconds`:** 정상 종료를 기다리는 유예 시간. 배치 위치를 정하는 속성은 아니며 Pod 종료 단계에 적용됨.
- **컨트롤러:** Deployment/StatefulSet이 관리하는 Pod가 사라지면 원하는 replica 수를 맞추기 위해 대체 Pod를 생성함. 새 Pod는 다시 스케줄링되므로 같은 Node에 배치된다는 보장은 없음.

## CKA에서 빠르게 확인할 명령어

```bash
kubectl get pod <pod-name> -o wide
kubectl describe pod <pod-name>               # Pending 이유와 Events
kubectl get nodes --show-labels
kubectl describe node <node-name>              # taints, allocatable, 요청량
kubectl get priorityclass
kubectl get pdb -A
```

**Pending 진단 순서:** Namespace 정책으로 생성이 거부됐는지 → Pod의 requests가 Node allocatable에 맞는지 → nodeSelector/affinity/topology 조건이 충족되는지 → taint에 대응하는 toleration이 있는지 → `describe pod`의 Events에 `FailedScheduling` 사유가 무엇인지 확인.

### 공식 문서

- [리소스 requests와 limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) · [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Pod를 Node에 배치하기](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) · [Taints와 Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Pod Priority와 Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) · [Pod 중단과 PDB](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
