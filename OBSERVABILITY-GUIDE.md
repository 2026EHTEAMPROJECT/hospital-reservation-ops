# Hospital Reservation — 관측 & 접속 가이드

팀원이 브라우저에서 서비스·모니터링 도구를 확인할 수 있도록 정리한 가이드입니다.  
아래 명령어는 모두 **터미널(Terminal) 하나씩** 열어서 실행하세요.  
포트포워드는 터미널을 닫으면 끊기므로, 확인하는 동안 열어 두어야 합니다.

---

## 1. 서비스 (병원 예약 앱) — localhost:8088

Istio IngressGateway가 이미 `localhost:80`으로 노출되어 있지만,  
다른 프로세스와 충돌을 피하기 위해 **8088 포트포워드**를 사용합니다.

```bash
kubectl port-forward svc/istio-ingressgateway 8088:80 -n istio-system
```

브라우저에서 접속: **http://localhost:8088**

| 화면 | 경로 |
|---|---|
| 로그인 / 대시보드 | http://localhost:8088 |
| 의사 목록 + 예약 | 로그인 후 자동 이동 |
| 내 예약 내역 | 상단 메뉴 "마이페이지" |

> 회원가입은 Keycloak 페이지로 이동합니다. Keycloak을 포트포워드하지 않은 경우 "회원가입하기" 버튼이 동작하지 않습니다 (아래 4번 참고).

---

## 2. Grafana — 로그 조회 (Loki)

```bash
kubectl port-forward svc/grafana 3000:3000 -n istio-system
```

브라우저에서 접속: **http://localhost:3000**  
계정: `admin` / `admin` (초기값, 첫 접속 시 변경 요청)

### 로그 보는 방법

1. 왼쪽 메뉴 **Explore** 클릭
2. 상단 데이터소스에서 **loki-1** 선택
3. 쿼리 입력창에 아래 입력 후 `Shift+Enter`

**전체 서비스 로그**
```
{namespace="hospital-dev"}
```

**특정 서비스만**
```
{namespace="hospital-dev", app="booking-service"}
{namespace="hospital-dev", app="user-service"}
{namespace="hospital-dev", app="login-service"}
{namespace="hospital-dev", app="payment-service"}
{namespace="hospital-dev", app="notification-service"}
```

**키워드 포함 로그 필터**
```
{namespace="hospital-dev", app="booking-service"} |= "ERROR"
{namespace="hospital-dev"} |= "403"
```

4. 우상단 시간 범위를 **Last 15 minutes** 또는 **Last 1 hour**로 설정
5. **Label browser** 버튼으로 app·pod 라벨별 필터링 가능

---

## 3. Kiali — 서비스 메쉬 / 트래픽 확인

```bash
kubectl port-forward svc/kiali 20001:20001 -n istio-system
```

브라우저에서 접속: **http://localhost:20001**  
계정: `admin` / `admin`

### 주요 화면

| 메뉴 | 용도 |
|---|---|
| **Graph** | 서비스 간 트래픽 흐름 시각화 |
| **Workloads** | 각 파드 상태·사이드카 확인 |
| **Services** | 서비스 목록 및 VirtualService 설정 |
| **Istio Config** | Gateway/VirtualService/AuthzPolicy YAML 확인 |

### Graph에서 트래픽 보기

1. 왼쪽 **Graph** 클릭
2. Namespace → **hospital-dev** 선택
3. 상단 시간 범위를 **Last 1m** → **Last 10m**으로 넓히면 최근 요청 흐름이 표시됨
4. 노드 클릭 → 오른쪽에 요청 수 / 에러율 확인 가능

> **KIA0301 경고**가 hospital-gateway에 뜨는 것은 Keycloak용 gateway와 병원 gateway가 같은 포트를 쓰기 때문에 생기는 Kiali 경고입니다. 실제 동작에는 영향 없습니다.

---

## 4. Keycloak — 회원가입 / 사용자 관리

```bash
kubectl port-forward svc/keycloak 8080:8080 -n hospital-dev
```

| 용도 | URL |
|---|---|
| 회원가입 페이지 | http://localhost:8080/realms/hospital/protocol/openid-connect/registrations?client_id=hospital-frontend&response_type=code&redirect_uri=http%3A%2F%2Flocalhost%3A8088%2F |
| 관리자 콘솔 | http://localhost:8080/admin |

관리자 계정: `admin` / `admin` (realm: `master`)

> Keycloak은 `start-dev` 모드로 실행 중이라 **파드 재시작 시 서명키가 바뀝니다**.  
> 토큰이 갑자기 전부 401이 되면 아래 명령을 실행하세요:
> ```bash
> kubectl -n istio-system rollout restart deploy/istiod
> ```

---

## 5. ArgoCD — 배포 동기화 확인

```bash
kubectl port-forward svc/argocd-server 8443:443 -n argocd
```

브라우저에서 접속: **https://localhost:8443**  
(인증서 경고 → "고급" → "계속 진행" 클릭)

계정: `admin` / `admin1234`

### 동기화 상태 확인

| 상태 | 의미 |
|---|---|
| **Synced** + **Healthy** | 정상 배포 완료 |
| **OutOfSync** | git 최신과 클러스터가 다름 (Refresh/Sync 필요) |
| **Progressing** | 배포 롤아웃 진행 중 |
| **Degraded** | 파드 장애 |

**수동 동기화 방법**:  
앱 카드 클릭 → 상단 **Sync** 버튼 → **Synchronize**

### 터미널에서 동기화 상태 확인 (포트포워드 없이)

```bash
# 전체 앱 상태 한눈에 보기
kubectl get application -n argocd

# 특정 앱 상세 확인
kubectl describe application booking-service -n argocd | grep -A5 "Sync Status"
```

---

## 6. 전체 한 번에 띄우기 (터미널 4개)

터미널 탭을 4개 열고 각각 실행하세요:

```bash
# 탭 1 — 서비스 (병원 앱)
kubectl port-forward svc/istio-ingressgateway 8088:80 -n istio-system

# 탭 2 — Grafana (로그)
kubectl port-forward svc/grafana 3000:3000 -n istio-system

# 탭 3 — Kiali (트래픽)
kubectl port-forward svc/kiali 20001:20001 -n istio-system

# 탭 4 — ArgoCD (배포 상태)
kubectl port-forward svc/argocd-server 8443:443 -n argocd
```

| 도구 | URL | 계정 |
|---|---|---|
| 병원 예약 앱 | http://localhost:8088 | Keycloak 계정 |
| Grafana | http://localhost:3000 | admin / admin |
| Kiali | http://localhost:20001 | admin / admin |
| ArgoCD | https://localhost:8443 | admin / admin1234 |
| Keycloak 콘솔 | http://localhost:8080/admin | admin / admin |

---

## 7. 자주 겪는 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| 로그인 후 예약이 전부 403 | Keycloak 재시작으로 서명키 변경 | `kubectl -n istio-system rollout restart deploy/istiod` |
| ArgoCD 비밀번호 오류 | 브라우저 자동완성 입력 오류 | `admin1234` 수동 입력 |
| 8080 포트 이미 사용 중 | 다른 프로세스 점유 | `lsof -i :8080` 으로 PID 확인 후 `kill <PID>` |
| Grafana No data | 쿼리 미입력 또는 시간 범위 밖 | 쿼리 입력 + 시간 범위 확인 |
| Kiali Graph 비어 있음 | 트래픽 없는 기간 조회 | 시간 범위를 Last 10m 이상으로 늘림 |
