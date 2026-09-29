# 문제풀이 - Helm으로 ArgoCD 배포

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a80aa8962e11c1bf1fafc)

<details>
<summary>실습 환경 준비용 CRD 설치 (문제 풀이에서는 생략)</summary>

개인 연습 클러스터에 CRD가 없을 때만 사용한다. 문제에서는 이미 설치되어 있으므로 실행하지 않는다.

```bash
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/application-crd.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/applicationset-crd.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/appproject-crd.yaml
```

</details>

<details>
<summary>실습 후 정리 (선택)</summary>

**기본 정리:** 본인의 개인 실습 환경에서만 실행한다.

```bash
helm uninstall argocd -n argocd
helm repo remove argo
rm ~/argo-helm.yaml
```

**네임스페이스:** `argocd` 네임스페이스가 이 실습 전용이고 다른 리소스가 없을 때만 `kubectl delete ns argocd`를 실행한다.

**CRD:** 문제에서 사전 제공된 CRD는 삭제하지 않는다. 직접 설치했고 다른 사용처가 없을 때만 다음 명령을 실행한다.

```bash
kubectl delete -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/application-crd.yaml
kubectl delete -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/applicationset-crd.yaml
kubectl delete -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/appproject-crd.yaml
```

</details>

## 문제 요구사항

- 공식 Argo CD Helm 저장소 `https://argoproj.github.io/argo-helm/`를 `argo`라는 이름으로 등록한다.
- 이미 설치된 Argo CD CRD는 Helm이 다시 설치하지 않도록 설정한다.
- `argo/argo-cd` 차트 **8.6.4**를 릴리스 이름 `argocd`, 네임스페이스 `argocd`로 렌더링해 `~/argo-helm.yaml`에 저장한다.
- 렌더링할 때와 **같은 차트 버전 및 설정**으로 `argocd` 릴리스를 설치한다. 서버 UI 접근 설정은 요구하지 않는다.

## 풀이

```bash
helm repo add argo https://argoproj.github.io/argo-helm/
helm template argocd argo/argo-cd --namespace argocd --version 8.6.4 --set crds.install=false > ~/argo-helm.yaml
helm install argocd argo/argo-cd --namespace argocd --version 8.6.4 --set crds.install=false --create-namespace
```

`helm template`은 YAML을 파일로 저장하는 단계이고, `helm install`이 실제 클러스터에 배포하는 단계다. 설치 시 `--create-namespace`를 붙여 `argocd` 네임스페이스가 없어도 생성되게 한다.

## 결과 확인

```bash
ls -lh ~/argo-helm.yaml
helm list -n argocd
helm get values argocd -n argocd
kubectl get pods -n argocd
```

- YAML 파일이 생성되어 있어야 한다.
- `helm list`에서 `argocd` 릴리스의 차트 버전이 `8.6.4`, 상태가 `deployed`인지 확인한다.
- `helm get values`에서 `crds.install: false`가 적용됐는지 확인한다.
- Pod 상태를 확인한다. 첨부된 실습 기록에서는 설치가 `STATUS: deployed`로 끝났고, Argo CD Pod들이 `Running`이었다. 이는 당시 실행 결과이며 현재 클러스터 상태를 다시 확인한 것은 아니다.

## 옵션과 실수 포인트

- `helm repo add NAME URL`: 저장소 이름 `argo`와 URL을 둘 다 적는다.
- `helm template RELEASE CHART`, `helm install RELEASE CHART`: 두 명령 모두 릴리스 이름 `argocd`가 첫 번째 인수다. 차트 이름은 `argo/argo-cd`다.
- `--version 8.6.4`: **Helm 차트 버전**을 고정한다. 저장소 검색에서 보이는 최신 버전으로 바꾸면 문제 조건과 달라진다.
- `--set crds.install=false`: 차트의 CRD 설치를 끈다. 템플릿 생성과 설치에 모두 동일하게 넣는다.
- `--namespace argocd`: 대상 네임스페이스다. 옵션 사이 공백을 빠뜨리면 인수가 잘못 해석된다.
- `~/argo-helm.yaml`은 **템플릿 결과 파일**이다. `helm install`에는 이 파일을 넘기지 않는다.

> **기억할 한 줄:** `repo add` → `template > 파일` → `install`. 두 번째와 세 번째 명령의 릴리스 이름·차트·버전·CRD 설정을 일치시킨다.

## 관련 개념

- CRD(CustomResourceDefinition)는 Kubernetes에 새 리소스 종류를 등록한다. 이 문제에서는 이미 설치되어 있으므로 풀이 중 다시 만들거나 삭제하지 않는다.
- Argo CD 차트 문서에서 `crds.install=false`는 차트의 CRD 설치를 비활성화하는 설정이다. [Argo CD Helm 차트 문서](https://github.com/argoproj/argo-helm/blob/main/charts/argo-cd/README.md)
