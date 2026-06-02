# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 주요 명령어

```bash
# 모니터링 스택 기동
docker compose up -d

# 모니터링 스택 중지
docker compose down

# 설정 변경 후 재시작
docker compose restart prometheus

# Prometheus 설정 유효성 검사
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml

# 로그 확인
docker compose logs -f prometheus
docker compose logs -f grafana
```

## 아키텍처

모니터링 전용 Docker Compose 스택. 3개 서비스로 구성.

```
cAdvisor (컨테이너 메트릭 수집)
    └─▶ Prometheus (메트릭 저장/집계)
            └─▶ Grafana (시각화, nginx-net 통해 외부 노출)
```

**스크레이프 대상** (`prometheus.yml`):
- `localhost:9090` — Prometheus 자체 메트릭
- `cadvisor:8080` — 컨테이너 리소스 메트릭 (내부 DNS)
- `host.docker.internal:8081` — Spring Boot 앱 (`/api/actuator/prometheus`)

**네트워크**:
- `monitoring` (bridge) — 내부 서비스 간 통신 전용
- `nginx-proxy` (external, name: `nginx-net`) — Grafana 외부 노출용. 이 네트워크는 별도 nginx 컨테이너가 먼저 생성해야 함.

**데이터 보존**: Prometheus TSDB 30일 (`--storage.tsdb.retention.time=30d`)

## 설정 파일

- `prometheus.yml` — 스크레이프 잡 정의. 대상 추가 시 이 파일 수정 후 `docker compose restart prometheus`
- `.env` — Grafana 관리자 계정(`GF_SECURITY_ADMIN_PASSWORD`) 및 익명 접근 설정

## Spring Boot 연동

Spring Boot 앱(`/home/appadmin/spring-app`)이 `host.docker.internal:8081`로 노출한 Actuator Prometheus 엔드포인트를 스크레이프한다. 앱의 `application.properties`에서 `management.endpoints.web.exposure.include=prometheus` 및 `server.port=8081`(또는 해당 포트)이 활성화되어 있어야 한다.
