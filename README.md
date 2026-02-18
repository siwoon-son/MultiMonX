# MultiMonX

**Prometheus 표준 Pull 생태계**를 따르는 오픈소스 다중 서버 모니터링 플랫폼입니다.  
Prometheus + Grafana + Exporter 구조를 Docker Compose로 쉽게 배포할 수 있게 해줍니다.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Agent 서버들 (모니터링 대상)                          │
├─────────────────────────────────────────────────────────────────────────┤
│  node_exporter (:9100)  │  dcgm_exporter (:9400)  │  smartctl (:9633)   │
│  ipmi_exporter (:9290, optional)                                        │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │  HTTP GET /metrics (Pull)
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Central Monitoring Server (1대)                       │
├─────────────────────────────────────────────────────────────────────────┤
│  Prometheus (:9090)  →  scrape 15s, rule 평가, TSDB 저장                   │
│  Alertmanager (:9093) → 알림 라우팅, Discord/Email 등                       │
│  Grafana (:3000)     → Prometheus 데이터소스, 대시보드 자동 프로비저닝           │
└─────────────────────────────────────────────────────────────────────────┘
```

- **Pull 모델**: Exporter는 `/metrics`만 제공하고, Prometheus가 주기적으로 수집합니다.
- **단일 중앙 서버**: relay/federation 없이 한 대의 Central로 관리합니다.

---

## 5분 Quickstart

### 0. (최초 1회) 공유 네트워크 생성

Central과 Agent는 **같은 Docker 네트워크(multimonx)** 를 사용합니다. Central만 단독, Agent만 단독, 또는 둘 다 실행하는 모든 경우에 동일하게 적용됩니다.

```bash
docker network create multimonx
```

이미 `multimonx` 네트워크가 있으면 위 명령은 무시해도 됩니다. 다른 프로젝트에서 만든 네트워크와 이름이 겹치지 않도록 한 번만 생성하면 됩니다.

### 1. Central 실행 (모니터링 서버 1대)

```bash
cd central
# 최초 1회: 설정 파일 준비 (.env, prometheus.yml, alertmanager.yml)
cp .env.example .env
cp prometheus/prometheus.yml.example prometheus/prometheus.yml
cp alertmanager/alertmanager.yml.example alertmanager/alertmanager.yml

docker compose up -d
```

| 서비스      | URL (기본값, .env에서 변경 가능) |
|------------|--------------------------|
| Prometheus | http://localhost:9090    |
| Alertmanager | http://localhost:9093  |
| Grafana    | http://localhost:3000 (admin / admin, .env에서 비밀번호 변경 권장) |

### 2. Agent 실행 (모니터링할 각 서버에서)

같은 서버에서 Central과 Agent를 함께 올릴 때도, 다른 서버에서 Agent만 올릴 때도 위 0단계에서 `multimonx` 네트워크가 있어야 합니다. (해당 호스트에서 최초 1회 `docker network create multimonx` 실행)

```bash
cd agent
# 포트 변경이 필요한 경우: .env 파일 생성 및 수정
# cp .env.example .env  # 여러 서버에서 다른 포트 사용 시

docker compose up -d
```

### 3. Central에 타겟 추가

최초 1회: `central/prometheus/prometheus.yml.example`을 `prometheus.yml`로 복사한 뒤 사용합니다.
`prometheus.yml`의 `scrape_configs`에서 `targets`에 Agent 서버 IP를 추가한 뒤, Prometheus 설정 리로드 또는 컨테이너 재시작합니다.

```yaml
- job_name: "nodes"
  static_configs:
    - targets:
        - "192.168.0.11:9100"
```

자세한 단계는 [docs/quickstart.md](docs/quickstart.md)를 참고하세요.

---

## 서버 추가 방법

1. 새 서버에서 `docker network create multimonx` (최초 1회), 이 저장소의 `agent/`만 복사한 뒤 `docker compose up -d` 실행
2. Central 서버의 `central/prometheus/prometheus.yml`에 해당 타겟을 `targets`에 추가  
   (같은 Docker 네트워크면 컨테이너 이름 예: `multimonx-node-exporter:9100`, 다른 호스트면 `호스트IP:포트`)
3. `curl -X POST http://CENTRAL_IP:9090/-/reload` 또는 Prometheus 컨테이너 재시작

예시는 [examples/sample-prometheus.yml](examples/sample-prometheus.yml)와 [docs/quickstart.md](docs/quickstart.md)를 참고하세요.

---

## Alert 설정 방법

- 기본 규칙: NodeDown, CPUUsageHigh, MemoryHigh, DiskUsageHigh, GPUUtilHigh, GPUTempHigh (규칙 파일: `central/prometheus/rules/`)
- 알림 전송: `central/alertmanager/alertmanager.yml`에서 Discord webhook 또는 Email 설정

자세한 내용은 [docs/alerting.md](docs/alerting.md)를 참고하세요.

---

## GPU 서버 구성 방법

NVIDIA GPU가 있는 서버에서는 Agent를 다음처럼 실행합니다.

```bash
cd agent
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

호스트에 **NVIDIA 드라이버**와 **nvidia-container-toolkit**이 설치되어 있어야 합니다.  
Central의 `prometheus.yml`에 `gpu` job으로 `호스트:9400`을 추가하면 Grafana **GPU Detail** 대시보드에서 확인할 수 있습니다.

---

## 저장소 구조

```
multimonx/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── quickstart.md
│   ├── alerting.md
│   └── security.md
├── central/
│   ├── .env.example                 # 템플릿 → .env 복사 (둘 다 Git 미포함)
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   ├── prometheus.yml.example   # 템플릿 → prometheus.yml 복사 (Git 미포함)
│   │   └── rules/
│   │       ├── node-alerts.yml
│   │       └── gpu-alerts.yml
│   ├── alertmanager/
│   │   └── alertmanager.yml.example # 템플릿 → alertmanager.yml 복사 (Git 미포함)
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/
│       │   └── dashboards/
│       └── dashboards/
├── agent/
│   ├── docker-compose.yml
│   ├── docker-compose.gpu.yml
│   ├── docker-compose.ipmi.yml
│   └── README.md
└── examples/
    └── sample-prometheus.yml
```

---

## 보안 고려 사항

- Exporter와 Prometheus는 **내부망에서만** 접근하도록 구성하는 것을 권장합니다.
- 기본 구성은 **인증 없이** 동작합니다. 외부 노출 시 [docs/security.md](docs/security.md)의 Basic Auth, HTTPS reverse proxy 예시를 참고해 구성하세요.

---

## 라이선스

MIT License. GitHub에 공개하여 사용·수정·재배포할 수 있습니다.
