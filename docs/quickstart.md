# MultiMonX 5분 Quickstart

## 사전 요구사항

- Docker 및 Docker Compose 설치
- (GPU 사용 시) NVIDIA 드라이버 + nvidia-container-toolkit

## 0. (최초 1회) 설정 파일 준비 (템플릿 → 실제 파일)

각 디렉터리에서 `.example` 템플릿을 복사해 사용하세요.

```bash
cd central

# Prometheus: scrape 타겟/alias 등
cp prometheus/prometheus.yml.example prometheus/prometheus.yml

# Alertmanager: Discord/Email 웹후크 등 (알림 사용 시)
cp alertmanager/alertmanager.yml.example alertmanager/alertmanager.yml

# 환경 변수: 포트, 비밀번호, 보관 기간 등 시스템 설정
cp .env.example .env
# 필요 시 .env 파일 편집:
#   - PROMETHEUS_PORT, ALERTMANAGER_PORT, GRAFANA_PORT (포트 변경)
#   - GF_SECURITY_ADMIN_PASSWORD (Grafana 비밀번호 변경 권장)
#   - PROMETHEUS_RETENTION_TIME (데이터 보관 기간)
```

이후 각 파일에서 URL, 비밀번호, targets 등을 환경에 맞게 수정합니다.

## 1. Central 서버 실행

```bash
cd central
docker compose up -d
```

- Prometheus: http://localhost:9090
- Alertmanager: http://localhost:9093
- Grafana: http://localhost:3000 (기본 로그인: `admin` / `admin`)

## 2. Agent 서버에서 exporter 실행

모니터링할 **각 서버**에서:

```bash
cd agent
docker compose up -d
```

기본으로 node_exporter(9100), smartctl_exporter(9633)가 올라갑니다.

## 3. Central에 타겟 등록

Central 서버의 `central/prometheus/prometheus.yml`(또는 위 0단계에서 복사한 파일)을 수정해, 해당 Agent 서버 IP를 넣습니다.

```yaml
scrape_configs:
  - job_name: "nodes"
    static_configs:
      - targets:
          - "192.168.0.11:9100"   # Agent 서버 IP
  - job_name: "smartctl"
    static_configs:
      - targets:
          - "192.168.0.11:9633"
```

저장 후 Prometheus 설정 리로드:

```bash
curl -X POST http://localhost:9090/-/reload
```

또는 Prometheus 컨테이너 재시작:

```bash
docker compose restart prometheus
```

## 4. Grafana 확인

1. http://localhost:3000 접속 → 로그인(admin/admin)
2. 좌측 메뉴 **Dashboards** → **MultiMonX** 폴더
3. **Fleet Overview**, **Node Detail**, **GPU Detail** 선택해 메트릭 확인

## 5. (선택) GPU 서버

GPU가 있는 Agent 서버에서:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

Central의 `prometheus.yml`에 `gpu` job 추가:

```yaml
  - job_name: "gpu"
    static_configs:
      - targets:
          - "192.168.0.11:9400"
```

이제 **GPU Detail** 대시보드에서 GPU 메트릭을 볼 수 있습니다.

## 6. (선택) Grafana 차트 범례에 서버 별칭(alias) 표시

차트 범례에는 기본적으로 수집 대상의 hostname(`instance`)이 나옵니다. 읽기 쉬운 이름(별칭)을 쓰려면 `prometheus.yml`의 해당 job에 **relabel**로 `alias`를 덮어쓰면 됩니다.

`central/prometheus/prometheus.yml`에서 `nodes` job의 예시 주석을 참고해, 아래처럼 `alias`를 바꾸는 블록을 추가합니다.

```yaml
# 예: hostname/IP를 별칭으로 변경
- source_labels: [alias]
  target_label: alias
  regex: "host\\.docker\\.internal"
  replacement: "My-Mac"
- source_labels: [alias]
  target_label: alias
  regex: "192\\.168\\.0\\.11"
  replacement: "web-server-1"
```

저장 후 `curl -X POST http://localhost:9090/-/reload` 로 Prometheus를 리로드하면, Grafana 차트 범례에 별칭이 반영됩니다.
