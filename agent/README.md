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

- **node_exporter**: 9100 — CPU, 메모리, 디스크, 네트워크 등
- **smartctl_exporter**: 9633 — 디스크 SMART 상태

## GPU 서버 (NVIDIA)

호스트에 NVIDIA 드라이버와 `nvidia-container-toolkit` 설치 후:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

- **dcgm-exporter**: 9400 — GPU 사용률, 온도, 메모리

## IPMI 서버 (BMC 하드웨어 모니터링)

```bash
docker compose -f docker-compose.yml -f docker-compose.ipmi.yml up -d
```

- **ipmi_exporter**: 9290 (host 네트워크) — 전원, 온도, 팬 등

IPMI는 `network_mode: host`로 동작하므로, Central의 Prometheus에서는 **해당 호스트 IP:9290**으로 scrape 하도록 설정하세요.

## Central에 타겟 추가

Central 서버의 `central/prometheus/prometheus.yml`에서 `scrape_configs`의 `targets`에 agent 서버 주소를 추가합니다.

- nodes: `호스트IP:9100`
- gpu: `호스트IP:9400`
- smartctl: `호스트IP:9633`
- ipmi: `호스트IP:9290` (host 모드 사용 시)
