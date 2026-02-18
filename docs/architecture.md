# MultiMonX 아키텍처

## 개요

MultiMonX는 **Prometheus 표준 Pull 모델**을 따릅니다.

- **Agent**: 각 서버에서 exporter만 실행해 `/metrics` HTTP endpoint 제공
- **Central**: Prometheus가 주기적으로 해당 endpoint를 scrape → Alertmanager로 알림 → Grafana로 시각화

복잡한 relay/federation 없이 **중앙 1대 + 대상 서버 N대** 구조입니다.

## 구성도

```
[ Agent 서버 1 ]     [ Agent 서버 2 ]     [ Agent 서버 N ]
  node_exporter        node_exporter        node_exporter
  smartctl_exporter    dcgm_exporter        (선택)
  (선택: dcgm, ipmi)   (선택: ipmi)

        │                     │                     │
        │  HTTP GET /metrics  │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │   Central Monitoring Server    │
              │  ┌─────────────┐               │
              │  │ Prometheus  │  scrape 15s   │
              │  └──────┬──────┘               │
              │         │                       │
              │  ┌──────▼──────┐  ┌──────────┐ │
              │  │Alertmanager │  │ Grafana  │ │
              │  └─────────────┘  └──────────┘ │
              └───────────────────────────────┘
```

## 컴포넌트

| 역할 | 컴포넌트 | 포트 | 설명 |
|------|----------|------|------|
| Central | Prometheus | 9090 | 메트릭 수집·저장·알림 규칙 평가 |
| Central | Alertmanager | 9093 | 알림 그룹핑·중복 제거·Discord/Email 등 전송 |
| Central | Grafana | 3000 | 대시보드·Prometheus 데이터소스 |
| Agent | node_exporter | 9100 | CPU, 메모리, 디스크, 네트워크 |
| Agent | smartctl_exporter | 9633 | 디스크 SMART 상태 |
| Agent (선택) | dcgm-exporter | 9400 | NVIDIA GPU 메트릭 |
| Agent (선택) | ipmi_exporter | 9290 | IPMI/BMC 하드웨어 |

## 데이터 흐름

1. **Scrape**: Prometheus가 `scrape_configs`에 정의된 target(예: `host:9100`)에 주기적(기본 15s) HTTP 요청
2. **Storage**: 수집한 메트릭을 TSDB에 저장(기본 15일 보관)
3. **Rules**: `rule_files`의 alert 규칙을 평가해 조건 충족 시 Alertmanager로 전달
4. **Alerts**: Alertmanager가 라우팅·그룹핑 후 receiver(Discord, Email 등)로 전송
5. **Visualization**: Grafana가 Prometheus를 데이터소스로 쿼리해 대시보드 표시

## 네트워크

Central과 Agent는 **동일한 Docker 네트워크(multimonx)** 를 공유합니다. `docker network create multimonx`를 최초 1회 실행한 뒤, Central만 단독, Agent만 단독, 또는 둘 다 실행하는 모든 조합에서 같은 네트워크를 사용합니다.

## 확장

- **서버 추가**: Agent를 새 서버에 설치한 뒤, Central의 `prometheus.yml`의 `static_configs.targets`에 `호스트:포트` 추가 (같은 네트워크면 컨테이너 이름 예: `multimonx-node-exporter:9100` 사용 가능)
- **동적 타겟**: 필요 시 `file_sd_configs`로 파일 기반 타겟 목록 전환 가능(구성만 변경)
- **인증**: Prometheus/Grafana 앞단에 reverse proxy 두고 Basic Auth 또는 HTTPS 적용 가능(문서 참고)
