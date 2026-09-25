# 문제풀이 - PVC 복구하기

> 원본 Notion 페이지에 포함된 이미지는 임시 서명 URL이라 GitHub에 안정적으로 보존하지 않았습니다.

<details>
<summary>사전 환경 구성</summary>

```bash
mkdir -p /home/cka0001/mariadb
kubectl create ns mariadb
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/pvc/storageclass.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/pvc/pv.yaml
curl -o ~/mariadb-deploy.yaml https://raw.githubusercontent.com/kubetm/exam-c/main/pvc/mariadb-deploy.yaml
```

</details>

<details>
<summary>리소스 정리</summary>

```bash
kubectl delete ns mariadb
kubectl delete pv mariadb-pv
kubectl delete sc local-path
rm -rf mariadb-deploy.yaml
rm -rf /home/cka0001/mariadb
```

</details>

## 요구사항

1. 이미 존재하는 PersistentVolume을 재사용하기
2. `mariadb` namespace 내에 `mariadb`라는 PersistentVolumeClaim 생성
   1. Access mode: `ReadWriteOnce`
   2. storage: `250Mi`
3. `/mariadb-deploy.yaml` 경로의 Deploy 파일을 수정하기

## 풀이

```bash
# PV 확인
k get pv

# PV 정보 확인
k describe pv mariadb-pv
```

PVC 생성:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
  namespace: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
  storageClassName: local-path
```

적용 및 확인:

```bash
k apply -f pvc.yaml
k get pvc -n mariadb

# mariadb-deploy.yaml 수정
k apply -f ~/mariadb-deploy.yaml

k get pvc -n mariadb
k get deploy -n mariadb
k get pods -n mariadb
```

## 용어 설명

- **PersistentVolume (PV)**: 실제 영구 스토리지 자원을 Kubernetes 리소스로 표현한 것
- **PersistentVolumeClaim (PVC)**: Pod가 필요한 스토리지 용량, AccessMode, StorageClass 등의 조건을 지정해 요청하는 리소스
- **ReadWriteOnce (RWO)**: 볼륨을 하나의 Node에서 read-write로 마운트할 수 있는 AccessMode. 하나의 Pod만 사용할 수 있다는 의미는 아님
- **StorageClass**: 동적 프로비저닝에 사용할 provisioner와 스토리지 정책을 정의하는 리소스
- **Bound**: PVC와 PV의 조건이 맞아 서로 연결된 상태
