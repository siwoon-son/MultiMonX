# MultiMonX 알림 설정

## 기본 제공 Alert 규칙

| 규칙 | 조건 | 지속 시간 | 심각도 |
|------|------|-----------|--------|
| NodeDown | up == 0 | 1m | critical |
| CPUUsageHigh | CPU > 90% | 1m | warning |
| MemoryHigh | 메모리 > 90% | 1m | warning |
| DiskUsageHigh | 디스크 > 90% | 5m | warning |
| GPUUtilHigh | GPU 사용률 > 90% | 1m | warning |
| GPUTempHigh | GPU 온도 > 85°C | 1m | warning |

규칙 파일 위치:

- `central/prometheus/rules/node-alerts.yml`
- `central/prometheus/rules/gpu-alerts.yml`

## Alertmanager 설정

알림 전송은 `central/alertmanager/alertmanager.yml`에서 설정합니다.  
(최초에는 `alertmanager.yml.example`을 `alertmanager.yml`로 복사한 뒤 수정하세요.)

### Discord Webhook

1. Discord 서버 → 채널 설정 → 연동 → 웹후크 생성 후 URL 복사
2. `alertmanager.yml`에서 receiver에 webhook_config 추가:

```yaml
receivers:
  - name: 'default'
    webhook_configs:
      - url: 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_TOKEN'
        send_resolved: true
```

3. Alertmanager 재시작: `docker compose restart alertmanager`

### Email

```yaml
receivers:
  - name: 'default'
    email_configs:
      - to: 'ops@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'your-password'
        headers:
          Subject: '[MultiMonX] {{ .GroupLabels.alertname }}'
```

Gmail 등 사용 시 앱 비밀번호 및 TLS 설정이 필요할 수 있습니다.

### 라우팅

- `route.receiver`: 기본 수신자
- `route.routes`: `match`(예: `severity: critical`)별로 다른 receiver 지정 가능
- `repeat_interval`: 동일 알림 재전송 간격(기본 4h)

설정 변경 후 `docker compose restart alertmanager`로 적용합니다.
