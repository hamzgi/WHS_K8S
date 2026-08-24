# WHS_K8S

VMware에 만들어 둔 Kubernetes 클러스터를 기준으로, Kubernetes 운영 도구와 GitOps 흐름을 직접 연습하기 위한 개인 저장소다.

팀 공용 레포에 작업을 올리기 전에 여기서 설치 과정을 다시 따라 해보고, 설정을 바꿨을 때 클러스터에 어떤 변화가 생기는지 확인하는 용도로 사용한다. 처음부터 완성된 구조를 만들기보다는 실습을 진행하면서 문서와 YAML을 조금씩 채워갈 예정이다.

## 현재 환경

현재는 Control Plane과 Worker Node 2대로 구성된 로컬 클러스터를 사용하고 있다.

| 항목 | 내용 |
| --- | --- |
| Control Plane | `master-node` |
| Worker | `worker-node1`, `worker-node2` |
| Kubernetes | `v1.29.15` |
| OS | Ubuntu 24.04.4 LTS |
| Container Runtime | containerd 2.2.1 |
| CNI | Calico |
| Helm | `v3.17.3` |
| Argo CD | `v2.11.7` |
| Argo CD Helm Chart | `7.3.11` |

클러스터 API 주소는 `192.168.241.10:6443`이고, Control Plane의 호스트 이름은 `master-node`다. 클러스터에는 초기 테스트용 `training/web` Pod가 남아 있으므로 실습 중에 임의로 삭제하지 않는다.

## 이 저장소에서 해볼 것

1. Helm Chart의 기본 구조와 values 사용 방법 익히기
2. Helm으로 Argo CD 설치하고 Release 상태 확인하기
3. 개인 GitHub 저장소를 Argo CD에 연결하기
4. GitHub의 YAML 변경 사항을 Kubernetes에 반영하기
5. Istio를 설치하고 Gateway API와 VirtualService를 각각 비교해보기
6. Prometheus와 Grafana로 클러스터 상태와 애플리케이션 지표 확인하기
7. Thanos는 기본 모니터링이 안정된 뒤 추가로 검토하기
8. 이후 Kubernetes 취약 실습 환경을 별도 namespace에 배포하고 침투테스트 기록 남기기

## 디렉터리 구조

```text
WHS_K8S/
├─ README.md
├─ .gitignore
├─ docs/
│  └─ 실습 과정과 설정 변경 내용을 기록
├─ helm/
│  ├─ charts/       # 직접 작성하거나 내려받아 사용하는 Chart
│  └─ values/       # 환경별 values 파일
├─ argocd/
│  └─ applications/ # Argo CD Application YAML
├─ platform/
│  ├─ istio/
│  │  ├─ gateway/       # Gateway API 관련 설정
│  │  └─ virtualservice/ # Istio VirtualService 실습
│  └─ monitoring/
│     ├─ prometheus/
│     ├─ grafana/
│     └─ thanos/
├─ apps/
│  └─ nginx/        # GitOps 동작 확인용 기본 애플리케이션
├─ labs/            # 이후 취약 Kubernetes 실습 환경
└─ scripts/         # 반복해서 사용하는 점검 명령어나 보조 스크립트
```

`platform`은 클러스터 공통 구성요소를 두는 곳이고, `apps`와 `labs`는 실제로 배포할 애플리케이션을 두는 곳이다. Argo CD가 어떤 경로를 바라보는지 헷갈리지 않도록 이 구분을 유지한다.

## 진행 순서

### 1. 클러스터 사전 점검

Control Plane에서 다음 명령어로 노드와 기본 Pod를 확인한다.

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

### 2. Helm 확인

현재 Helm Release와 Chart 저장소를 확인한다.

```bash
helm version
helm list -A
helm repo list
```

현재 Argo CD Chart 저장소는 다음과 같이 등록되어 있다.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 3. Argo CD

Argo CD는 `argocd` namespace에 Helm으로 설치했다. 설치 확인 명령어는 다음과 같다.

```bash
helm list -n argocd
kubectl get pods -n argocd
kubectl get svc -n argocd
```

현재는 설치를 직접 수행한 상태이고, 다음 단계에서 이 저장소를 Argo CD의 Application source로 연결한다.

Argo CD UI는 Control Plane에서 포트 포워딩한 뒤 Windows의 SSH 터널을 통해 접속한다.

Control Plane SSH 터미널:

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Windows PowerShell:

```powershell
ssh -N -L 8080:127.0.0.1:8080 k8s1@192.168.241.10
```

브라우저 주소:

```text
https://localhost:8080
```

### 4. GitOps 기본 동작

먼저 `apps/nginx`에 Deployment와 Service를 작성한다.

```text
GitHub main branch
        ↓
Argo CD Application
        ↓
Kubernetes Deployment / Service
        ↓
nginx Pod
```

처음에는 수동 Sync로 동작을 확인하고, 기본 흐름이 이해되면 자동 Sync와 self-heal 설정을 비교해본다.

### 5. Istio와 서비스 네트워킹

Istio 설치 후 다음 두 가지 방식을 각각 실습한다.

- Gateway API: `Gateway`, `HTTPRoute`
- Istio API: `Gateway`, `VirtualService`

같은 애플리케이션에 두 방식을 동시에 적용하지 않고, 어떤 리소스가 실제 트래픽을 처리하는지 확인하면서 하나씩 테스트한다.

### 6. 모니터링

Prometheus와 Grafana를 설치한 뒤 다음 내용을 확인한다.

- Node CPU와 메모리
- Pod 상태와 재시작 횟수
- Kubernetes API 및 시스템 구성요소 상태
- nginx 요청 지표

Thanos는 Prometheus와 Grafana가 정상적으로 동작한 후에 저장 기간과 장기 보존이 필요한지 판단해서 추가한다.

## 작업 방식

개인 실습에서도 팀 레포와 비슷한 흐름을 유지한다.

```text
main
 └─ feature/<주제>
       ↓
    로컬 테스트
       ↓
    커밋
       ↓
    main 병합
```

예시:

```bash
git switch -c feature/helm-nginx
git status
git add .
git commit -m "Add nginx Helm practice"
```

설치 버전은 가능한 한 명령어에 명시한다. 예를 들어 Argo CD는 다음처럼 Chart 버전을 고정한다.

```bash
helm install argocd argo/argo-cd \
  --version 7.3.11 \
  --namespace argocd \
  --create-namespace
```

## 기록할 내용

실습이 끝난 뒤 결과만 적기보다는 다음 내용을 함께 남긴다.

- 어떤 명령어를 실행했는지
- 설치 전과 후에 무엇이 달라졌는지
- Pod가 Pending 또는 CrashLoopBackOff가 된 이유
- values를 변경했을 때 실제 리소스가 어떻게 바뀌었는지
- Argo CD에서 OutOfSync가 발생한 원인
- 문제를 해결하기 위해 확인한 로그와 명령어
- 같은 문제가 다시 생겼을 때 먼저 확인할 항목

## 보안 관련 주의사항

- kubeconfig, 토큰, 비밀번호, 개인 키는 저장소에 올리지 않는다.
- Argo CD 초기 관리자 비밀번호는 로그인 후 변경하고 문서에 기록하지 않는다.
- 실습용 Secret은 실제 서비스 계정이나 개인 계정의 비밀번호를 사용하지 않는다.
- 취약 Lab은 `default`, `argocd`, `istio-system`, `monitoring` namespace와 분리한다.
- 명령어를 실행하기 전에 현재 context와 namespace를 확인한다.

```bash
kubectl config current-context
kubectl get namespace
```

## 현재 진행 상황

- [x] VMware Kubernetes 클러스터 상태 확인
- [x] Control Plane 및 Worker Node 2대 확인
- [x] Helm 3.17.3 설치
- [x] Argo CD Chart 저장소 등록
- [x] Argo CD Chart 7.3.11 설치
- [x] Argo CD UI 로그인
- [ ] 개인 GitHub 저장소를 Argo CD에 연결
- [ ] nginx GitOps 배포
- [ ] Istio 설치
- [ ] Gateway API 실습
- [ ] VirtualService 실습
- [ ] Prometheus와 Grafana 구축
- [ ] Thanos 검토
- [ ] Practice Lab 배포
- [ ] 침투테스트 보고서 작성

이 README도 실습 과정에서 구조가 바뀌거나 더 나은 방법을 찾으면 그때그때 수정한다.
