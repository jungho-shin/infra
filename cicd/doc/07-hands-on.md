# 7. 실습: 처음부터 구축

## 이 장의 목표

1~6장에서 익힌 개념을 실제로 조립해, **develop 브랜치에 push 하면 dev 에 배포되고, main 은 승인 후 prod 에 배포되는** 환경을 내 PC(WSL2) 안에 만든다.
이 문서는 README 를 대신하지 않는다. 각 단계에서 **무엇을 왜 하는지, 끝났는지 어떻게 확인하는지, 어디서 자주 막히는지**를 알려 주는 길잡이다. 실제 명령 전체는 각 절에 적은 `README.md:줄` 을 열어 따라 한다.

진행 원칙:

- **순서를 바꾸지 않는다.** 뒤 단계가 앞 단계의 결과물(토큰, 레지스트리, DNS)을 쓴다.
- **단계마다 "확인" 명령을 통과한 뒤 넘어간다.** 문제는 생긴 단계에서 잡는 것이 가장 싸다.
- **비밀 값은 화면·파일·채팅에 남기지 않는다.** README 가 `read -rsp` 와 권한 600 파일을 쓰는 이유다.

```
0 사전 준비 → 1 클러스터 → 2 GitLab → 3 Nexus → 4 Argo CD → 5 Jenkins → 6 webhook → 7 파이프라인
                                                                                     └→ 7.5 prod → 롤백 → 10 재시작 점검
```

---

## 7.1 0장 사전 준비 (도구 버전, 토큰 보관 위치)

**무엇을 왜.** 도구 버전이 어긋나면 나중에 원인을 찾기 어려운 오류가 난다. 예를 들어 kubectl 은 클러스터(v1.34)와 마이너 버전 ±1 안이어야 한다. 또 구축 중에 비밀번호·토큰이 9개 가까이 생기므로, 어디에 보관할지 미리 정해 둔다.

**따라 할 곳.** `README.md:119` (0.1 도구 표), `README.md:151` (0.2 보관 위치 표)

**확인.**

```bash
ps -p 1 -o comm=                     # systemd
command -v docker                    # /usr/bin/docker (Docker Desktop 이 아니다)
kind version                         # v0.30.0 이상
kubectl version --client             # v1.33 ~ v1.35
git config --global user.name        # 값이 있어야 한다
```

**자주 막히는 곳.**

- 리포지토리를 `/mnt/c/...` 아래에 clone 한다. **WSL 의 ext4(`~/workspace/cicd`)에** 둔다(`README.md:144`).
- git 작성자가 없으면 나중에 gitops 커밋이 `Author identity unknown` 으로 실패한다.

> 0.2 표를 출력해 옆에 두고, 값을 만들 때마다 "어디에 보관했는지" 체크하면 헷갈리지 않는다.

### 학습 노트: 실제 점검 결과와 준비 (2026-09-27)

Windows PowerShell 과 `wsl.exe` 로 README 0.1 표를 그대로 점검한 결과.

| 항목 | 결과 | 판정 | 왜 필요한가 (앞 장 연결) |
|---|---|---|---|
| Windows RAM / WSL 메모리 | 47.9GB / WSL 23GB | ✅ (16GB↑, `.wslconfig` 불필요) | GitLab 하나가 2~4Gi (`kind-config.yaml:31-34`) |
| Windows 80 포트 | 비어 있음 | ✅ | ingress-nginx 정문 hostPort 80 (6.1) |
| WSL 배포판 / systemd | Ubuntu 26.04.1, `systemd` | ✅ | docker 를 서비스로 자동 기동 |
| 루트 FS | ext4 | ✅ | `/data` 와 리포지토리는 ext4 에 (6.4 — `/mnt/c` 금지) |
| docker (WSL 네이티브) | **없음** | ❌ 설치 | kind 노드 = Docker 컨테이너 (6.1) |
| kind | **없음** | ❌ 설치 (v0.30.0) | `kind-config.yaml` 의 노드 이미지·CoreDNS 원본이 v0.30.0 기준 (`coredns-cm.yaml:41-42`) |
| kubectl | **없음** | ❌ 설치 (v1.34) | 클러스터 v1.34 와 마이너 ±1 |
| git 작성자 | **없음** (`~/.gitconfig` 자체가 없음) | ❌ 설정 | 없으면 수동 커밋이 `Author identity unknown` |
| `~/workspace/cicd` | 없음 | ❌ clone | README 경로가 모두 이 위치 기준 |
| `/data` | 없음 | (7.2 에서 만든다) | hostPath PV 창고 (6.4) |

- **Docker Desktop 이 아니라 WSL 네이티브 docker** 인 이유: Desktop 은 별도 VM 에서 돌아 `/data` 마운트·localhost 포워딩 경로가 달라진다.
  README 전체가 "WSL 안의 docker" 를 전제로 쓰였다(`kind-config.yaml:7-11`).
- **비밀 값 보관 원칙 (README 0.2)**: 토큰은 `read -rsp` 로 화면에 안 보이게 받아 WSL 홈의 **권한 600 파일**, 사람 로그인용 비밀번호는 비밀번호 관리 도구.
  리포지토리·채팅·스크린샷 금지 — 4.3 `withCredentials` 가 로그를 가리는 것과 같은 이유. 이 학습 대화에도 값을 붙여 넣지 않는다.

재점검 (설치 후): docker 29.1.3 (enabled·active, docker 그룹 적용), kind v0.30.0, kubectl v1.34.12, `~/workspace/cicd` clone(ext4) ✅.
git 작성자만 여전히 비어 있음 — `git config --global` 이 실제로 적용됐는지 `git config --global --list` 로 확인할 것
(흔한 원인: `sudo` 를 붙여 root 의 설정에 들어감, 또는 다른 배포판/창에서 실행).
→ 사용자가 다시 설정, `git config --global --list` 로 name·email 확인 ✅. **7.1 완료.**

## 7.2 1장 클러스터 준비

**무엇을 왜.** 모든 구성요소가 올라갈 쿠버네티스 클러스터를 kind 로 만든다. kind 노드는 Docker 컨테이너라서 **포트 매핑한 것만 밖(WSL/Windows)에서 보인다.** 그래서 웹 UI 는 모두 80 포트 하나(Ingress)로 받고, 호스트 이름(`*.example.com`)으로 나눈다.

이 절은 여섯 조각으로 되어 있고, 각각 이유가 있다.

| 순서 | 작업 | 왜 필요한가 | README |
|---|---|---|---|
| 1.1 | `/data` 디렉터리, DNS 고정, inotify | 영구 데이터를 WSL 디스크에 두고, 외부 DNS 가 멈추는 문제를 예방 | `README.md:213` |
| 1.2 | `kind create cluster` | 노드 3대, 80/443 매핑, 스토리지 노드 1대 | `README.md:269` |
| 1.3 | ingress-nginx (kind 판) | 80 포트로 들어온 요청을 호스트 이름별로 나눔 | `README.md:319` |
| 1.5 | Windows hosts 파일 | 브라우저에서 `gitlab.example.com` → `127.0.0.1` | `README.md:385` |
| 1.6 | CoreDNS rewrite | **클러스터 안**에서도 같은 이름이 Ingress 로 가게 함 | `README.md:412` |
| 1.8 | metrics-server | prod 의 HPA 가 CPU 지표를 읽음 | `README.md:536` |

클러스터 설정 파일에서 "control-plane 에만 80/443 을 연다"는 부분이 핵심이다.

`kind-config.yaml:85`

```yaml
    labels:
      ingress-ready: "true"
    extraPortMappings:
      # ingress-nginx hostPort. 모든 웹 UI(gitlab/jenkins/nexus/argocd)가 여기로 들어온다.
      - containerPort: 80
        hostPort: 80
```

**확인.**

```bash
kubectl get nodes -L ingress-ready,storage-node        # 3대 Ready, 라벨이 각각 한 대씩
kubectl -n ingress-nginx get pods -o wide              # NODE 가 devops-control-plane
curl -sS -o /dev/null -w '%{http_code}\n' -H 'Host: nexus.example.com' http://localhost/   # 지금은 404 가 정상
kubectl top nodes                                      # 1분쯤 뒤 값이 나온다
```

`404` 는 실패가 아니다. "nginx 까지는 도착했는데 아직 그 이름의 Ingress 가 없다"는 뜻이다(`README.md:373`).

**자주 막히는 곳.**

- **CoreDNS rewrite(1.6)를 건너뛴다.** 파드 안에서 `nexus-docker.example.com` 이 `127.0.0.1`(파드 자신)로 풀려, 나중에 kaniko push·Argo CD 의 Git 접근이 모두 `connection refused` 로 죽는다. 이름이 "안 풀리는" 게 아니라 "엉뚱하게 풀리는" 문제라 진단이 어렵다(`README.md:425`).
- ingress-nginx 를 kind 판이 아닌 매니페스트로 설치하면 80 포트가 열리지 않는다(`README.md:321`).
- 1.6·1.8 은 `kind delete cluster` 와 함께 사라진다. 클러스터를 다시 만들면 다시 적용한다.

### 학습 노트: 실습 기록 — README 1.1 WSL 사전 준비 (2026-09-27)

- `/data` + UID 소유자 ✅ (`nexus 200`, `jenkins 1000`, `postgres 999`, `gitlab` 은 사용자 소유 — omnibus 가 스스로 맞추므로 무관), ext4 ✅.
- inotify `max_user_instances` 128 → 1024 ✅.
- **DNS 고정이 한 번 되돌아갔다.** 사용자가 `/etc/resolv.conf` 를 일반 파일로 바꾼 직후, 점검용으로 `wsl.exe` 를 새로 호출하자
  다시 `/mnt/wsl/resolv.conf` 링크(`nameserver 172.22.224.1`)로 돌아왔다.
  원인: `/etc/wsl.conf` 의 `generateResolvConf = false` 는 **WSL 이 다시 시작돼야** 읽힌다. 그 전까지는 옛 설정(자동 생성)이 살아 있어,
  새 세션이 붙을 때 링크를 다시 만든다.
  → 순서: `wsl.conf` 수정 → **`wsl --shutdown`** → 재접속 후 `rm` + `tee` → `ls -l /etc/resolv.conf` 가 링크(`->`)가 아닌지 확인.
  README 1.1 의 "설정 후 `wsl --shutdown` 으로 재시작해도 일반 파일로 남아 있는지 확인한다" 가 바로 이 확인이다.
- 재시도: `wsl --shutdown` 뒤 링크가 이번엔 **`../run/systemd/resolve/stub-resolv.conf`** (systemd-resolved) 를 가리키고 있었다.
  WSL 이 더는 만들지 않으니, Ubuntu 26.04 의 tmpfiles 규칙 `L! /etc/resolv.conf ... stub-resolv.conf` 가 **비어 있는 자리에** 링크를 만든 것.
  `L`(+ 없음)은 이미 있는 파일을 덮지 않으므로, 일반 파일로 써 둔 뒤엔 유지된다 → 새 세션에서도 일반 파일 + 고정 DNS 3개 확인 ✅.
  사용자가 `wsl --shutdown` 을 한 번 더 하고 재접속해 **재시작 후에도 일반 파일 + 고정 DNS 유지**를 확인 ✅ — README 1.1 확인 항목 완료.

### 학습 노트: 실습 기록 — README 1.2 클러스터 생성 (2026-09-27)

- `kind create cluster` 성공, 3노드 v1.34.0 Ready. 라벨: `ingress-ready` = control-plane, `storage-node` = devops-worker 만 ✅ (6.1·6.4).
- `docker ps` 의 PORTS: **control-plane 에만** `80`, `443`, `30022`, `30082-30083`, `127.0.0.1:6443` — 두 worker 는 빈칸.
  → 6.1 "정문은 control-plane 에만" 을 눈으로 확인. 6443(API 서버)만 `127.0.0.1` 로 묶여 외부 노출이 없다.
- `/data`: devops-worker 에선 WSL 과 같은 내용·같은 UID, devops-worker2 엔 없음 ✅ (6.4 "창고 문은 한 채에만").
  시각이 14:35(UTC) vs WSL 23:35(KST) — 노드 컨테이너 시계가 UTC 라서, 같은 파일이다(README 1.2).
- containerd 미러 로드 확인: `ctr deprecations list` 에 `cri-registry-mirrors` 경고 = `kind-config.yaml` 의 미러가 **읽혔다는 증거** (6.5).
  (폐기 예정 형식이라는 경고 자체는 현재 무해 — 노드 이미지를 올릴 때 `config_path` 방식으로 바꿔야 할 수 있음.)
- 재시작 정책: 생성 직후 `on-failure` → `docker update --restart=unless-stopped` 로 바꿔야 WSL 재시작 때 노드가 자동으로 뜬다.

### 학습 노트: 실습 기록 — README 1.3 ingress-nginx, 1.4 접속 확인 (2026-09-27)

- 노드 재시작 정책 `unless-stopped` 적용 ✅.
- ingress-nginx v1.15.1 (kind 판) + `nodeSelector ingress-ready=true` → 컨트롤러가 **devops-control-plane** 에 Running ✅ (6.1 Q2 의 반대 경우를 예방).
- `reuse-port: false`, `worker-processes: 4` 적용 후 재시작.
- 1.4: WSL `curl -H 'Host: nexus.example.com' http://localhost/` → **404**, Windows `curl.exe ... http://127.0.0.1/` → **404** ✅.
  404 = nginx(안내 데스크)가 답했다 = 정문(포트 매핑)·localhost 포워딩·컨트롤러 위치가 모두 맞다. Ingress 규칙이 아직 없을 뿐(6.2 표).
- 이 시점 Windows hosts 파일에는 `*.example.com` 줄이 **하나도 없다** → 1.5 에서 추가.

### 학습 노트: 실습 기록 — README 1.5 hosts 파일 (2026-09-28)

- 관리자 PowerShell 에 **여러 줄 스크립트를 붙여 넣자 줄 순서가 뒤집혀** 들어가 파싱 오류(`InvalidLeftHandSide`). 파싱 오류라 아무것도 실행되지 않았다.
  → **한 줄** 명령으로 바꿔 해결(이미 있는 줄은 건너뛰는 멱등 스크립트, hosts 복사본으로 먼저 검증). 교훈: Windows PowerShell 콘솔엔 한 줄로 붙여 넣는다.
- 11줄 추가 ✅. Windows 에서 `nexus.example.com` → 127.0.0.1, `curl.exe http://nexus.example.com/` 이 **`-H` 없이** 404 ✅
  (① hosts 가 이름을 풀고 ② curl 이 URL 의 이름을 Host 헤더로 넣음 — 6.2 의 두 단계).

- `wsl --shutdown` 후: WSL `/etc/hosts` 에도 반영(`getent hosts nexus.example.com` → 127.0.0.1) ✅,
  노드 3대가 **손대지 않았는데 Up** — `unless-stopped` 가 동작 ✅, 곧 3대 Ready, WSL 에서도 `-H` 없이 404 ✅.
- 1.6 준비: 현재 클러스터의 CoreDNS 원본과 `bootstrap/coredns/coredns-cm.yaml` 에서 rewrite 줄을 뺀 내용이 **완전히 같음**을 diff 로 확인
  (파일이 kind v0.30.0 / k8s v1.34.0 기준이라 통째 교체해도 다른 설정이 날아가지 않는다 — `coredns-cm.yaml:41-44` 주의사항).

### 학습 노트: 실습 기록 — README 1.6 CoreDNS, 그리고 WSL 재시작 후 Pod 네트워크 장애 (2026-09-28)

- 증상: `kubectl run dnstest ... nslookup` 이 rewrite 적용 **전후 모두** `;; connection timed out; no servers could be reached`.
  → 답이 틀린 게 아니라 **DNS 서버(CoreDNS)에 닿지도 못함**. rewrite 와 무관한 더 아래층 문제.
- 진단 순서 (위에서 아래로)
  1. CoreDNS Pod 2개 Running/Ready, `kube-dns` Service·EndpointSlice 정상 → 서버 쪽 문제 아님.
  2. 노드별로 테스트 Pod 를 고정해(`nodeName`) 각 CoreDNS Pod IP 로 질의 → **같은 노드의 CoreDNS 만 응답, 다른 노드는 전부 timeout.**
     Service IP(10.96.0.10)는 kube-proxy 가 두 Pod 중 하나를 고르므로 **될 때도 안 될 때도** 있다(간헐적 = 헷갈림).
  3. 노드의 `ip route`: control-plane 엔 `10.244.1.0/24 via 172.18.0.2` 같은 **다른 노드 Pod 대역 경로**가 있는데, 두 worker 에는 **없다**.
     → 노드 간 Pod 통신 불가(갈 길 또는 돌아올 길이 없음).
  4. 경로를 까는 건 CNI **kindnet**(DaemonSet). worker 의 kindnet 로그: WSL 재시작 직후 API 서버가 아직 뜨는 중이라
     `TLS handshake timeout` 이 연달아 났고, 그 뒤로 노드 목록을 다시 받아 **경로를 깔지 못한 채** 멈춰 있었다.
     지금은 worker → API 서버(IPv4·IPv6 모두) 200 으로 정상 → **kindnet 을 재시작하면 경로를 다시 깐다.**
- 교훈
  - "Pod 가 Running/Ready" 와 "Pod 끼리 통신됨" 은 별개다. 노드 Ready 도 이것을 보장하지 않는다.
  - **WSL 재시작 → 노드 자동 기동(unless-stopped) 후에도 kindnet 이 경로를 못 까는 일이 생길 수 있다** → 7.11 재시작 점검 항목에 추가:
    `docker exec devops-worker ip route | grep 10.244` 에 다른 노드 대역이 보이는지.
  - **README 1.6 의 "127.0.0.1 이 나온다" 는 이 환경에선 재현되지 않는다.** 노드에서 `getent hosts nexus-docker.example.com` 은 **빈 결과**.
    노드 DNS 는 Docker 게이트웨이(172.18.0.1) → WSL 의 `/etc/resolv.conf` 로 가는데, 7.2 1.1 에서 이걸 **외부 DNS 로 고정**했기 때문에
    사슬이 더는 Windows DNS 프록시·hosts 파일에 닿지 않는다(WSL 자신의 `getent` 는 WSL `/etc/hosts` 덕에 127.0.0.1).
    즉 rewrite 가 없으면 Pod 는 "틀린 답(127.0.0.1)" 대신 "**답 없음**" 을 받는다 — 증상은 달라도 rewrite 가 필요한 건 같다.
    (6.3 에서 배운 "엉뚱하게 풀린다" 는 WSL 기본 DNS 설정일 때의 이야기.)

- 해결: `kubectl -n kube-system rollout restart ds/kindnet` → worker 에 `10.244.0.0/24 via 172.18.0.4`, `10.244.2.0/24 via 172.18.0.3` 경로 복구.
  노드 3 × (CoreDNS Pod 2 + Service) = 9 조합 모두 응답 ✅.
- rewrite 확인: Pod 에서 `nslookup nexus-docker.example.com` → **10.96.20.207** = `ingress-nginx-controller` 의 CLUSTER-IP ✅ (6.3 "1층 안내 데스크 내선").

### 학습 노트: 실습 기록 — README 1.8 metrics-server (2026-09-28)

- 설치 전 `kubectl top nodes` → `error: Metrics API not available` (검침원 없음 — 이 상태면 prod HPA 가 `<unknown>`, 6.6).
- `kubectl apply -k bootstrap/metrics-server` → 1분 뒤 `kubectl top nodes`·`top pods` 에 값이 나옴 ✅ (`--kubelet-insecure-tls` 패치 덕에 x509 없이 Ready).
- **CPU 0%, 메모리 1~3% 로 보이는 이유**: 노드 3대가 모두 WSL 전체(**88 코어, 약 23GiB**)를 자기 용량으로 보고한다.
  kind 노드는 VM 이 아니라 컨테이너라 자원을 나눠 갖지 않는다(마트료시카, 6.1). 스케줄러는 합계 264 코어·69GiB 가 있다고 믿지만
  실제로는 같은 23GiB 를 나눠 쓴다 → **메모리 부족은 스케줄러가 막아 주지 못한다.** GitLab(2~4Gi)을 올린 뒤 WSL 의 `free -g` 로 직접 본다.
- **7.2 (README 1장 클러스터 준비) 완료.**

## 7.3 2장 GitLab CE 설치 + 토큰 발급

**무엇을 왜.** 앱 리포지토리와 gitops 리포지토리가 모두 여기에 산다. omnibus 이미지 하나로 DB·Redis·git 서버까지 한 파드에 띄우므로 가장 무겁고(메모리 4~6Gi), 첫 기동에 8~15분 걸린다.

**따라 할 곳.** `README.md:593` (설치), `README.md:760` (토큰)

GitLab 은 `external_url` 로 clone 주소와 리디렉션을 만든다. 그래서 브라우저에서 쓰는 주소와 반드시 같아야 한다.

`bootstrap/gitlab/gitlab.yaml:113`

```ruby
external_url 'http://gitlab.example.com'
```

**토큰 두 종류를 구분한다.** 이 실습에서 가장 헷갈리는 부분이다.

| 누가 쓰나 | 종류 | 권한 | 언제 발급 |
|---|---|---|---|
| Argo CD (gitops 읽기만) | Deploy token | `read_repository` | 4장에서 gitops 프로젝트를 만든 직후 |
| Jenkins (앱 체크아웃 + gitops push) | Group access token `ci-bot`, **Maintainer** | `read_repository`, `write_repository` | 지금, 그룹 `my-group` 을 만든 뒤 |

권한을 최소로 나누는 것이 원칙이다. Argo CD 는 읽기만 하면 되므로 쓰기 권한을 주지 않는다.

**확인.**

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -H 'Host: gitlab.example.com' http://localhost/   # 302 (로그인 페이지로)
kubectl -n gitlab exec sts/gitlab -- gitlab-rails runner \
  'u = User.find_by(username: "root"); puts u ? "root EXISTS" : "root MISSING"'
ls -l ~/gitlab-group-token.txt       # 크기만 본다. 내용은 출력하지 않는다
```

**자주 막히는 곳.**

- **약한 root 비밀번호**(`admin123` 등)면 컨테이너가 exit 1 로 재시작을 반복한다. 메모리 문제처럼 보여서 헷갈린다. 비밀번호를 고친 뒤에도 root 계정이 안 생기는 함정이 있다(`README.md:652`).
- 로그인 ID 는 이메일이 아니라 `root` 다.
- Auto DevOps 를 끄지 않으면 push 할 때마다 쓸모없는 파이프라인이 Pending 으로 쌓인다(`README.md:646`).
- 토큰 role 을 Developer 로 주면 7장에서 gitops push 가 `pre-receive hook declined` 로 거부된다.

### 학습 노트: 실습 기록 — README 2.2 GitLab 설치 (2026-09-28)

- root 비밀번호: `read -rsp` → Secret `gitlab-secrets`(DATA 1) — 화면·셸 기록·파일·채팅 어디에도 남기지 않음.
- `kubectl kustomize bootstrap/gitlab | kubectl apply -f -` → PV 2개 + PVC 2개 + StatefulSet + Ingress.
- Pod 생애 (6장 연결):

| 시각 | 상태 | 의미 |
|---|---|---|
| 0~13s | `Pending`, NODE 없음 | PVC ↔ PV 짝짓기 + 스케줄링. PV 의 nodeAffinity 때문에 **devops-worker 만** 후보 (6.4) |
| 14s~117s | `ContainerCreating` | 노드 containerd 가 `gitlab/gitlab-ce:17.11.1-ce.0`(수 GB)을 pull — 트럭 기사 (6.5). Nexus 가 아직 없어 Docker Hub 에서 직접 |
| ~5m14s | `0/1 Running` | reconfigure + DB 마이그레이션. startupProbe(15s×80)가 기다려 줌 |
| 5m14s | `1/1 Running` | startupProbe 통과 → Ready → Service Endpoint 에 등록 |

- WSL `free -g`: used 5 / available 17 GiB — GitLab 이 올라온 뒤 실제 여유 (스케줄러는 노드당 23GiB 로 착각, 7.2 1.8).
- `curl http://gitlab.example.com/` → **302** (로그인 페이지로 리디렉션 = GitLab 이 응답), `root EXISTS id=1 state=active` ✅ — 비밀번호 거부 함정 없음.

### 학습 노트: 실습 기록 — 인스턴스 설정·그룹·토큰 (README 2.2 후반, 2.4) (2026-09-28)

- `gitlab-rails runner` 로 확인(읽기 전용): `auto_devops_enabled=false`, `signup_enabled=false` ✅, 그룹 `my-group` (private) ✅.
- `ci-bot` Group access token: 스코프 `read_repository,write_repository`, **access_level=40 (= Maintainer)**, revoked=false ✅.
  GitLab 은 그룹 토큰마다 **봇 사용자**(`group_2_bot_...`)를 만들어 그룹 멤버로 넣는다 — 토큰의 권한 = 그 봇의 role.
- 토큰 파일 `~/gitlab-group-token.txt`: `-rw-------`, 26 bytes (`glpat-` + 20자) — 내용은 출력하지 않고 확인.
- **401 과 403 의 차이로 토큰이 살아 있음을 확인**: 가짜 토큰으로 API 호출 → **401**(누군지 모름), 진짜 토큰 → **403**(누군지는 알지만 권한 없음).
  403 이 정상인 이유: 스코프가 git 전용(`*_repository`)이라 REST API(`read_api` 필요)는 거부된다. git clone/push 로의 실제 동작은 리포지토리를 만든 뒤 확인.
- ⚠️ **`expires_at=2026-10-28`** — 발급 화면 기본 만료가 30일이다. 만료되면 Jenkins 의 체크아웃·gitops push 가 401 로 **한꺼번에** 깨진다.
  학습 기간을 넘기면 재발급 → 파일 교체 → Jenkins Secret 갱신(4.5 재료별 반영 시점) 순서로 바꾼다. 달력에 적어 둘 것.
  → **바로 1년짜리로 재발급** (만료 2027-09-27). 확인: 새 토큰 revoked=false·Maintainer·같은 스코프, 옛 토큰 **revoked=true**,
  파일 갱신(00:59, 26 bytes, 600). 새 파일로 API 호출 → 403(= 유효한 토큰; 폐기된 옛 토큰이었다면 401). Jenkins 넣기 전이라 교체는 파일 하나로 끝.
  (재발급하면 봇 사용자도 새로 생긴다 — 이름은 같은 `ci-bot` 이어도 다른 봇.)
- **7.3 완료.** (2.3 git+ssh 는 선택 — HTTP clone 만 쓰므로 건너뜀. Deploy token 은 gitops 프로젝트를 만드는 7.5 에서.)

## 7.4 3장 Nexus 설치 + 레지스트리 구성

**무엇을 왜.** CI 가 만든 이미지를 보관하고, 클러스터가 그 이미지를 내려받는 창고다. 동시에 Docker Hub·gcr.io·quay.io 의 **프록시(캐시)** 역할도 해서, Jenkins 에이전트 이미지(kaniko, argocd)를 여기서 받는다.

**따라 할 곳.** `README.md:820` (설치), `README.md:848` (UI 에서 리포지토리 만들기), `README.md:985` (`regcred`)

만들 레지스트리의 관계를 그림으로 잡아 두면 UI 설정이 쉽다.

```
docker-hosted (8082)  ← CI 가 앱 이미지를 push. pull 은 인증 필요(regcred)
docker-group  (8083)  ← 익명 pull. 멤버 순서: hosted → docker-hub → gcr-proxy → quay-proxy
```

이 레지스트리는 TLS 없는 평문 HTTP 다. 그래서 접속하는 주체마다 예외 처리가 필요하고, 이 프로젝트는 세 곳에 나눠 이미 넣어 두었다(`README.md:909` 표).

- 노드의 이미지 pull: `kind-config.yaml:57` 의 `containerdConfigPatches`
- kaniko 의 push: CoreDNS rewrite + `--insecure` 옵션 (`ci/Jenkinsfile:105`)
- WSL 에서 수동 push: `localhost:30082` (Docker 가 localhost 는 예외로 취급)

마지막으로 앱이 뜰 네임스페이스 4곳에 pull 용 Secret `regcred` 를 만든다. Deployment 가 이 이름을 참조한다.

`manifests/react-app/base/deployment.yaml:67`

```yaml
      imagePullSecrets:
        - name: regcred
```

**확인.**

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -H 'Host: nexus.example.com' http://localhost/   # 200
docker exec devops-worker crictl pull nexus-docker-group.example.com/library/alpine:3.20  # 익명 pull 성공
kubectl get secret -A --field-selector metadata.name=regcred                              # 4개
```

**자주 막히는 곳.**

- 초기 비밀번호 `admin123` 을 바로 바꾸고, 첫 로그인 마법사의 익명 접근은 **Enable** 로 고른다.
- 8082/8083 포트는 **UI 에서 커넥터를 만들어야** 열린다.
- `docker-group` 멤버 순서를 바꾸면 Docker Hub 의 엉뚱한 `argoproj/argocd` 이미지를 받을 수 있다(`README.md:868`).
- `no basic auth credentials` 는 경로는 맞고 인증에서 막힌 것이다. 익명 접근 설정을 다시 본다.

### 학습 노트: 실습 기록 — README 3.2 Nexus 설치 (2026-09-28)

- `nexus-0` 1/1 Running on **devops-worker** ✅, initContainer `fix-permissions` Completed(exit 0), `/data/nexus` = `200:200` ✅ (6.4 두 겹 대비).
- `curl http://nexus.example.com/` → **200** — 7.2 에서 404 였던 같은 요청이 처음으로 끝(Pod)까지 도달 (hosts → 정문 → 안내 데스크 → Ingress → Service → Pod).
- admin 비밀번호 변경 확인: `admin:admin123` 으로 API 호출 → **401** ✅ (새 비밀번호는 쓰지 않고 옛 값이 막혔는지만 확인).
- 노드에서 `localhost:30082`·`30083` → **000 (연결 안 됨)**. NodePort Service 는 있지만 Nexus 안에서 그 포트를 듣는 **Docker 커넥터가 아직 없다**
  (README 3.3 "포트 8082/8083 은 커넥터를 만들어야 열린다"). = 지금 앱 이미지를 pull 하면 노드 쪽 연결 오류 → 6.5 재확인 문제 2 의 상황.
- 기본 리포지토리: maven-central(proxy)·maven-public(group)·maven-releases/snapshots·nuget-* 가 이미 있다. 이 프로젝트의 CI 는 maven/npm 캐시를 쓰지 않으므로
  (Jenkinsfile·Dockerfile 에 Nexus npm/maven 설정 없음) README 3.3 의 maven/npm 줄은 **선택**.

### 학습 노트: 실습 기록 — README 3.3 레지스트리 구성 (2026-09-28)

- UI 로 Docker 리포지토리 5개 생성: `docker-hosted`(8082) · `docker-hub` · `gcr-proxy` · `quay-proxy` · `docker-group`(8083) ✅. maven/npm 은 생략(선택).
- 노드에서 `localhost:30082/v2/`·`30083/v2/` → **000 → 401** 로 바뀜. 000 = 문 뒤에 아무도 없음, 401 = **누군가 있고 "인증부터"** 라고 답함
  (Docker Registry v2 규약: `/v2/` 는 토큰을 받으러 오라는 401 로 시작 — group 의 익명 pull 도 이 401 → 익명 토큰 → pull 순서로 진행).
- 노드 containerd 로 README 3.3 의 익명 pull 3개(`alpine`, `kaniko executor -debug`, `argocd v3.5.2`) **모두 성공** —
  미러(`nexus-docker-group.example.com` → `localhost:30083`, 6.5) → docker-group → 각 proxy(Docker Hub / gcr.io / quay.io) 경로가 자격증명 없이 동작.
- 확인 못 한 것: group 멤버 **순서**는 관리자 API 라 익명으로 볼 수 없다(403). argocd pull 성공은 순서를 증명하지 못한다
  (Docker Hub 에 v3.5.2 가 없으면 다음 멤버로 넘어가므로). → UI 에서 눈으로 확인.
  hosted 의 익명 pull 차단은 첫 앱 이미지가 올라간 뒤(7.8) 확인 — 단 group 경유로는 익명 pull 이 **된다**(5.5 의 빈틈).

### 학습 노트: 실습 기록 — README 3.5 regcred (2026-09-28)

- `read -rp`/`read -rsp` 로 받아 4개 네임스페이스(`react-app-dev/prod`, `python-api-dev/prod`)에 `regcred`(`kubernetes.io/dockerconfigjson`) 생성 ✅.
  네임스페이스를 **Argo CD 보다 먼저** 만들어 둔 셈 — Argo CD 의 `CreateNamespace=true` 는 이미 있으면 그냥 넘어간다.
- 검증(값 비출력): 각 Secret 의 `auth` 로 `localhost:30082/v2/` 호출 → **4개 모두 200**, 자격증명 없이 → **401** ✅.
  server 키가 `nexus-docker.example.com` — 이미지 이름의 호스트와 **글자 그대로 같아야** kubelet 이 이 출입증을 꺼내 쓴다(미러로 가도 매칭은 이름 기준, 6.5).
- 남은 빈틈(8장): `argocd` 네임스페이스엔 `regcred` 가 없다 — Image Updater 가 참조(`bootstrap/image-updater/config.yaml:22`).
- **7.4 완료.**

## 7.5 4장 Argo CD 설치 + Application 등록

**무엇을 왜.** GitOps 의 "CD" 담당이다. gitops 리포지토리를 읽어 클러스터를 그 내용과 똑같이 맞춘다. 이 절에서 **gitops 리포지토리를 처음 만든다.**

**따라 할 곳.** `README.md:1060` (설치), `README.md:1130` (로그인), `README.md:1156` (CI 토큰), `README.md:1175` (Application 등록)

단계는 다음과 같다.

1. 설치 — `--server-side` 필수. 빼면 ApplicationSet CRD 가 annotation 크기 한도를 넘어 만들어지지 않는다.
2. admin 로그인과 비밀번호 변경.
3. CI 전용 계정 `cicd` 의 토큰 발급. 이 계정은 설정에 이미 정의돼 있다.

   `bootstrap/argocd/configs/argocd-cm.yaml:17`

   ```yaml
   accounts.cicd: apiKey, login
   ```

4. GitLab 에 `gitops-manifests` 프로젝트를 만들고 이 리포지토리의 `apps/`, `manifests/`, `pipelines/` 를 **루트에** 올린다.
5. Deploy token 을 발급해 Argo CD 에 리포지토리 자격증명으로 등록한다.
6. `apps/app-of-apps.yaml` **하나만** apply 한다. 나머지 Application 은 Argo CD 가 gitops 리포지토리에서 읽어 스스로 만든다(App of Apps).

**확인.**

```bash
argocd repo list --grpc-web --refresh hard     # STATUS Successful
argocd app list --grpc-web
```

이 시점의 기대 상태는 README 표(`README.md:1240`)와 비교한다. **Degraded·Missing 이 섞여 있는 것이 정상이다.**

| Application | 기대 상태 | 이유 |
|---|---|---|
| `bootstrap` | Synced / Healthy | |
| `react-app-dev` | Synced / Degraded | 이미지 태그 `dev-0000000` 이 아직 없다. 7장 첫 빌드에서 풀린다 |
| `react-app-prod` | OutOfSync / Missing | 수동 동기화 대상이라 아직 아무것도 안 만들었다 |

봐야 할 것은 **`ComparisonError` 가 없느냐**다. 있으면 Git 을 못 읽고 있는 것이다.

**자주 막히는 곳.**

- `argocd login` 은 평문 HTTP 라 `argocd.example.com:80 --grpc-web --plaintext --skip-test-tls` 네 가지가 모두 필요하다. 하나라도 빠지면 `context deadline exceeded` 로 멈춘다(`README.md:1143`).
- gitops 프로젝트를 만들 때 *Initialize repository with a README* 를 켜면 첫 push 가 거부된다.
- Deploy token 의 사용자명은 `root` 가 아니라 `gitlab+deploy-token-<N>` 이다.
- 한 번 틀린 자격증명을 고쳐도 `repo list` 는 캐시 때문에 계속 `Failed` 로 보인다. `--refresh hard` 를 붙인다.

### 학습 노트: 실습 기록 — README 4.1·4.3·4.4 Argo CD 설치·로그인·cicd 토큰 (2026-09-28)

- 7개 컴포넌트 모두 Ready — 전부 **devops-worker2** 에 떴다(PV 가 없는 상태 없는 워크로드라 스토리지 노드에 묶이지 않음, 6.4).
- CRD 3개(`applications`, `applicationsets`, `appprojects`) ✅ — `--server-side` 덕에 ApplicationSet CRD 도 생성.
- `argocd-initial-admin-secret` 삭제됨(NotFound) ✅, UI `http://argocd.example.com/` → 200, CLI v3.5.2.
- 3장 설정이 실제로 반영됐는지:
  - `argocd-cmd-params-cm`: `server.insecure=true`, **`timeout.reconciliation=180s`** ✅ — 폴링 주기는 `argocd-cm` 이 아니라 **cmd-params-cm** 에 있다
    (확인 요청 때 "argocd-cm 의 timeout.reconciliation" 이라고 잘못 말함 — 정정. `configs/argocd-cmd-params-cm.yaml:14`).
  - `argocd-cm`: `accounts.cicd: apiKey,login`, `url: http://argocd.example.com` ✅.
  - `argocd-rbac-cm`: `g, cicd, role:ci` → `role:ci` = applications **get·sync 만** (3.3 최소 권한) ✅.
- `~/cicd-argocd-token.txt`: 600, 236 bytes(JWT). 그 토큰으로 `get-user-info` → `Username: cicd` ✅. 만료 없는 토큰 — 노출 시 delete-token 후 재발급.
- AppProject 확인: `apps/project.yaml` 의 destinations 에 `dev-*` 패턴 외에 `react-app-dev`·`python-api-dev`·`postgres-dev`(prod 도 같음)가 **명시돼 있다**
  → 3.3 에서 걱정한 "패턴이 실제 네임스페이스와 안 맞아 sync 거부" 는 일어나지 않는다.

### 학습 노트: 실습 기록 — README 4.5-1·4.5-2 gitops 리포지토리와 리포지토리 자격증명 (2026-09-28)

- GitLab `my-group/gitops-manifests` (private), `main` 첫 커밋 `112be8f` — 최상위 `apps manifests pipelines` ✅. 로컬 clone 과 `origin/main` 일치.
- `main` 은 **보호 브랜치**(프로젝트 생성 시 기본) — 그래서 Jenkins 의 `ci-bot` 은 Developer 가 아니라 **Maintainer** 여야 push 된다(README 2.4, 7.3 에서 40 확인).
- Deploy token `argocd` → username `gitlab+deploy-token-1`, 스코프 **`read_repository` 만**, 만료 없음 ✅.
  역할 분리: Argo CD = 읽기 전용 Deploy token(프로젝트 1개), Jenkins = 읽기·쓰기 Group token(그룹 전체) — 3.1·4.6.
- Secret `repo-gitlab-https` 에 라벨 `argocd.argoproj.io/secret-type: repository` → Argo CD 가 이 라벨로 자격증명을 찾는다.
  `argocd repo list --refresh hard` → **Successful** ✅ (repo-server Pod 가 CoreDNS rewrite 로 `gitlab.example.com` 에 닿음, 6.3).

### 학습 노트: 실습 기록 — README 4.5-4 App of Apps 등록 (2026-09-28)

- `kubectl apply -f apps/app-of-apps.yaml` 한 번 → `bootstrap` 이 gitops 리포의 `apps/` 를 읽어 AppProject 2개 + Application 6개를 만들었다 (3.4).
  finalizer 경고는 k8s 1.34 의 권고일 뿐 무해(README).
- 결과가 README 예상 표와 일치:
  - `*-dev` 3개: Synced / **Progressing** — 자동 sync 로 배포는 됐지만 Pod 가 아직 Ready 가 아님. ImagePullBackOff 는 곧바로 Degraded 가 아니라
    Deployment 의 `progressDeadlineSeconds`(기본 600초)가 지나기 전까지 **Progressing** 이다(3.7 에서 본 "ImagePullBackOff = Progressing").
  - `*-prod` 3개: OutOfSync / Missing — `automated` 가 없어 아무도 sync 하지 않았다 (3.2).
- **예상 밖: `postgres-prod` 가 처음 1~2분 `Unknown / InvalidSpecError`** → 90초쯤 뒤 스스로 `OutOfSync / Missing` 으로 바뀜(이벤트로 확인).
  추정 원인: `apps/` 에 sync-wave 가 없어 AppProject `prod` 와 Application 들이 **같은 순간에** 만들어졌고, 컨트롤러가 그 앱을 처음 검사할 때
  프로젝트를 아직 못 봤던 것(프로젝트가 없으면 spec 이 무효). 다음 재조정에서 풀렸다. 재발을 막으려면 AppProject 에
  `argocd.argoproj.io/sync-wave: "-1"` 을 붙여 먼저 만들게 할 수 있다(제안만, 미적용).

### 학습 노트: 이미지 pull 주체를 눈으로 확인 — react-app-dev 의 Pod 이벤트 (2026-09-28)

6.5 에서 틀렸던 "이미지를 받는 주체" 를 실제 이벤트로 확인했다.

| 이벤트 | From | 의미 |
|---|---|---|
| `Scheduled ... to devops-worker2` | **default-scheduler** | 스케줄러가 노드를 정함 (배치도 → 자리 배정) |
| `Pulling image "nexus-docker.example.com/my-group/react-app:dev-0000000"` | **kubelet** | 그 노드의 kubelet 이 containerd 에 pull 지시 (트럭 기사 출발) |
| `Failed ... NotFound ... not found` → `ErrImagePull` → `ImagePullBackOff` | **kubelet** | 가져오지 못함 → 간격을 늘려 재시도(x17) |

- **Argo CD 는 이벤트에 한 번도 등장하지 않는다.** Argo CD 는 YAML 을 API 서버에 넣고 끝(Synced). 그 뒤는 scheduler → kubelet → containerd 의 일.
- 에러 종류로 **어디까지 갔는지** 판별 (5.6·6.5 종합):

| 에러 | 뜻 | 볼 곳 |
|---|---|---|
| `connection refused` / timeout | 창고까지 못 감 | 미러(`kind-config.yaml`)·`nexus/nodeport.yaml`·Nexus 커넥터 (6.5) |
| `401 Unauthorized` | 창고엔 갔는데 출입증 거부 | `regcred` (5.6) |
| **`NotFound`** ← 이번 | 창고에 들어가 찾아봤는데 **그 상자가 없음** | 태그 — `dev-0000000` 은 자리표시자, 아직 CI 가 이미지를 안 올림 (5.7) |

  → 이번 NotFound 는 **미러·Nexus 커넥터·regcred 가 모두 정상** 이라는 증거이기도 하다. 첫 Jenkins 빌드가 `develop-<sha8>` 태그를 커밋하면 풀린다(7.8).
- **7.5 완료.**

## 7.6 5장 Jenkins 설치

**무엇을 왜.** CI 담당이다. 설치 마법사 없이 **JCasC**(설정을 코드로 적는 방식)로 뜨므로, 관리자 계정·자격증명·에이전트 Pod 템플릿이 `bootstrap/jenkins/casc.yaml` 에 이미 다 들어 있다. 우리가 할 일은 **비밀 값 Secret 을 넣어 주는 것**뿐이다.

**따라 할 곳.** `README.md:1286` (시크릿), `README.md:1329` (설치), `README.md:1366` (에이전트)

컨트롤러는 빌드를 직접 돌리지 않는다. 빌드마다 에이전트 Pod 를 새로 띄운다.

`bootstrap/jenkins/casc.yaml:58`

```yaml
      numExecutors: 0
```

`jenkins-secrets` 에는 앞 단계에서 모은 값이 한꺼번에 들어간다. 여기서 **2장·3장·4장의 결과물이 처음 한데 모인다.**

| Secret 키 | 어디서 왔나 |
|---|---|
| `GITOPS_USER` / `GITOPS_TOKEN` | 2.4 의 그룹 토큰 `ci-bot` |
| `NEXUS_USER` / `NEXUS_PASSWORD` | 3장 Nexus 계정 |
| `ARGOCD_AUTH_TOKEN` | 4.4 의 `cicd` 토큰 |

**확인.**

```bash
kubectl -n jenkins get secret jenkins-secrets                                # DATA 7
kubectl -n jenkins get pods                                                  # jenkins-0 1/1
curl -sS -o /dev/null -w '%{http_code}\n' http://jenkins.example.com/login   # 200
```

UI 에서 **Manage Jenkins > Credentials** 에 자격증명 3개, **Clouds > kubernetes** 에 Pod 템플릿 `build` 가 보이면 된다. 첫 빌드 전에 두 워커 노드에 에이전트 이미지를 미리 받아 두면(`README.md:1390`) 이미지 경로 문제를 빌드 전에 잡을 수 있다.

**자주 막히는 곳.**

- `/data/jenkins` 소유자가 1000 이 아니면 `Permission denied` 로 CrashLoopBackOff.
- 토큰 파일 끝의 줄바꿈이 섞이면 인증이 401 로 실패한다. README 명령이 `tr -d '\n'` 을 쓰는 이유다.
- Secret 값을 나중에 바꾸면 `rollout restart sts/jenkins` 를 해야 반영된다. 환경 변수는 파드가 뜰 때만 읽힌다.
- 플러그인은 인터넷에서 받는다. `install-plugins` 가 이름 해석 실패로 죽으면 1.1 의 DNS 고정을 확인한다.

### 학습 노트: 실습 기록 — README 5.1 사전 조건·5.2 시크릿 (2026-09-29)

- 5.1 사전 조건은 모두 이미 충족: `/data/jenkins` 소유자 `1000:1000`(1.1 에서 만듦), hosts 는 Windows·WSL 양쪽에 있음, WSL 여유 메모리 약 14GB. 설치 전 `jenkins.example.com` 은 `404` — Ingress 까지는 오지만 그 이름의 Ingress 가 아직 없는 상태(1장의 Nexus 404 와 같은 상황).
- `jenkins-secrets` 생성 → `DATA 7`. Nexus push 계정은 로컬 학습용이라 7.4 의 `admin` 을 그대로 썼다(운영이면 push 전용 계정을 따로 만든다 — `ci-bot` 과 같은 최소 권한 원칙).
- **여러 줄 명령을 붙여 넣으면 터미널 화면이 깨져 보일 수 있다**(`-from-li>` 처럼 겹쳐 찍힘). 화면만 보고 판단하지 말고, 값은 출력하지 않고 **길이·줄바꿈·파일과 같은지**만 확인한다:

  ```bash
  g() { kubectl -n jenkins get secret jenkins-secrets -o jsonpath="{.data.$1}" | base64 -d; }
  for k in JENKINS_ADMIN_ID JENKINS_ADMIN_PASSWORD GITOPS_USER GITOPS_TOKEN ARGOCD_AUTH_TOKEN NEXUS_USER NEXUS_PASSWORD; do v="$(g $k)"; printf '%-24s len=%-4s newline=%s\n' "$k" "${#v}" "$(printf '%s' "$v" | tr -cd '\n' | wc -c)"; done
  [ "$(g GITOPS_TOKEN)" = "$(tr -d '\n' < ~/gitlab-group-token.txt)" ] && echo "GITOPS_TOKEN == file"
  ```

  결과: 7개 모두 `len>0`, `newline=0`, 두 토큰 모두 파일과 일치. 비밀 값 확인은 본인이 한다(Claude 가 디코딩하려던 명령은 권한 정책에 막혔다 — 옳은 동작).
- 이 확인 명령도 한꺼번에 붙여 넣다가 마지막 `curl` 줄의 URL 이 잘렸다(`curl: (2) no URL specified`). 긴 명령은 한 줄씩 붙여 넣는다.

### 학습 노트: 실습 기록 — README 5.3 설치 (2026-09-29)

- `kubectl kustomize bootstrap/jenkins | kubectl apply -f -` → SA 2개(`jenkins`, `jenkins-agent`), Role/RoleBinding, ConfigMap 2개(`jenkins-casc`, `jenkins-plugins`), Service 2개(UI 8080 / 에이전트 50000), PV/PVC, StatefulSet, Ingress 생성. `namespace/jenkins configured` 는 5.2 에서 이미 만들어서다.
- 파드 기동: `Init:1/2`(30s) → `PodInitializing`(60s) → `Running 0/1` → `1/1`(83s). 약 1분 반 만에 끝. `jenkins.example.com/login` 이 `404` → `200`.
- init 컨테이너 순서: `fix-permissions`(root, 소유자 1000 으로 맞춤) → `install-plugins`(인터넷에서 플러그인 받음) → `jenkins`(envFrom 으로 Secret 읽고 JCasC 적용). `plugins.txt` 변경에 재시작이 필요한 이유가 이것 — 플러그인은 init 컨테이너가 받으므로 파드가 새로 떠야 다시 받는다.
- `logs -f ... -c install-plugins` 는 아무것도 출력하지 않고 끝났지만 플러그인 디렉터리에 `.jpi` 91개가 있어 다운로드는 성공했다. **로그가 비었다고 실패로 보지 말고 결과물을 확인한다.**
- 컨트롤러 로그: `Jenkins is fully up and running`. worker(`devops-worker`)에 떴고 기동 직후 메모리 약 470Mi(요청 1Gi).
- 로그에 `entry for admin ... duplicates one seen earlier for admin` WARNING 이 한 줄 있었다. 로그인이 되면 동작에는 영향 없다(아래 로그인 확인에서 판단).
- UI: `admin` 로그인 성공. **Credentials** 에 `gitops-repo` / `argocd-auth-token` / `nexus-registry` 3개, **Clouds > kubernetes** 에 Pod 템플릿 `build`. → 위 WARNING 은 동작에 영향 없음.
- Secret 키 7개가 자격증명 3개가 되는 이유: `JENKINS_ADMIN_*` 2개는 로그인 계정, 나머지는 user+token / 토큰 하나 / user+password 로 묶인다.

### 학습 노트: 실습 기록 — README 5.4 에이전트 이미지 미리 받기 (2026-09-29)

- 두 worker × 이미지 5개(`alpine/k8s`, `node`, `python`, `kaniko`, `argocd`)를 `crictl pull nexus-docker-group.example.com/...` → **10/10 OK**. 노드 containerd → 미러 설정 → Nexus `docker-group` 익명 pull 경로가 에이전트 이미지 전부에 대해 열려 있다.
- 에이전트 Pod 는 nodeSelector 가 없어 어느 worker 에나 뜨므로 **두 worker 모두** 받아 둔다. 첫 빌드에서 `ImagePullBackOff` 가 나면 Jenkins 가 아니라 이 경로(노드 kubelet)를 먼저 의심한다.

### 학습 노트: 확인 문제 Q&A (2026-09-29) — 1/3

1. **Secret 값을 바꾼 뒤 필요한 것** — 답: `rollout restart sts/jenkins`. (고른 답: JCasC Reload — 오답)
   Reload 는 **`casc.yaml` 파일을 다시 읽는 것**이지 환경 변수를 다시 읽는 것이 아니다. `${NEXUS_PASSWORD}` 는 이미 떠 있는 프로세스의 환경 변수에서 치환되는데, `envFrom` 환경 변수는 **컨테이너가 시작될 때 한 번** 채워지고 바뀌지 않는다. 그래서 Reload 하면 **옛 값**으로 다시 치환될 뿐이다.

   | 바꾼 것 | 반영 방법 | 이유 |
   |---|---|---|
   | `casc.yaml` (ConfigMap) | apply → 1분 대기 → Reload | 파일은 kubelet 이 파드 안에서 갱신 |
   | `jenkins-secrets` (envFrom) | `rollout restart` | 환경 변수는 시작 때만 채워짐 |
   | `plugins.txt` | `rollout restart` | init 컨테이너가 파드 기동 때만 받음 |

2. **kaniko 컨테이너 `ImagePullBackOff` 의 주체** — 답: 노드 kubelet/containerd. (정답 ✅ — 6.5 부터 이어진 "이미지는 누가 받나" 오개념이 해소됨)
3. **`install-plugins` 의 `Temporary failure in name resolution`** — 답: WSL DNS 고정(1.1). (고른 답: `/data/jenkins` UID — 오답)
   **에러 문구가 이미 원인 종류를 말해 준다.** "name resolution" = 이름 → IP 를 못 바꿈 = DNS. UID 가 틀렸다면 `Permission denied` 가 나오고, 컨테이너도 `install-plugins` 가 아니라 파일을 쓰는 쪽에서 실패한다. 플러그인은 Nexus 를 거치지 않고 `updates.jenkins.io`(인터넷)에서 받으므로, WSL 재시작 뒤 DNS 가 풀리면 여기서 가장 먼저 깨진다.
   → 진단 순서: **① 어느 컨테이너가(`-c install-plugins`) ② 무슨 문구로(`name resolution` / `Permission denied` / `401`) 실패했나** 를 먼저 읽고, 그 문구에 맞는 원인으로 간다.

### 학습 노트: 재확인 0/2 → 실제 파드로 다시 보기 (2026-09-29)

재확인에서 두 문제 모두 **직전 문제의 정답**(restart, DNS)을 골랐다. 규칙이 아니라 답을 기억한 것이라, 살아 있는 `jenkins-0` 안을 직접 들여다봤다.

```text
/var/jenkins_home/casc.d/jenkins.yaml -> ..data/jenkins.yaml     # ConfigMap 이 "파일"로 마운트됨
45:  image: "nexus-docker-group.example.com/library/node:22-alpine"   # 값이 파일에 그대로 적힘
78:  username: "${GITOPS_USER}"                                       # 파일엔 자리표시자만 있음
printenv GITOPS_USER → ci-bot                                          # 실제 값은 프로세스 환경 변수
컨테이너 시작: 2026-09-28T20:25:23Z                                     # 환경 변수는 이 순간 채워지고 고정
```

- **이미지 태그(45행)** 는 파일에 값이 직접 적혀 있다 → 파일만 바뀌면 되므로 apply 후 **Reload** 로 충분하다. restart 해도 되지만 **필요하지는 않다**.
- **자격증명(78행)** 은 파일에 `${...}` 만 있고 값은 환경 변수에서 온다 → Reload 는 같은(옛) 환경 변수로 다시 채운다 → 값을 바꾸려면 **restart** 로 환경 변수를 새로 채워야 한다.
- 한 줄 규칙: **"바꾼 값이 파일에 있나, 환경 변수에 있나?"** 파일이면 Reload, 환경 변수면 restart.
- 에러 문구 → 원인 표 (문구를 보고 고른다, 직전 답을 고르지 않는다):

  | 로그 문구 | 뜻 | 볼 곳 |
  |---|---|---|
  | `name resolution`, `UnknownHost` | 이름 → IP 실패 | DNS (WSL 고정 1.1, CoreDNS 1.6) |
  | `Permission denied`, `AccessDeniedException` | 파일 소유자·권한 | `/data/<구성요소>` UID |
  | `401`, `Unauthorized` | 인증 실패 | 토큰·비밀번호 (줄바꿈 섞임, 만료) |
  | `ImagePullBackOff` | 노드가 이미지 못 받음 | 노드 containerd 미러·Nexus |

## 7.7 6장 GitLab webhook 연결

**무엇을 왜.** Argo CD 는 기본으로 180초마다 Git 을 확인한다(폴링). webhook 을 달면 gitops 리포지토리에 push 되는 즉시 GitLab 이 Argo CD 에 알려 준다. **없어도 동작은 한다.** 배포가 최대 3분 늦어질 뿐이다.

`bootstrap/argocd/configs/argocd-cmd-params-cm.yaml:14`

```yaml
  timeout.reconciliation: 180s
```

**따라 할 곳.** `README.md:1400`

webhook 시크릿은 "이 요청이 진짜 GitLab 에서 왔는가"를 확인하는 암호다. 같은 값을 **Argo CD 의 `argocd-secret`** 과 **GitLab webhook 설정** 양쪽에 넣는다.

**확인.** GitLab 의 webhook 화면에서 **Test > Push events** → `Hook executed successfully: HTTP 200`.

```bash
kubectl -n argocd logs deploy/argocd-server --since=2m | grep -i webhook
```

**자주 막히는 곳.**

- GitLab 은 기본으로 내부망 주소로 webhook 을 보내지 못하게 막는다. **Admin Area > Settings > Network > Outbound requests** 에서 먼저 허용한다(`README.md:1423`).
- 시크릿 파일이 비어 있는 채로 patch 하면 Argo CD 가 서명 검증 없이 모든 요청을 받는다. README 명령의 `test -s` 가 이것을 막는다.

### 학습 노트: 실습 기록 — README 6장-1 webhook 시크릿 (2026-09-29)

- 처음 조회 결과는 **빈 줄** — `argocd-secret` 에 `webhook.gitlab.secret` 키가 아직 없었다(= 서명 검증 없이 받는 상태).
- `openssl rand -hex 20` → `~/argocd-webhook-secret.txt`(umask 077 → 권한 600) → `test -s` 통과 후 patch → `argocd-server` 재시작 → 파일과 Secret 값이 같음 `OK` (값은 출력하지 않음).
- 이 값은 **세 곳**에 같은 내용으로 있어야 한다: 파일 · `argocd-secret` · GitLab webhook. 바꿀 때는 셋을 함께 바꾼다.

### 학습 노트: 실습 기록 — README 6장-2·3 GitLab 등록과 확인 (2026-09-29)

- Admin Area > Settings > Network > Outbound requests 에서 내부망 허용 → `gitops-manifests` 에 webhook 등록(URL `http://argocd-server.argocd.svc.cluster.local/api/webhook`, Push events).
- GitLab 파드 → **클러스터 안 Service DNS** 로 직접 호출한다. Ingress·hosts·CoreDNS rewrite 를 거치지 않는다.
- **첫 Test 는 `HTTP 400 Webhook processing failed`.** argocd-server 로그:

  ```text
  GitLab webhook token verification failed
  Webhook processing failed: X-Gitlab-Token validation failed
  ```

  요청은 Argo CD 까지 **도착했고**, 서명 검사에서 거절됐다(= 1단계 방어가 동작). 원인: `clip.exe` 로 시크릿을 복사한 **뒤** URL 을 채팅에서 복사해 붙여 클립보드가 덮어써진 것으로 보인다(안내 순서 문제). → 시크릿을 다시 `clip.exe` 로 복사, Edit 에서 Secret token 만 다시 붙여 넣고 저장 → **`HTTP 200`**.
- **200 이후 실제 refresh 확인**(3.6: repoURL 이 다르면 200 인데 refresh 없음):

  ```text
  argocd-server:  Requested app 'react-app-dev' refresh   (7개 앱 모두, 21:09:18)
  controller:     Refreshing app status (normal refresh requested), level (3)
  ```

  7개 Application 의 `repoURL` 이 모두 `http://gitlab.example.com/my-group/gitops-manifests.git` 이고 GitLab `external_url` 과 같아서 전부 refresh 됐다.
- 로그의 `GHCR webhook secret is not configured` 경고는 GitHub 컨테이너 레지스트리용이라 무관하다.
- 확인 문제 3/3: ① 400 + `X-Gitlab-Token validation failed` = 도착은 했고 시크릿 불일치 ② webhook 이 없어져도 폴링(최대 약 180초)으로 배포는 된다 ③ GitLab → argocd-server 는 클러스터 내부 Service DNS 로 직접(Ingress 안 거침).
- 곁가지 발견: `postgres-dev` 가 Progressing — `dev-postgres-0` Pending, PVC `dev-postgres-data` 28시간째 Pending. postgres 는 8장(응용) 범위라 그때 본다.

## 7.8 7장 react-app 파이프라인: develop push → dev 배포 확인

**무엇을 왜.** 드디어 앱을 올린다. 지금까지 만든 모든 조각이 한 번에 동작하는지 보는 단계다.

**따라 할 곳.** `README.md:1472` (로컬 확인), `README.md:1497` (GitLab push), `README.md:1523` (Jenkins 잡), `README.md:1544` (dev 확인)

1. **로컬에서 먼저 확인한다.** 이미지가 클러스터와 같은 조건(읽기 전용 루트, UID 10001)으로 뜨는지 WSL 의 docker 로 본다. CI 에서 실패하면 원인 찾기가 훨씬 오래 걸린다.
2. GitLab 에 `react-app` 프로젝트를 만들고 `main`, `develop` 두 브랜치를 push 한다.
3. Jenkins 에 **Multibranch Pipeline** 잡을 만든다. 저장하면 브랜치 스캔이 돌고 두 브랜치에 빌드가 걸린다.

그다음 무슨 일이 일어나는지는 0장에서 본 흐름 그대로다.

```
develop 빌드: 준비 → 테스트·빌드 → 이미지 푸시(Nexus) → gitops 태그 갱신 → 배포 대기
                                                           │
                             gitops-manifests 에 커밋: chore(dev): react-app -> develop-xxxxxxxx
                                                           │
                             Argo CD 가 감지(webhook) → react-app-dev 자동 동기화
```

**확인.** 콘솔 로그 마지막 줄에 `배포 완료: dev → http://react-app.dev.example.com (version: develop-xxxxxxxx)`.

```bash
argocd app get react-app-dev --grpc-web | grep -E 'Sync Status|Health Status'   # Synced / Healthy
kubectl -n react-app-dev get pods
curl -s http://react-app.dev.example.com/healthz                                 # ok
```

브라우저에서 `environment: dev`, `version: develop-<sha>` 가 보이면 성공이다. 4장에서 Degraded 였던 `react-app-dev` 가 이제 Healthy 로 바뀐 것도 확인한다.

**자주 막히는 곳.** README 의 표(`README.md:1602`)가 잘 정리돼 있다. 입문자가 특히 많이 막히는 것:

- **push 해도 빌드가 안 걸린다.** 자동 트리거가 없다. **Scan Multibranch Pipeline Now** 를 누른다.
- 에이전트 Pod 가 `ImagePullBackOff` → Nexus `gcr-proxy`/`quay-proxy` 설정(3.3).
- `main` 빌드가 "prod 승인"에서 멈춰 있다 → 정상이다. 다음 단계에서 승인한다.

### 학습 노트: 실습 기록 — README 7.1 로컬 확인 (2026-09-29)

- `~/workspace/react-app` 에 `samples/react-app` + `ci/Jenkinsfile` 복사, `git init -b main`.
- `node:22-alpine` 컨테이너로 `npm install && npm test` → 98 패키지, vitest `1 passed`, `package-lock.json` 생성(CI 의 `npm ci` 에 필수).
- `docker build --build-arg REGISTRY=docker.io` → 16 스텝 성공. 멀티 스테이지(5.1)가 로그에 그대로 보인다: `FROM .../node:22-alpine AS build` → `npm ci` → `npm run build`(vite, `dist/`) → `FROM .../nginx-unprivileged:1.28-alpine` → **`COPY --from=build /app/dist`** 만 넘어옴 → `USER 10001`.
- `--read-only --tmpfs /tmp -u 10001 -e APP_ENV=local` 로 실행 → `healthz` `ok`, `config.js` `window.APP_CONFIG = { env: "local" }`, 로그에 `Read-only file system` 없음. 클러스터의 securityContext 조건을 로컬에서 먼저 통과했다.
- 로그에 `/config.js` 요청만 찍히고 `/healthz` 는 없다 — `nginx.conf:33` 의 `access_log off`. kubelet 프로브 로그로 도배되지 않게 한 것이다.

### 학습 노트: 실습 기록 — README 7.2 GitLab 프로젝트와 push (2026-09-29)

- `my-group/react-app`(Private, README 초기화 끔) 생성. 초기화하면 원격에 커밋이 먼저 생겨 첫 push 가 거부된다.
- `ci-bot` 그룹 토큰: 앱 리포지토리 `git-upload-pack`(읽기) **200**, gitops `git-receive-pack`(쓰기) **200**. Jenkins 가 쓸 권한 두 가지를 빌드 전에 확인.
- `git push -u origin main`, `git push origin main:develop` 성공 — 두 브랜치가 **같은 커밋**을 가리킨다. 이미지 태그는 `<브랜치>-<sha8>` 이라 `main-xxxxxxxx` / `develop-xxxxxxxx` 로 SHA 부분만 같게 나온다.

### 학습 노트: Q&A — Freestyle project 와 Multibranch Pipeline 의 차이 (2026-09-29)

| | Freestyle project | Pipeline | **Multibranch Pipeline** (우리가 쓰는 것) |
|---|---|---|---|
| 빌드 절차가 사는 곳 | Jenkins UI 설정 화면(체크박스·입력칸) | Jenkinsfile (UI 에 붙여 넣거나 리포지토리에서 1개 브랜치) | **리포지토리의 Jenkinsfile** |
| 브랜치 | 잡 1개 = 브랜치 1개(보통 고정) | 잡 1개 = 브랜치 1개 | 브랜치를 **스캔해서 자동으로 하위 잡 생성** (`react-app » develop`, `react-app » main`) |
| 단계(stage)·승인(input)·여러 컨테이너 | 어렵다(플러그인·셸 스크립트로 흉내) | 가능 | 가능 |
| 변경 이력 | Jenkins 안에만(누가 언제 바꿨는지 추적 어려움) | Jenkinsfile 을 리포지토리에 두면 Git 이력 | Git 이력 — 코드 리뷰 대상 |
| 브랜치별 다른 동작 | 잡을 따로 만든다 | 잡을 따로 만든다 | 한 Jenkinsfile 안에서 브랜치 이름으로 분기 (`ci/Jenkinsfile:58` 의 if → `DEPLOY_ENV`, 각 stage 의 `when { expression ... }`) |

- 비유: Freestyle 은 **요리사에게 말로 순서를 알려 주는 것**(가게에만 남음), Multibranch 는 **레시피 카드를 재료 상자(리포지토리)에 넣어 두는 것** — 상자(브랜치)가 늘면 Jenkins 가 알아서 상자마다 요리를 한다.
- 이 프로젝트가 Multibranch 여야 하는 이유: ① `develop`→dev / `main`→prod 규칙이 `ci/Jenkinsfile:58` 의 if 블록에 있다(4.6) — 브랜치마다 같은 파일이 돌고 브랜치 이름으로 갈린다 ② 태그가 `env.BRANCH_NAME` 에서 나온다 — Multibranch 가 채워 주는 변수다(`:47` 에 Multibranch 가 아닐 때를 위한 `GIT_BRANCH` 대비책이 있지만 기본 전제는 Multibranch) ③ 에이전트 Pod(`agent { label 'build' }`)·`input` 승인은 Pipeline 문법이다 ④ 파이프라인 변경도 Git 으로 리뷰된다(GitOps 와 같은 생각).
- Freestyle 이 나은 경우: 브랜치 개념이 없는 단순 반복 작업(예: 매일 밤 백업 스크립트 한 줄 실행).

## 7.9 7.5 prod 배포 (승인 → 수동 sync)

**무엇을 왜.** 운영은 사람이 확인하고 배포한다. dev 와 무엇이 다른지 직접 체감하는 단계다.

**따라 할 곳.** `README.md:1560`

**사전 조건 두 가지**: metrics-server(1.8)와 hosts 의 `react-app.example.com`.

prod Application 에는 `automated` 가 없다. 그래서 gitops 커밋이 생겨도 Argo CD 는 "OutOfSync" 라고 표시만 하고 스스로 배포하지 않는다.

`apps/react-app.yaml:52`

```yaml
  syncPolicy:
    # 운영은 수동 승인 배포. 자동화하려면 automated 블록을 추가한다.
    syncOptions:
      - CreateNamespace=true
```

그래서 Jenkins 가 승인 후 직접 sync 를 건다(`ci/Jenkinsfile:172`). 흐름: **배포 버튼 클릭 → gitops 커밋 → `argocd app sync` → Synced + Healthy 대기.**

**확인.** 핵심은 **같은 태그(`main-xxxxxxxx`)가 모든 단계에서 보이는지**다. 태그가 달라지는 곳 바로 앞이 멈춘 단계다.

```bash
# Argo CD 가 본 커밋과 상태
kubectl -n argocd get application react-app-prod \
  -o jsonpath='{.status.sync.status} {.status.health.status} {.status.sync.revision}{"\n"}'
# 클러스터에 실제로 뜬 이미지
kubectl -n react-app-prod get deploy prod-react-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n react-app-prod get pods,hpa,pdb      # 파드 3개
curl -s http://react-app.example.com/config.js  # env: "prod"
```

**자주 막히는 곳.**

- 첫 prod 배포에서 `app wait 실패 - 일시적인 Degraded 인지 2분 더 지켜본다` 가 찍힌다 → 정상이다. 새 HPA 가 첫 지표를 받기 전 잠깐 Degraded 로 보인다.
- 같은 커밋을 다시 빌드하면 파드가 안 바뀐다 → 정상이다. 태그가 같아 gitops 커밋이 생략된다.
- 승인 대기는 60분이 지나면 빌드가 중단된다. 그때는 `main` 을 다시 빌드한다.

## 7.10 롤백 해 보기: gitops 리포지토리 커밋 revert

> **이 절은 README 에 없는 내용이다.** GitOps 의 일반적인 롤백 방법을 이 프로젝트 구조에 맞춰 정리한 것이므로, 실습 전에 무엇이 바뀌는지 한 번 더 확인하고 진행한다.

**무엇을 왜.** GitOps 에서는 "지금 떠 있어야 할 버전"이 gitops 리포지토리에 적혀 있다. 따라서 롤백 = **그 파일을 이전 태그로 되돌리는 커밋**이다. 클러스터에 `kubectl` 로 직접 손대지 않는다. dev 는 `selfHeal: true`(`apps/react-app.yaml:22`)라서, 손으로 이미지를 바꿔도 Argo CD 가 Git 내용으로 곧바로 되돌려 버린다.

**준비.** develop 에 두 번 이상 배포해서 gitops 리포지토리에 `chore(dev): ...` 커밋이 두 개 이상 있어야 한다. 앱 화면 문구를 조금 고쳐 push 하고 빌드하면 된다.

**절차 (dev 기준).**

```bash
cd ~/workspace/gitops-manifests
git pull                                   # Jenkins 가 쌓은 커밋을 먼저 받는다
git log --oneline -5 -- manifests/react-app/overlays/dev
#   b2c3d4e chore(dev): react-app -> develop-bbbbbbbb   ← 되돌릴 커밋
#   a1b2c3d chore(dev): react-app -> develop-aaaaaaaa

git revert --no-edit <되돌릴 커밋 SHA>
git show --stat HEAD                       # overlays/dev/kustomization.yaml 한 파일만 바뀌어야 한다
git push origin main
```

**확인.**

```bash
argocd app get react-app-dev --grpc-web | grep -E 'Sync Status|Health Status'
kubectl -n react-app-dev get deploy dev-react-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'   # develop-aaaaaaaa
```

브라우저의 `version` 이 이전 태그로 돌아오면 성공이다. Argo CD UI 의 **History** 에서도 배포 이력을 볼 수 있다.

**prod 라면.** 커밋을 revert 하고 push 한 뒤, prod 는 자동 동기화가 아니므로 직접 sync 한다.

```bash
argocd app sync react-app-prod --grpc-web
```

prod 프로젝트에는 `syncWindows` 가 있지만 `manualSync: true` 라서 수동 sync 는 허용된다(`apps/project.yaml:66`).

**주의할 점.**

- 롤백 커밋은 **다음 빌드가 덮어쓴다.** 새 커밋을 develop 에 push 하면 Jenkins 가 새 태그로 다시 커밋한다. 근본 수정은 앱 리포지토리에서 한다.
- 되돌릴 이미지가 Nexus 에 남아 있어야 한다. 지웠다면 파드가 `ImagePullBackOff` 가 된다.
- Argo CD UI 의 **Rollback** 버튼은 Git 을 바꾸지 않고 클러스터만 과거 상태로 돌린다. 자동 동기화가 켜진 dev 에서는 쓸 수 없고, 쓰더라도 Git 과 클러스터가 어긋나므로 이 실습에서는 git revert 방식을 쓴다.

## 7.11 10장 재시작 후 점검

**무엇을 왜.** 로컬 환경은 PC 재부팅, `wsl --shutdown` 으로 자주 멈췄다 다시 뜬다. 그때마다 어디까지 살아났는지 **아래에서 위로** 점검한다: docker → 노드 → DNS → 파드 → Ingress → Argo CD.

**따라 할 곳.** `README.md:2102` (점검 명령), `README.md:2137` (알려진 문제)

**확인.** 재시작 직후에는 **5~10분 기다린 뒤** 본다. 그 전의 `503` 은 GitLab 이 기동 중이라 정상이다.

```bash
docker ps -a --filter label=io.x-k8s.kind.cluster=devops --format 'table {{.Names}}\t{{.Status}}'   # 3개 Up
kubectl get nodes                                  # 3대 Ready
getent ahostsv4 pypi.org | head -1                 # 외부 DNS
kubectl get pods -A | grep -vE 'Running|Completed' # 비어 있어야 한다
for h in gitlab nexus argocd jenkins; do
  printf '%-8s ' "$h"; curl -sS -o /dev/null -w '%{http_code}\n' -H "Host: $h.example.com" http://localhost/
done
argocd app list --grpc-web                         # ComparisonError 없음
```

**자주 막히는 곳.**

- 노드 컨테이너 일부가 `Exited` → `docker start $(docker ps -aq --filter label=io.x-k8s.kind.cluster=devops)`. 1.2 에서 재시작 정책을 `unless-stopped` 로 바꿔 두면 대부분 예방된다.
- `argocd` 명령이 `token is expired` → admin 세션은 24시간이면 만료된다. 다시 `argocd login`.
- `argocd.example.com` 이 504 → kindnet 문제. `kubectl -n kube-system rollout restart ds/kindnet` (`README.md:2151`).

**처음부터 다시 하고 싶을 때.** `README.md:2168` 의 초기화 절차를 따른다. 클러스터, `/data`, 로컬 작업 리포지토리, 토큰 파일을 지운다. **되돌릴 수 없으므로** 필요하면 `/data` 를 먼저 백업한다. Windows hosts, WSL 설정, 설치한 CLI 는 남겨 둬도 된다.

---

## 정리

| 단계 | 끝났다는 신호 | 여기서 만든 것이 쓰이는 곳 |
|---|---|---|
| 0 사전 준비 | 도구 표 전부 통과 | 전 단계 |
| 1 클러스터 | `localhost` 가 `404`, 파드 안 DNS 가 Ingress IP | 모든 접속 |
| 2 GitLab | `302`, root 로그인, 그룹 토큰 파일 | 4장(gitops), 5장(Jenkins) |
| 3 Nexus | `200`, 익명 pull 성공, `regcred` 4개 | 5장 에이전트 이미지, 7장 push/pull |
| 4 Argo CD | `repo list` Successful, `ComparisonError` 없음, `cicd` 토큰 | 5장 Jenkins, 7장 배포 |
| 5 Jenkins | 자격증명 3개, Pod 템플릿 `build` | 7장 |
| 6 webhook | Test → HTTP 200 | 배포 지연 제거 |
| 7 파이프라인 | dev·prod 에서 같은 태그가 브라우저까지 보임 | — |

- 문제가 생기면 **태그를 따라가며** 어느 단계에서 끊겼는지 찾는다: Jenkins 로그 → gitops 커밋 → Argo CD revision → Deployment 이미지 → 브라우저.
- 롤백은 **Git 에서** 한다. 클러스터를 직접 고치지 않는다.

## 확인 문제

1. 1장 직후 `curl -H 'Host: nexus.example.com' http://localhost/` 가 `404` 를 돌려줬다. 실패인가?

<details><summary>답</summary>

실패가 아니다. ingress-nginx 까지는 도착했지만 아직 그 이름의 Ingress 가 없어서 기본 백엔드가 404 를 준 것이다. 오히려 docker 포트 매핑 → 컨트롤러까지 통로가 열렸다는 증거다. Nexus 를 설치한 뒤에는 200 이 된다.

</details>

2. CoreDNS rewrite(1.6)를 빼먹으면 7장의 어느 단계가 어떻게 실패하는가?

<details><summary>답</summary>

파드 안에서 `nexus-docker.example.com` 이 Windows hosts 를 거쳐 `127.0.0.1`(파드 자신)로 풀린다. 그래서 "이미지 빌드·푸시" 단계에서 kaniko 가 자기 자신에게 붙으려다 `connection refused` 로 실패한다. Argo CD 의 `gitlab.example.com` 접근, "배포 대기" 의 `argocd.example.com` 접근도 같은 이유로 깨진다.

</details>

3. Argo CD 에는 Deploy token 을, Jenkins 에는 Group access token 을 쓴다. 왜 하나로 통일하지 않는가?

<details><summary>답</summary>

권한을 필요한 만큼만 주기 위해서다. Argo CD 는 gitops 리포지토리를 **읽기만** 하므로 그 프로젝트 전용 `read_repository` 토큰이면 된다. Jenkins 는 앱 리포지토리를 읽고 gitops 리포지토리에 **써야** 하므로 그룹 범위의 쓰기 권한(Maintainer)이 필요하다.

</details>

4. 4장 직후 `react-app-dev` 가 Degraded, `react-app-prod` 가 OutOfSync/Missing 이다. 각각 언제, 무엇 때문에 풀리는가?

<details><summary>답</summary>

`react-app-dev` 는 overlay 의 이미지 태그 `dev-0000000` 이 Nexus 에 없어서 Degraded 다. 7장에서 Jenkins 가 첫 태그를 gitops 에 커밋하면 풀린다. `react-app-prod` 는 자동 동기화가 없어서 아직 아무것도 만들지 않은 상태다. 7.5 에서 승인 후 Jenkins 가 `argocd app sync` 를 걸면 풀린다.

</details>

5. dev 에 문제가 있는 버전이 배포됐다. `kubectl set image` 로 이전 이미지를 넣으면 어떻게 되고, 올바른 롤백 방법은 무엇인가?

<details><summary>답</summary>

dev Application 은 `selfHeal: true` 라서 Argo CD 가 곧바로 Git 에 적힌 (문제 있는) 버전으로 되돌린다. 올바른 방법은 gitops 리포지토리에서 해당 `chore(dev): ...` 커밋을 `git revert` 하고 push 하는 것이다. 그러면 Argo CD 가 이전 태그로 동기화한다. (이 절차는 README 에 없는 일반적인 GitOps 방법이다.)

</details>
