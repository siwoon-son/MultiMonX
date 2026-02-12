# MultiMonX Agent

각 모니터링 대상 서버에서 **exporter만** 실행합니다. Central 서버의 Prometheus가 이 endpoint를 주기적으로 Pull 합니다.

## Linux / macOS 공통 실행

동일한 `docker-compose`로 **Linux(amd64)** 와 **macOS(Intel 및 Apple Silicon)** 에서 모두 실행할 수 있습니다.

- Compose에 `platform: linux/amd64`를 지정해 두었습니다.
- **Linux x86_64**: 네이티브 실행.
- **macOS Intel**: 네이티브 실행.
- **macOS Apple Silicon (M1/M2/M3)**: Docker가 amd64 이미지를 에뮬레이션으로 실행합니다. 동작은 정상이며, 약간의 성능 차이가 있을 수 있습니다.

Apple Silicon에서 `no matching manifest for linux/arm64` 오류가 났다면, 위 설정이 이미 적용된 최신 compose를 사용 중인지 확인하세요.

## 기본 구성 (모든 서버)

```bash
cd agent
docker compose up -d
```

- **node_exporter**: 9100 (기본값) — CPU, 메모리, 디스크, 네트워크 등
- **smartctl_exporter**: 9633 (기본값) — 디스크 SMART 상태

## GPU 서버 (NVIDIA)

호스트에 NVIDIA 드라이버와 `nvidia-container-toolkit` 설치 후:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

- **dcgm-exporter**: 9400 (기본값) — GPU 사용률, 온도, 메모리

## IPMI 서버 (BMC 하드웨어 모니터링)

```bash
docker compose -f docker-compose.yml -f docker-compose.ipmi.yml up -d
```

- **ipmi_exporter**: 9290 (기본값, host 네트워크) — 전원, 온도, 팬 등

로컬 BMC 메트릭은 **설정 파일 없이** 수집됩니다. IPMI는 `network_mode: host`로 동작하므로, Central의 Prometheus에서는 **해당 호스트 IP:포트**로 scrape 하도록 설정하세요.

## 포트 지정 (여러 서버에서 다른 포트 사용 시)

여러 서버에서 exporter를 실행할 때 포트 충돌을 피하려면 `.env` 파일을 사용하세요.

**방법 1: .env 파일 사용 (권장)**

`.env.example`을 복사한 뒤 포트를 수정:

```bash
cd agent
cp .env.example .env
# .env 파일 편집: NODE_EXPORTER_PORT=9101, SMARTCTL_EXPORTER_PORT=9634 등
docker compose up -d
```

**방법 2: 환경 변수로 직접 지정**

```bash
NODE_EXPORTER_PORT=9101 SMARTCTL_EXPORTER_PORT=9634 docker compose up -d
```

**사용 가능한 환경 변수:**

- `NODE_EXPORTER_PORT` (기본값: 9100)
- `SMARTCTL_EXPORTER_PORT` (기본값: 9633)
- `DCGM_EXPORTER_PORT` (기본값: 9400)
- `IPMI_EXPORTER_PORT` (기본값: 9290)

**예시: 서버별 포트 설정**

```bash
# 서버 1: 기본 포트
docker compose up -d

# 서버 2: 다른 포트 사용
echo "NODE_EXPORTER_PORT=9101
SMARTCTL_EXPORTER_PORT=9634" > .env
docker compose up -d

# 서버 3: GPU 서버, 포트 변경
echo "DCGM_EXPORTER_PORT=9401" > .env
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

원격 IPMI 타겟/인증이 필요하면 `docker-compose.ipmi.yml`의 volumes/command 주석을 참고해 config 파일을 마운트할 수 있습니다.

## Central에 타겟 추가

Central 서버의 `central/prometheus/prometheus.yml`에서 `scrape_configs`의 `targets`에 agent 서버 주소를 추가합니다.

- nodes: `호스트IP:포트` (기본: 9100, .env에서 NODE_EXPORTER_PORT로 변경 가능)
- gpu: `호스트IP:포트` (기본: 9400, .env에서 DCGM_EXPORTER_PORT로 변경 가능)
- smartctl: `호스트IP:포트` (기본: 9633, .env에서 SMARTCTL_EXPORTER_PORT로 변경 가능)
- ipmi: `호스트IP:포트` (기본: 9290, host 모드 사용 시. .env에서 IPMI_EXPORTER_PORT로 변경 가능)
