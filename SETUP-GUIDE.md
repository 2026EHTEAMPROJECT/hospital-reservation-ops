# Hospital Reservation — 팀원 클러스터 초기 세팅 가이드

처음 환경을 세팅하는 팀원이 따라하는 가이드입니다.  
한 번만 하면 이후엔 `OBSERVABILITY-GUIDE.md`의 포트포워드 명령어만 쓰면 됩니다.

---

## 0. 사전 준비 — 설치해야 할 것

| 도구 | 설치 방법 |
|---|---|
| **Docker Desktop** | https://www.docker.com/products/docker-desktop/ |
| **kubectl** | Docker Desktop 설치 시 자동 포함 |
| **istioctl 1.29.2** | 아래 명령 참고 |
| **helm** | `brew install helm` |

```bash
# istioctl 설치 (1.29.2)
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.29.2 sh -
sudo mv istio-1.29.2/bin/istioctl /usr/local/bin/
istioctl version   # 확인
```

---

## 1. Docker Desktop Kubernetes 활성화

1. Docker Desktop 실행
2. 상단 아이콘 → **Settings** → **Kubernetes** 탭
3. **Enable Kubernetes** 체크 → **Apply & Restart**
4. 완료 후 확인:

```bash
kubectl get nodes
# NAME             STATUS   ROLES           AGE   VERSION
# docker-desktop   Ready    control-plane   ...   v1.x.x
```

---

## 2. Istio 설치

```bash
# demo 프로필로 설치 (Ingress Gateway 포함)
istioctl install --set profile=demo -y

# 설치 확인
kubectl get pods -n istio-system
# istiod, istio-ingressgateway 등이 Running이면 정상
```

---

## 3. Istio 모니터링 애드온 설치 (Grafana / Kiali / Prometheus)

Istio 설치 디렉토리에 샘플 애드온 YAML이 포함되어 있습니다.

```bash
# istio 다운로드 디렉토리 안에서 실행
cd istio-1.29.2
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/kiali.yaml

# 설치 확인
kubectl get pods -n istio-system | grep -E "grafana|kiali|prometheus"
```

---

## 4. Loki + Promtail 설치 (로그 수집)

```bash
# Loki 저장소 추가
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# monitoring 네임스페이스 생성
kubectl create namespace monitoring

# Loki 설치
helm install loki grafana/loki \
  --namespace monitoring \
  --version 7.0.0 \
  --set loki.auth_enabled=false \
  --set loki.commonConfig.replication_factor=1 \
  --set loki.storage.type=filesystem \
  --set singleBinary.replicas=1

# Promtail 설치 (파드 로그 → Loki 전송)
helm install promtail grafana/promtail \
  --namespace monitoring \
  --version 6.17.1 \
  --set config.clients[0].url=http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push

# 설치 확인
kubectl get pods -n monitoring
```

Grafana에서 Loki 데이터소스 추가:
1. `http://localhost:3000` (포트포워드 먼저)
2. Connections → Data sources → Add → **Loki**
3. URL: `http://loki.monitoring.svc.cluster.local:3100`
4. **Save & Test**

---

## 5. ArgoCD 설치

```bash
# ArgoCD 네임스페이스 + 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 파드 Ready 대기
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=120s

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# 비밀번호를 admin1234로 변경 (팀 통일)
NEW_HASH=$(htpasswd -bnBC 10 "" admin1234 | tr -d ':\n' | sed 's/$2y/$2a/')
kubectl -n argocd patch secret argocd-secret \
  -p "{\"stringData\": {\"admin.password\": \"$NEW_HASH\", \"admin.passwordMtime\": \"$(date +%FT%T%Z)\"}}"
```

---

## 6. hospital-dev 네임스페이스 + Istio 사이드카 주입 설정

```bash
kubectl create namespace hospital-dev
kubectl label namespace hospital-dev istio-injection=enabled
```

---

## 7. ops 레포 클론 + ArgoCD 앱 등록

```bash
# ops 레포 클론
git clone https://github.com/2026EHTEAMPROJECT/hospital-reservation-ops.git
cd hospital-reservation-ops

# 서비스 앱 등록 (booking/user/payment/notification + infra)
kubectl apply -f argocd/

# 보안 앱 등록 (Keycloak + Istio 정책 + login-service)
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hospital-security
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/2026EHTEAMPROJECT/hospital-reservation-security.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: hospital-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

---

## 8. ArgoCD 동기화 확인

```bash
# 전체 앱 상태 확인
kubectl get application -n argocd

# 모두 Synced + Healthy가 될 때까지 대기 (1~3분 소요)
watch kubectl get application -n argocd
```

모든 앱이 **Synced + Healthy** 되면 준비 완료입니다.

---

## 9. 파드 전체 확인

```bash
kubectl get pods -n hospital-dev
```

아래 파드들이 모두 Running (2/2) 이어야 합니다:

```
booking-service       2/2   Running
booking-mysql         2/2   Running
user-service          2/2   Running
user-mysql            2/2   Running
payment-service       2/2   Running
payment-mysql         2/2   Running
notification-service  2/2   Running
rabbitmq              2/2   Running
login-service         2/2   Running
keycloak              2/2   Running
```

> `2/2`는 앱 컨테이너 + Istio 사이드카(envoy)가 모두 정상임을 의미합니다.

---

## 10. 접속 테스트

```bash
# 병원 앱 포트포워드
kubectl port-forward svc/istio-ingressgateway 8088:80 -n istio-system
```

브라우저: **http://localhost:8088** → 로그인 화면이 나오면 성공

이후 모니터링 접속 방법은 **OBSERVABILITY-GUIDE.md** 참고.

---

## 자주 겪는 문제

| 증상 | 해결 |
|---|---|
| `istioctl install` 후 파드가 Pending | Docker Desktop 메모리를 최소 8GB로 설정 (Settings → Resources) |
| ArgoCD 앱이 계속 OutOfSync | `kubectl -n argocd patch app <앱명> -p '{"operation":{"sync":{}}}' --type=merge` |
| Keycloak 파드가 안 뜸 | `kubectl logs -n hospital-dev deploy/keycloak` 로 에러 확인 |
| `2/2` 아닌 `1/2` 상태 | Istio 사이드카 주입 실패 → `kubectl delete pod <파드명> -n hospital-dev` 로 재시작 |
