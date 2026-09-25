# 시험 개요

## 시험 환경

- CKA는 **2시간 동안 진행되는 실습형 시험**이며, 명령줄에서 실제 Kubernetes 작업을 수행
- **2026-09-25 기준 시험 버전은 Kubernetes v1.35**
- 문제마다 지정된 **host / context / namespace**를 먼저 확인하고 작업
- `kubectl` 명령에 항상 `sudo`가 필요한 것은 아님. 노드의 시스템 설정이나 root 권한이 필요한 작업에서만 `sudo` 사용

## 시험 중 사용 가능한 문서

- Kubernetes Documentation: <https://kubernetes.io/docs>
- Kubernetes Blog: <https://kubernetes.io/blog>
- Helm Documentation: <https://helm.sh/docs>
- 문제의 **Quick Reference**에서 제공하는 task-specific 문서
- CKA에서는 Gateway API Documentation도 사용 가능: <https://gateway-api.sigs.k8s.io>
- 개인 북마크나 임의의 외부 웹사이트는 사용하지 않음

## 풀이 습관

1. 문제에 지정된 host/context 확인
2. namespace 확인
3. 리소스 현재 상태 확인
4. 필요한 최소 변경만 수행
5. `kubectl get`, `describe`, 실제 요청 명령 등으로 결과 검증

## 용어 설명

- **context**: `kubectl`이 접속할 Kubernetes 클러스터, 사용자 인증 정보, 기본 namespace 조합을 가리키는 kubeconfig 설정 단위
- **namespace**: 하나의 클러스터 안에서 namespaced 리소스를 논리적으로 구분하는 범위
- **host**: 문제에서 명령을 실행하도록 지정한 머신 또는 노드
