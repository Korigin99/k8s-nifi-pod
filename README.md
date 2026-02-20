# NiFi → Kubernetes API → Python Pod 생성 테스트 (Rancher Desktop / k3s)

로컬 Rancher Desktop(k3s, containerd)에서 **NiFi 2.6.0**이 Kubernetes API를 사용해 **Python Pod**를 생성하는 테스트입니다.  
Python 컨테이너는 로그만 출력하는 최소 구성입니다.  
아래 프로세서 설정은 **Apache NiFi 2.6.0** 문서 기준입니다.

## 사전 조건

- Rancher Desktop 설치 및 Kubernetes(k3s) 실행 중
- `kubectl`이 Rancher Desktop 컨텍스트로 연결됨 (`kubectl cluster-info` 확인)

## 1. 매니페스트 적용 및 이미지 풀

Rancher Desktop(k3s + containerd)에서는 **이미지를 미리 풀한 뒤** 매니페스트를 적용하는 편이 좋습니다.  
아래 순서대로 하면 NiFi·Python 이미지가 풀된 상태에서 배포됩니다.

### 1.1 네임스페이스 적용

```powershell
# 저장소 루트(k8s-nifi-pod)에서 실행
# 예) cd "D:\path\to\k8s-nifi-pod"

kubectl apply -f .\k8s\00-namespaces.yaml
```

### 1.2 이미지 풀 (NiFi, Python)

containerd/k3s는 Pod를 만들 때 이미지를 풀하므로, 임시 Pod를 띄웠다 삭제해 두 이미지를 미리 받습니다.

```powershell
# NiFi 이미지 풀 (apache/nifi:2.6.0)
kubectl run img-pull-nifi -n nifi-system --image=apache/nifi:2.6.0 --restart=Never -- sleep 60
kubectl wait --for=condition=Ready pod/img-pull-nifi -n nifi-system --timeout=300s
kubectl delete pod img-pull-nifi -n nifi-system

# Python 이미지 풀 (python:3.11-slim) — NiFi가 나중에 Pod 생성 시 사용
kubectl run img-pull-python -n python-jobs --image=python:3.11-slim --restart=Never -- sleep 10
kubectl wait --for=condition=Ready pod/img-pull-python -n python-jobs --timeout=120s
kubectl delete pod img-pull-python -n python-jobs
```

### 1.3 NiFi 배포 및 RBAC 적용

```powershell
kubectl apply -f .\k8s\nifi-rbac.yaml
kubectl apply -f .\k8s\nifi-deployment.yaml
```

이미지를 1.2에서 풀해 두었으면 NiFi Pod가 곧바로 `Running`이 됩니다.  
1.2를 생략해도 됩니다. 이 경우 `k8s/nifi-deployment.yaml` 적용 시 k3s가 이미지를 그때 풀하며, NiFi Pod가 `ContainerCreating` 상태로 잠시 머무를 수 있습니다.

### 1.4 (선택) 로컬 테스트용 Python 이미지 빌드 → containerd에 올리기

`print("test")`만 하는 테스트 이미지를 만들어 Rancher Desktop(containerd)에 넣고, 그 이미지로 Pod를 띄우는 흐름을 테스트할 수 있습니다.

1. **Docker CLI가 Rancher Desktop을 바라보도록**  
   (Rancher Desktop 실행 후) `docker context use rancher-desktop`

2. **이미지 빌드** (같은 containerd를 쓰는 k3s가 이 이미지를 사용합니다)

```powershell
cd .\python
docker build -t test-python:latest .
```

3. **이미지 확인** (선택)

```powershell
docker images test-python
```

4. NiFi 플로우의 **Custom Text**에는 `templates/python-pod-template.json` 대신 **`templates/python-pod-template-local.json`** 내용을 넣습니다.  
   - 이미지: `test-python:latest`  
   - `imagePullPolicy: Never` → 레지스트리에서 안 받고 로컬(containerd) 이미지만 사용.

## 2. NiFi 접속

**NiFi 2.x는 HTTP를 지원하지 않고 HTTPS(8443)만 사용합니다.**  
Service는 ClusterIP이므로, Rancher Desktop의 **포트 포워드** 기능으로 로컬에서 접속합니다.

### 포트 포워드 설정 (Rancher Desktop)

1. Rancher Desktop 메뉴에서 **Kubernetes** → **Port Forwarding** (또는 서비스 목록에서 **nifi** 선택)
2. `nifi-system` 네임스페이스의 **nifi** 서비스에 대해 포트 포워드 추가  
   - 로컬 포트: `8443` (또는 원하는 포트)  
   - 원격 포트: `8443`
3. 포트 포워드가 활성화되면 브라우저에서 **https://localhost:8443/nifi** 접속

(또는 터미널에서 `kubectl port-forward -n nifi-system svc/nifi 8443:8443` 로 포워드한 뒤 동일하게 접속할 수 있습니다.)

4. 자체 서명 인증서 경고 → **고급** → **localhost(으)로 이동** (또는 **계속**)
5. 로그인: **admin** / **AdminPassword123**

NiFi 기동에 2~3분 걸릴 수 있으므로 Pod가 Running이어도 잠시 기다린 뒤 접속하세요.

## 3. NiFi 플로우: Kubernetes API로 Pod 생성

NiFi가 **클러스터 내부**에서 동작하므로 `https://kubernetes.default.svc`와 ServiceAccount 토큰으로 API 호출합니다.

### 3.1 프로세서 구성

| 순서 | 프로세서            | 설정 요약 |
|------|---------------------|-----------|
| 1    | **GenerateFlowFile** | Data Format=Text, Unique FlowFiles=false, **Custom Text**에 Pod JSON 붙여넣기 (아래 3.4 참고). |
| 2    | **ReplaceText**      | Replacement Strategy=**Literal Replace**, Search Value=`PLACEHOLDER`, Replacement Value=`${now():format('yyyyMMddHHmmss')}`. |
| 3    | **ExecuteStreamCommand** | 토큰을 속성에 넣고, flow file 본문(JSON)은 그대로 유지. |
| 4    | **InvokeHTTP**       | POST로 K8s API 호출, Body=flow file 내용, Dynamic Property `Authorization` = `Bearer ${k8s.token}`. |

### 3.2 ExecuteStreamCommand 상세 (NiFi 2.6.0)

| Display Name | 값 |
|--------------|-----|
| **Command Path** | `cat` |
| **Command Arguments** | `/var/run/secrets/kubernetes.io/serviceaccount/token` |
| **Ignore STDIN** | `true` (flow file 내용을 cat에 넘기지 않음 → 본문 유지) |
| **Output Destination Attribute** | `k8s.token` (명령 출력을 이 속성에 저장) |
| **Max Attribute Length** | `4096` (기본 256은 K8s SA 토큰 길이에 부족하므로 반드시 증가) |

- NiFi Pod에 ServiceAccount 토큰이 기본 마운트 경로에 있어야 합니다.

### 3.3 InvokeHTTP 상세 (NiFi 2.6.0)

- **HTTP Method**: `POST`
- **Remote URL**: `https://kubernetes.default.svc/api/v1/namespaces/python-jobs/pods`
- **Content-Type**: `application/json`
- **Dynamic Property** 추가: 이름 `Authorization`, 값 `Bearer ${k8s.token}` (Expression Language로 이전 프로세서의 속성 참조)
- **Send Body**: flow file content 사용 (기본)
- **SSL Context Service**: K8s API TLS 검증용. 테스트 시 인증서 검증 완화 서비스 사용 가능 (선택)

### 3.4 GenerateFlowFile (NiFi 2.6.0) — "Custom Text"에 넣을 JSON

- **Data Format**: `Text`
- **Unique FlowFiles**: `false` (Custom Text가 적용되려면 false여야 함)
- **Custom Text**: 아래 JSON을 그대로 붙여넣기

- **공개 이미지 사용**: `templates/python-pod-template.json` 내용을 복사 (이미지: `python:3.11-slim`).
- **로컬 테스트 이미지 사용** (1.4에서 빌드한 `test-python:latest`): `templates/python-pod-template-local.json` 내용을 복사.

**PLACEHOLDER**는 그대로 두고 붙여넣습니다.  
- **ReplaceText (NiFi 2.6.0)**: Replacement Strategy = **Literal Replace**, Search Value = `PLACEHOLDER`, Replacement Value = `${now():format('yyyyMMddHHmmss')}`.  
  (표현식 `${now():format('yyyyMMddHHmmss')}`는 NiFi 2.6 Expression Language로 실행 시점 타임스탬프로 치환됩니다.)

한 줄로 넣을 경우 예:

```json
{"apiVersion":"v1","kind":"Pod","metadata":{"name":"python-log-PLACEHOLDER","namespace":"python-jobs","labels":{"app":"python-log"}},"spec":{"restartPolicy":"Never","containers":[{"name":"python-worker","image":"python:3.11-slim","command":["python","-c"],"args":["import sys; from datetime import datetime; print('[python-pod] hello from k8s pod', datetime.now().isoformat(), flush=True); sys.exit(0)"],"resources":{"requests":{"memory":"128Mi","cpu":"50m"},"limits":{"memory":"256Mi","cpu":"200m"}}}]}}
```

## 4. 실행 확인

플로우 실행 후 Pod 생성 여부와 Python 로그 확인:

```powershell
kubectl get pods -n python-jobs
kubectl logs -n python-jobs -l app=python-log --tail=20
```

Pod 이름이 `python-log-<timestamp>` 형태라면 NiFi에서 생성된 것입니다.  
로그에 `[python-pod] hello from k8s pod ...` 가 출력되면 정상 동작입니다.

## 5. 리소스 정리

```powershell
kubectl delete -f .\k8s\nifi-deployment.yaml
kubectl delete -f .\k8s\nifi-rbac.yaml
kubectl delete -f .\k8s\00-namespaces.yaml
```

## 파일 설명

| 파일 | 용도 |
|------|------|
| `k8s/00-namespaces.yaml` | `nifi-system`, `python-jobs` 네임스페이스 |
| `k8s/nifi-rbac.yaml` | NiFi SA가 `python-jobs`에 Pod create/get/list/watch/delete 할 수 있는 Role·RoleBinding |
| `k8s/nifi-deployment.yaml` | NiFi Deployment, ServiceAccount, Service (containerd 표준 이미지) |
| `templates/python-pod-template.json` | NiFi에서 POST Body로 사용할 Pod 템플릿 (공개 이미지 `python:3.11-slim`) |
| `templates/python-pod-template-local.json` | 로컬 빌드 이미지 `test-python:latest`용 템플릿 (`imagePullPolicy: Never`) |
| `python/app.py` | 테스트용 스크립트 (`print("test")`) |
| `python/Dockerfile` | 테스트 이미지 빌드용 (base: python:3.11-slim) |

Python 이미지는 `python:3.11-slim`(표준 OCI 이미지)이며, containerd에서 그대로 사용 가능합니다.
