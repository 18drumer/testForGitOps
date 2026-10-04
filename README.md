# k8s-gitops

Argo CD 로 dev / staging 환경에 **PostgreSQL · Kafka · Redis · 백엔드(HPA 2~8)** 를 자동 배포하는 저장소입니다.

## 구성 요약

| 구분 | 도구 | 비고 |
|---|---|---|
| PostgreSQL | CloudNativePG (chart 0.29.1) | dev 1대 / staging 2대 |
| Kafka | Strimzi (chart 1.2.0), KRaft | 1대 |
| Redis | OT redis-operator (chart 0.26.1) | 단일 인스턴스 |
| HPA 지표 | metrics-server (chart 3.14.0) | |
| 외부 접속 | Cloudflare Tunnel (cloudflared 2026.9.3) | `*.rio.dpdns.org` → Traefik |
| Ingress | Traefik (chart 41.6.1, v3.7.13) | Service ClusterIP (외부 노출은 터널이 담당) |
| 인증서 | cert-manager (chart v1.21.2) + Let's Encrypt | DNS-01 (Cloudflare API), staging / prod 발급자 |
| 백엔드 | Deployment + HPA | Pod 2~8개 |

배포 순서 (sync-wave): 오퍼레이터(0) → DB·Kafka·Redis(1) → 백엔드(2)

## 디렉토리

```
k8s-gitops/
├── bootstrap/root-app.yaml        # 최초 1회 kubectl apply 하는 파일
├── argocd-apps/                   # Argo CD Application 목록 (root 가 읽음)
│   ├── op-cnpg.yaml / op-strimzi.yaml / op-redis.yaml / op-metrics-server.yaml
│   ├── op-traefik.yaml / cloudflared.yaml / op-cert-manager.yaml / cert-issuers.yaml
│   ├── infra-dev.yaml / infra-staging.yaml
│   └── backend-dev.yaml / backend-staging.yaml
├── infra/
│   ├── base/                      # postgres.yaml, kafka.yaml, redis.yaml
│   └── overlays/{dev,staging}/    # 환경별 차이
├── platform/
│   ├── cloudflared/               # cloudflared Deployment (토큰 Secret 은 Git 밖에서 생성)
│   └── cert-manager/              # ClusterIssuer (letsencrypt-staging / letsencrypt-prod)
└── apps/backend/
    ├── base/                      # configmap, deployment, service, hpa
    └── overlays/{dev,staging}/
```

---

# 설치 가이드 (A to Z)

아래 명령은 모두 **kubectl 이 동작하는 곳(보통 마스터 노드 VM)** 에서 실행합니다.

## 1. 클러스터 사전 점검

```bash
kubectl get nodes                 # 모든 노드 Ready 확인
kubectl get storageclass          # (default) 표시된 StorageClass 가 있는지 확인
```

**StorageClass 가 없으면** (Hyper-V 로 직접 만든 클러스터는 대부분 없음) DB·Kafka·Redis 의 PVC 가 Pending 에 멈춥니다. 실습용 local-path 를 설치합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get storageclass          # local-path (default) 확인
```

권장 여유 자원: 워커 노드 합계 메모리 8GB 이상 (dev + staging 동시 실행 기준).
마스터 노드 1대뿐이면 taint 때문에 Pod 가 안 뜰 수 있습니다 → `kubectl taint nodes --all node-role.kubernetes.io/control-plane-`

## 2. GitHub 에 올리기

1. GitHub 에서 **Public** 저장소 `k8s-gitops` 생성 (README 추가 체크 X)
2. 저장소 주소를 본인 것으로 바꾸고 push

```bash
cd k8s-gitops
grep -rl YOUR_GITHUB_ID . | xargs sed -i 's/YOUR_GITHUB_ID/실제깃허브아이디/g'
git init && git add . && git commit -m "init gitops"
git branch -M main
git remote add origin https://github.com/실제깃허브아이디/k8s-gitops.git
git push -u origin main
```

> Private 저장소라면 5단계 전에 Argo CD UI → Settings → Repositories → CONNECT REPO 에서 GitHub 토큰으로 등록해야 합니다.

## 3. Argo CD 설치

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.3/manifests/install.yaml

kubectl get pods -n argocd -w     # 전부 Running 될 때까지 대기 (1~3분), Ctrl+C 로 종료
```

## 4. Argo CD 웹 UI 접속

Hyper-V VM 밖(Windows)에서 접속하기 위해 NodePort 로 엽니다.

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'
kubectl get svc argocd-server -n argocd
# PORT(S) 예: 80:30080/TCP,443:30443/TCP  → 443 옆 숫자가 접속 포트

# 초기 admin 비밀번호
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Windows 브라우저에서 `https://<노드VM IP>:<443 에 매핑된 포트>` 접속 → 인증서 경고는 "계속" → ID `admin` / 위 비밀번호로 로그인.
(노드 IP 는 `kubectl get nodes -o wide` 의 INTERNAL-IP)

## 5. 루트 앱 등록 (이것 한 번이면 끝)

**Cloudflare Tunnel 사전 준비** (외부 도메인 접속용, 터널 토큰은 Public 저장소에 넣지 않음):

1. Cloudflare Zero Trust → Networks → Tunnels → Create a tunnel (Cloudflared) → 토큰 복사 (설치 명령은 실행 X)
2. 터널의 Public Hostname: `*` . `rio.dpdns.org` → `HTTP` / `traefik.traefik.svc.cluster.local:80`
3. DNS 에 `*` CNAME → `<터널ID>.cfargotunnel.com` (Proxied) 가 있는지 확인, 없으면 추가
4. 토큰 Secret 생성:

```bash
kubectl create ns cloudflared
kubectl create secret generic cloudflared-token -n cloudflared --from-literal=token=<터널 토큰>
```

**Let's Encrypt (cert-manager) 사전 준비** — Cloudflare API 토큰 Secret 생성:

1. Cloudflare 대시보드 → My Profile → API Tokens → Create Token → "Edit zone DNS" 템플릿
   - Permissions: `Zone / DNS / Edit`, `Zone / Zone / Read`
   - Zone Resources: `Include / Specific zone / rio.dpdns.org`
2. 토큰 Secret 생성:

```bash
kubectl create ns cert-manager
kubectl create secret generic cloudflare-api-token -n cert-manager --from-literal=api-token=<API 토큰>
```

이후 새 서비스 공개는 Cloudflare 수정 없이 Ingress 의 `host: <이름>.rio.dpdns.org` 만 추가하면 됩니다.
(무료 인증서는 한 단계 서브도메인만 지원 → `api-dev.rio.dpdns.org` O, `api.dev.rio.dpdns.org` X)

```bash
git clone https://github.com/18drumer/testForGitOps.git   # VM 에 저장소가 없다면
kubectl apply -f k8s-gitops/bootstrap/root-app.yaml
```

이후 흐름:
1. `root` 앱이 `argocd-apps/` 의 12개 Application 을 생성
2. 오퍼레이터 4개 + Traefik + cloudflared + cert-manager 설치 (CRD 생성)
3. infra-dev / infra-staging 이 PostgreSQL·Kafka·Redis 생성 (CRD 가 아직 없으면 자동 재시도)
4. backend-dev / backend-staging 배포

전체 완료까지 5~10분 정도 걸립니다 (이미지 다운로드 속도에 따라 다름).

## 6. 배포 확인

UI 에서 13개 앱이 모두 **Synced / Healthy** 가 되면 성공입니다. 터미널로도 확인:

```bash
kubectl get applications -n argocd

kubectl get cluster -n dev        # PostgreSQL : STATUS "Cluster in healthy state"
kubectl get kafka -n dev          # Kafka      : READY True
kubectl get redis -n dev
kubectl get pods -n dev
kubectl get hpa -n dev            # TARGETS 가 cpu: x%/70% 로 보이면 metrics-server 정상
```

각 구성요소 접속 테스트:

```bash
# PostgreSQL
kubectl exec -it pg-1 -n dev -- psql -c "\l"

# Redis
kubectl exec -it redis-0 -n dev -- redis-cli ping          # PONG

# Kafka : 토픽 생성 후 목록 조회
kubectl exec -it kafka-dual-role-0 -n dev -- \
  bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test
kubectl exec -it kafka-dual-role-0 -n dev -- \
  bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

# 백엔드
kubectl run curl -n dev --rm -it --image=curlimages/curl --restart=Never -- curl -s http://backend

# 백엔드 (외부: Cloudflare → 터널 → Traefik → backend)
curl https://api-dev.rio.dpdns.org/   # dev
curl https://api-prd.rio.dpdns.org/   # staging
```

Let's Encrypt 인증서 발급 확인:

```bash
kubectl get clusterissuer                     # letsencrypt-staging / prod  READY True
kubectl get certificate -n dev                # api-dev-tls  READY True (1~3분)
kubectl describe certificate api-dev-tls -n dev   # 실패 시 Events 확인
kubectl get challenge -A                      # 진행 중인 DNS-01 검증 (끝나면 사라짐)

# Traefik 이 실제로 내보내는 인증서 확인 (발급자: (STAGING) ... 또는 Let's Encrypt R1x)
kubectl run tls -n dev --rm -it --image=alpine/openssl --restart=Never -- \
  s_client -connect traefik.traefik.svc.cluster.local:443 -servername api-dev.rio.dpdns.org </dev/null 2>/dev/null \
  | grep -E "subject=|issuer="
```

Traefik 대시보드 (NodePort 30900, 내부망 전용 — 터널/인터넷에는 노출 안 됨):

Windows 브라우저에서 `http://172.20.10.11:30900/dashboard/` (노드 IP 아무거나, 끝의 `/` 필수)

## 7. HPA 테스트 (Pod 2 → 최대 8)

터미널 2개를 엽니다.

```bash
# 터미널 1 : 관찰
kubectl get hpa,pods -n dev -w

# 터미널 2 : 부하 발생 (Ctrl+C 로 중지)
kubectl run load -n dev --rm -it --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://backend >/dev/null; done"
```

CPU 가 70% 를 넘으면 Pod 가 늘어나고, 부하를 멈추면 약 5분 뒤 2개로 줄어듭니다.
(nginx 는 가벼워서 잘 안 늘어나면 `load` 를 이름만 바꿔 여러 개 띄우세요.)

## 8. GitOps 체험 — Git 만 고치면 클러스터가 바뀐다

`infra/overlays/staging/kustomization.yaml` 의 `value: 2` 를 `3` 으로 바꾸고 push.

```bash
git commit -am "staging pg 3대" && git push
```

Argo CD 는 3분마다 Git 을 확인합니다. 바로 반영하려면 UI 에서 `infra-staging` → **REFRESH**.
`kubectl get pods -n staging -l cnpg.io/cluster=pg` 로 pg-3 이 생기는 것을 확인할 수 있습니다.

반대로 `kubectl scale` 같은 수동 변경은 selfHeal 때문에 Git 상태로 되돌아갑니다.

## 9. 백엔드 새 버전 배포

백엔드 이미지는 testForK8s 저장소의 GitHub Actions 가 Docker Hub(`start31/for_k8s_test`)에 올립니다.
태그는 커밋마다 `sha-xxxxxxx` 로 붙으므로, 배포할 태그로 `apps/backend/overlays/{dev,staging}/kustomization.yaml` 의 newTag 를 수정 후 push:

```yaml
images:
  - name: start31/for_k8s_test
    newTag: sha-xxxxxxx
```

DB 접속 정보는 환경변수(`DB_HOST`, `DB_PORT`, `DB_NAME` ← ConfigMap / `DB_USERNAME`, `DB_PASSWORD` ← `pg-app` Secret)로 주입됩니다.
Spring 프로필은 dev 는 `dev`, staging 은 `prod` 입니다.

## 10. 문제 해결

| 증상 | 원인 / 조치 |
|---|---|
| PVC 가 Pending | StorageClass 없음 → 1단계 local-path 설치 |
| infra 앱이 "the server could not find the requested resource" | 오퍼레이터 설치 중. 자동 재시도됨. 오래가면 UI 에서 SYNC |
| backend Pod `CreateContainerConfigError` | `pg-app` Secret 이 아직 없음 (DB 생성 중). DB 가 뜨면 자동 해결 |
| HPA TARGETS 가 `<unknown>` | metrics-server 준비 중. `kubectl top nodes` 가 동작하면 정상 |
| Pod Pending (Insufficient memory) | 노드 메모리 부족 → VM 메모리 증설 또는 staging 앱 삭제 |
| cloudflared Pod `CreateContainerConfigError` | `cloudflared-token` Secret 없음 → 아래 "Cloudflare Tunnel" 참고 |
| 도메인 접속 시 404 | Traefik 까지는 도착. Ingress 의 host 가 접속한 도메인과 같은지 확인 |
| 도메인 접속 시 502 / 1033 | 터널 끊김 또는 Public Hostname URL 오타 (`traefik.traefik.svc.cluster.local:80`) |
| 앱이 OutOfSync 반복 | 클러스터를 수동으로 바꾼 것. Git 을 수정해야 함 |

## 11. 전체 삭제

```bash
kubectl delete applications --all -n argocd   # Argo CD 앱 삭제 (먼저 해야 자동 복구가 안 됨)
kubectl delete ns dev staging                  # DB·Kafka·Redis·백엔드 삭제 (데이터 포함)
```
