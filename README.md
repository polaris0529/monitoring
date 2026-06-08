# Monitoring Stack — Cheatsheet

Prometheus + Grafana + Node Exporter 기반 모니터링 스택. **Pull로 긁고, 레이블로 구분하고, PromQL로 질의한다.**

> 설계 철학·내부 구조 등 심화 내용은 [`docs/PROMETHEUS_PRODUCTION_GUIDE.md`](docs/PROMETHEUS_PRODUCTION_GUIDE.md) 참고.

---

## 구성

| 서비스 | 이미지 | 컨테이너 | 포트(내부) | 역할 |
|--------|--------|----------|-----------|------|
| Prometheus | `prom/prometheus:v2.48.0` | `prometheus` | 9090 | 수집·저장(TSDB, 보존 30d)·룰 평가 |
| Grafana | `grafana/grafana:10.2.2` | `grafana` | 3000 | 시각화 대시보드 |
| Node Exporter | `prom/node-exporter:v1.7.0` | `host-node-exporter` | 9100 | 호스트 메트릭(CPU/메모리/디스크/네트워크) |

- 네트워크: `monitoring`(내부), `nginx-net`(외부 노출은 nginx-proxy 경유) — **포트를 호스트에 직접 공개하지 않음**
- 스크레이프 타겟(`prometheus.yml`): `prometheus`, `host-node-exporter`, `workflow-app`(`/metrics`)

---

## 빠른 시작

```bash
# 1) 환경변수 준비 (키 목록은 .env.example 참고)
cp .env.example .env && vi .env

# 2) 외부 네트워크가 없으면 먼저 생성
docker network create monitoring   # 이미 있으면 생략

# 3) 기동
docker compose up -d

# 4) 상태 확인
docker compose ps
```

---

## 운영 명령어

```bash
# 전체 기동 / 중지 / 재시작
docker compose up -d
docker compose down
docker compose restart prometheus

# 로그
docker compose logs -f prometheus

# 설정 무중단 재로드 (prometheus.yml / rules 수정 후)
docker exec prometheus kill -HUP 1
#   또는 POST /-/reload (--web.enable-lifecycle 필요)

# 타겟 상태 확인 (up=1 / down=0)
docker exec prometheus wget -qO- http://localhost:9090/api/v1/targets | \
  grep -o '"job":"[^"]*"\|"health":"[^"]*"'

# 룰 / 알림 확인
docker exec prometheus wget -qO- http://localhost:9090/api/v1/rules
docker exec prometheus wget -qO- http://localhost:9090/api/v1/alerts

# 설정 문법 검증
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml
```

| 엔드포인트 | 용도 |
|-----------|------|
| `GET /-/healthy` | 헬스체크 |
| `GET /api/v1/query` | 즉시 쿼리(Instant) |
| `GET /api/v1/query_range` | 범위 쿼리(Range) |
| `GET /api/v1/targets` | 스크레이프 타겟 상태 |
| `POST /-/reload` | 설정 무중단 재로드 |

---

## PromQL 치트시트

### 셀렉터 / 범위

```promql
http_requests_total                          # 메트릭 이름
http_requests_total{job="api", code="200"}   # 정확히 일치
http_requests_total{job=~"api.*"}            # 정규식
http_requests_total{code!="500"}             # 부정
http_requests_total[5m]                       # 최근 5분 범위 벡터
http_requests_total offset 1h                 # 1시간 전 시점
```

### 핵심 함수

| 함수 | 타입 | 용도 |
|------|------|------|
| `rate(counter[d])` | Counter | 초당 평균 변화율 (그래프용) |
| `irate(counter[d])` | Counter | 순간 변화율 (스파이크 탐지) |
| `increase(counter[d])` | Counter | 기간 내 총 증가량 |
| `delta(gauge[d])` | Gauge | 기간 내 변화량 |
| `avg_over_time(gauge[d])` | Gauge | 기간 평균 |
| `histogram_quantile(0.99, ...)` | Histogram | 분위수(p99 등) |

### 집계 연산자

```promql
sum(rate(http_requests_total[5m]))                  # 전체 합산
sum by (job) (rate(http_requests_total[5m]))        # 레이블 그룹 합산
sum without (instance) (http_requests_total)        # 특정 레이블 제외 집계
max by (container) (container_memory_usage_bytes)   # 그룹별 최댓값
```

### 자주 쓰는 실전 쿼리

```promql
# CPU 사용률 (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

# 메모리 사용률 (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 루트 파일시스템 사용률 (%)
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# 에러율
rate(http_requests_total{code=~"5.."}[5m]) / rate(http_requests_total[5m])

# p99 응답시간
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

---

## 알림 / 레코딩 룰 (`rules/*.yml`)

```yaml
groups:
  - name: example
    rules:
      # 레코딩 룰 — 무거운 쿼리 사전 계산 (네이밍: {집계수준}:{메트릭}:{연산})
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # 알림 룰 — 인스턴스 다운
      - alert: InstanceDown
        expr: up == 0
        for: 1m                  # for 기간 연속 충족 시 FIRING (순간 스파이크 방지)
        labels:
          severity: critical
        annotations:
          summary: "인스턴스 다운: {{ $labels.instance }}"
```

| 필드 | 역할 |
|------|------|
| `expr` | 평가할 PromQL 조건 |
| `for` | 이 기간 연속 충족돼야 알림 발화 (INACTIVE → PENDING → FIRING) |
| `labels` | Alertmanager 라우팅 기준 |
| `annotations` | 메시지 본문 (`{{ $value }}`, `{{ $labels.x }}` 템플릿) |

> 룰 수정 후 `docker exec prometheus kill -HUP 1` 로 재로드.

---

## 디렉토리

```
.
├── docker-compose.yml       # 스택 정의
├── prometheus.yml           # 스크레이프 / 룰 파일 설정
├── rules/                   # 알림·레코딩 룰 (*.yml)
├── .env.example             # 환경변수 템플릿 (.env 로 복사 후 값 입력)
├── docs/                    # 심화 운영 가이드
└── samples/                 # 설정 예시
```
