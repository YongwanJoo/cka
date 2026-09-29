# 문제풀이 - kubectl로 CRD 출력

> [Notion 원본](https://app.notion.com/p/3ea47fbe9c2a8083b19df0eb31d71d85)

<details>
<summary>리소스 정리</summary>

이 풀이는 클러스터 리소스를 생성하거나 수정하지 않는다. 제출·검토에 필요한 `~/resources.yaml`과 `~/subject.yaml`은 보관하고, 불필요해진 경우 개인 실습 환경에서 이 두 로컬 파일만 정리한다.

</details>

## 문제 요구사항

1. 클러스터에 이미 설치된 **cert-manager CRD 목록**을 `kubectl get`의 **기본 출력 형식** 그대로 `~/resources.yaml`에 저장한다. CRD를 새로 생성하는 작업이 아니다.
2. `Certificate` Custom Resource의 `spec.subject` 필드 설명을 `kubectl explain`으로 추출해 `~/subject.yaml`에 저장한다.

## 풀이

```bash
kubectl get crd | grep 'cert-manager.io' > ~/resources.yaml
kubectl explain certificate.spec.subject --api-version=cert-manager.io/v1 > ~/subject.yaml
```

`grep`은 cert-manager의 `cert-manager.io`와 `acme.cert-manager.io` 그룹 이름을 모두 찾는다. 첫 번째 명령에 `-o yaml`을 붙이면 문제에서 요구한 **기본 표 출력**이 아니므로 사용하지 않는다. `resources.yaml`이라는 파일 이름도 YAML 매니페스트 형식을 뜻하지 않는다.

## 결과 확인

```bash
cat ~/resources.yaml
head -20 ~/subject.yaml
```

첫 파일에 cert-manager CRD 이름들이, 두 번째 파일에 `subject` 필드 설명과 하위 필드가 들어 있는지 확인한다. 파일이 비어 있다면 CRD 설치 상태와 명령 오류를 먼저 확인한다.

## 관련 개념

- **CRD**는 새 API 리소스 종류의 정의이고, **Certificate CR**은 그 정의에 따라 만든 실제 객체다.
- `kubectl get crd`는 설치된 종류를 나열한다. `kubectl explain certificate.spec.subject`는 클러스터가 제공하는 OpenAPI 스키마에서 필드 문서를 읽는다.
- [kubectl get 공식 문서](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/) · [kubectl explain 공식 문서](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_explain/) · [cert-manager Certificate API](https://cert-manager.io/docs/reference/api-docs/)
