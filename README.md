# 모니터링 스택

Prometheus + Grafana + node-exporter 묶음. 서버 메트릭 긁어다가 그라파나로 본다.
Prometheus 내부 동작이나 이론적인 건 `docs/PROMETHEUS_PRODUCTION_GUIDE.md` 에 따로 정리해뒀다.

## 뭐가 떠있나

- `prometheus` (v2.48.0) — 수집하고 저장. 보존기간 30일로 잡아둠
- `grafana` (10.2.2) — 대시보드
- `host-node-exporter` (v1.7.0) — 호스트 CPU/메모리/디스크/네트워크

포트는 호스트로 직접 안 열어두고 nginx-proxy(`nginx-net`) 통해서만 접근한다. 컨테이너끼리는 `monitoring` 네트워크로 붙음.
스크레이프 대상은 `prometheus.yml` 에 셋 — 프로메테우스 자기 자신, node-exporter, 그리고 `workflow-app`(`/metrics`).

## 띄우기

`.env` 부터 만든다. 필요한 키는 `.env.example` 에 다 적어놨으니 복사해서 값만 채우면 됨.

```bash
cp .env.example .env
# 값 채우고

# monitoring 네트워크 없으면 먼저 만들고
docker network create monitoring

docker compose up -d
docker compose ps
```

## 자주 쓰는 명령

설정(`prometheus.yml`이나 `rules/`) 바꿨으면 재시작 말고 재로드만 하면 된다.

```bash
docker exec prometheus kill -HUP 1
```

타겟 살아있는지 (`up=1` 이면 정상, `0`이면 죽은 거):

```bash
docker exec prometheus wget -qO- http://localhost:9090/api/v1/targets
```

설정 문법 틀렸나 확인:

```bash
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml
```

로그는 그냥:

```bash
docker compose logs -f prometheus
```

## PromQL 메모

매번 까먹어서 적어둠.

셀렉터:

```promql
http_requests_total{job="api", code="200"}   # 정확히 일치
http_requests_total{job=~"api.*"}             # 정규식
http_requests_total{code!="500"}              # 부정
http_requests_total[5m]                        # 최근 5분 범위
```

카운터엔 `rate()`, 게이지엔 그냥 값이나 `avg_over_time()`. 스파이크 잡을 땐 `irate()`, 누적 증가량은 `increase()`.
분위수는 히스토그램에 `histogram_quantile(0.99, rate(..._bucket[5m]))`.

집계는 `sum`/`avg`/`max` 에 `by (라벨)` 나 `without (라벨)` 붙이는 식:

```promql
sum by (job) (rate(http_requests_total[5m]))
```

자주 보는 거:

```promql
# CPU 사용률 (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

# 메모리 사용률 (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 루트 디스크 사용률 (%)
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# 에러율
rate(http_requests_total{code=~"5.."}[5m]) / rate(http_requests_total[5m])

# p99 응답시간
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

## 알림 룰

`rules/*.yml` 에 넣으면 프로메테우스가 알아서 평가한다. 기본 형태는 이렇다.

```yaml
groups:
  - name: example
    rules:
      # 무거운 쿼리는 미리 계산해서 메트릭으로 저장 (레코딩 룰)
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # 조건 걸어서 알림 (알림 룰)
      - alert: InstanceDown
        expr: up == 0
        for: 1m          # 1분 내내 참이어야 발화. 순간 튀는 거 무시하려고
        labels:
          severity: critical
        annotations:
          summary: "인스턴스 다운: {{ $labels.instance }}"
```

`for` 가 핵심이다. 조건 충족돼도 이 시간 동안 계속 참이어야 진짜 알림이 나간다 (PENDING → FIRING).
룰 고쳤으면 위에 적은 `kill -HUP 1` 로 재로드.

## 폴더

```
docker-compose.yml   스택 정의
prometheus.yml       스크레이프/룰 설정
rules/               알림·레코딩 룰
.env.example         환경변수 템플릿
docs/                심화 가이드
samples/             설정 예시
```
