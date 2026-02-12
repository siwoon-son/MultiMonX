# MultiMonX 보안 가이드

## 권장 사항

- **Exporter**: 모니터링 대상 서버의 exporter(9100, 9400, 9633, 9290)는 **내부망에서만** 접근 가능하도록 방화벽/네트워크 분리
- **Prometheus / Grafana**: 외부에 노출할 경우 반드시 인증 또는 reverse proxy로 보호

## Basic Auth (Prometheus / Grafana)

### Prometheus

Prometheus는 기본 설정으로 Basic Auth를 지원하지 않습니다. **Reverse proxy**로 Basic Auth를 적용하는 방식을 권장합니다.

Nginx 예시:

```nginx
server {
    listen 9090;
    server_name _;
    location / {
        auth_basic "Prometheus";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://prometheus:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`.htpasswd` 생성: `htpasswd -c .htpasswd admin`

### Grafana

Grafana는 기본 로그인(admin/admin)만 설정되어 있습니다. 배포 시 `GF_SECURITY_ADMIN_PASSWORD` 환경 변수로 강한 비밀번호를 설정하고, 필요 시 LDAP/OAuth 등 연동을 검토하세요.

## HTTPS Reverse Proxy 예시

Central 서버 앞단에 Nginx/Caddy를 두고 HTTPS 종료:

```nginx
server {
    listen 443 ssl;
    server_name monitor.example.com;
    ssl_certificate     /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    location /grafana/ {
        proxy_pass http://grafana:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location /prometheus/ {
        proxy_pass http://prometheus:9090/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Grafana의 `GF_SERVER_ROOT_URL`을 `https://monitor.example.com/grafana/`로 맞추어 주세요.

## 요약

| 항목 | 권장 |
|------|------|
| Exporter | 내부망 전용 |
| Prometheus | 내부망 또는 Basic Auth/HTTPS proxy |
| Grafana | 강한 admin 비밀번호 + HTTPS |
| Alertmanager | 내부망 또는 인증된 채널(Discord/Email)만 사용 |

Prometheus 생태계 표준을 활용하고, 인증·암호화는 proxy와 환경 변수로 선언적으로 구성하는 것을 권장합니다.
