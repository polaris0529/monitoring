# Prometheus 설계 철학 & 핵심 정리

---

## 1. 설계 언어 — "Pull 기반 시계열 DB"

Prometheus는 **SoundCloud에서 2012년에 만든 오픈소스 모니터링 시스템**으로, Go 언어로 작성됐다.

핵심 설계 언어는 **"수집 대상이 아닌, Prometheus가 능동적으로 긁어온다(Pull)"** 는 것.

```
[Push 방식]  App ──→ 모니터링 서버    (대부분의 APM)
[Pull 방식]  Prometheus ──→ App /metrics  (Prometheus)
```

---

## 2. 명확한 설계 의도

| 의도 | 내용 |
|------|------|
| **단순성 우선** | 외부 의존성 없이 단일 바이너리 하나로 동작 |
| **신뢰성** | 모니터링 시스템이 다운되면 안 된다 → 분산 합의 없이 독립 동작 |
| **Pull 방식** | 수집 대상의 상태를 Prometheus가 판단 (대상이 응답 없으면 장애로 인식) |
| **레이블(Label) 중심** | 계층 구조 대신 레이블 키-값으로 다차원 데이터 표현 |
| **로컬 스토리지** | 분산 스토리지 없이 로컬 TSDB에 저장 → 단순하지만 수평 확장은 한계 |
| **정확성보다 근사값** | 100% 정밀도보다 "충분히 정확한 현재 상태 파악"이 목표 |

> 핵심 철학: **"운영 중에도 살아남는 모니터링"** — 모니터링 시스템 자체는 단순하고 견고해야 한다.

### 신뢰성 상세 — "분산 합의 없이 독립 동작"

모니터링 시스템이 장애를 감지하려면 **모니터링 시스템 자신이 먼저 살아있어야 한다.**

Prometheus는 이 문제를 다음 방식으로 해결한다.

**분산 합의(Consensus)를 제거한다**

Zookeeper, etcd 같은 분산 합의 시스템은 클러스터 구성원 과반수가 동의해야 쓰기가 성공한다.
네트워크 파티션이 발생하면 합의가 깨져 시스템 전체가 멈출 수 있다.

Prometheus는 합의 프로토콜 자체를 사용하지 않는다.
각 인스턴스가 독립적으로 스크래핑하고 독립적으로 저장한다.

```
[일반 분산 시스템]
Node A ──┐
Node B ──┼──▶ 합의(Consensus) ──▶ 쓰기 성공
Node C ──┘
          ↑ 과반수 실패 시 전체 중단

[Prometheus]
Node A  ──▶ 로컬 TSDB (독립 동작)
Node B  ──▶ 로컬 TSDB (독립 동작)  ← 서로 무관하게 동작
```

**결과적 일관성을 허용한다**

모든 노드의 데이터가 완전히 동일하지 않아도 괜찮다는 입장.
인프라 장애 상황에서도 **각 Prometheus가 독립적으로 수집을 계속**하는 것이 더 중요하다.

**실제 고가용성 구성**

단일 Prometheus의 한계(단일 장애점)는 **동일한 설정의 인스턴스를 2개 병렬 운영**으로 해결한다.
두 인스턴스가 같은 대상을 동시에 스크래핑 → 하나가 죽어도 나머지가 계속 수집.

```
Prometheus A ──┐
               ├──▶ Grafana (두 datasource를 동시에 바라봄)
Prometheus B ──┘
```

> 핵심: Prometheus의 신뢰성 전략은 "복잡한 분산 합의로 완벽함을 추구"하는 것이 아니라
> "단순하게 독립 동작해서 장애 전파를 차단"하는 것이다.

---

## 3. 아키텍처 흐름

```
[Exporter / App /metrics]
         ↑ scrape (15s마다 Pull)
[Prometheus Server]
  ├── TSDB (로컬 저장, 기본 15일)
  ├── Rule Engine (알림 조건 평가)
  └── HTTP API (PromQL 쿼리)
         ↓
[Alertmanager]   [Grafana]   [PromQL CLI]
```

---

## 4. Prometheus 서버 구성요소 상세

### 4-1. Retrieval (스크레이핑 엔진)

Prometheus 서버의 핵심 수집 모듈. 설정된 스케줄대로 타겟 엔드포인트에 HTTP GET 요청을 보내 메트릭을 긁어온다.

| 항목 | 내용 |
|------|------|
| 기본 수집 주기 | `scrape_interval: 15s` (전역 설정, 타겟별 오버라이드 가능) |
| 타임아웃 | `scrape_timeout: 10s` (이 안에 응답 없으면 실패로 처리) |
| 수집 경로 | 기본 `/metrics`, 타겟별 `metrics_path` 로 변경 가능 |
| 실패 처리 | 스크레이프 실패 시 `up` 메트릭이 `0`이 됨 (`up{job="..."} == 0`) |

```yaml
# prometheus.yml 수집 설정 예시
global:
  scrape_interval: 15s
  scrape_timeout: 10s

scrape_configs:
  - job_name: "spring-app"
    scrape_interval: 5s        # 개별 오버라이드
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["app:8080"]
```

---

### 4-2. Service Discovery (서비스 디스커버리)

타겟을 정적으로 하드코딩하지 않고 동적으로 발견하는 메커니즘.

| 종류 | 설명 |
|------|------|
| `static_configs` | IP/포트 직접 지정 — 소규모 고정 환경 |
| `file_sd_configs` | JSON/YAML 파일을 주기적으로 읽어 타겟 갱신 |
| `docker_sd_configs` | Docker 컨테이너 자동 감지 |
| `kubernetes_sd_configs` | K8s Pod/Service/Node 자동 감지 |
| `ec2_sd_configs` | AWS EC2 인스턴스 자동 감지 |

```
[Service Discovery 흐름]
설정 파일 / 외부 시스템
       ↓ 타겟 목록 갱신
  Retrieval 엔진
       ↓ 레이블 재지정 (relabeling)
   실제 스크레이프
```

**Relabeling** — 수집 전후 레이블을 변환/필터링하는 파이프라인. 불필요한 타겟 제거, 레이블 이름 변경, 값 추출 등에 사용.

---

### 4-3. TSDB (Time Series Database)

Prometheus 내장 시계열 저장소. 외부 DB 없이 로컬 디스크에 저장.

**저장 구조**

```
data/
├── 01BKGV7JBM69T2G1BGBGM6KB12/   ← Block (2시간 단위)
│   ├── chunks/                     ← 실제 메트릭 데이터 (압축)
│   ├── index                       ← 레이블 → 청크 인덱스
│   └── meta.json                   ← 블록 메타데이터
├── chunks_head/                    ← 현재 수집 중인 메모리 블록
└── wal/                            ← Write Ahead Log (충돌 복구용)
```

| 항목 | 기본값 | 설명 |
|------|--------|------|
| 보존 기간 | `15d` | `--storage.tsdb.retention.time` 플래그로 변경 |
| 보존 용량 | 무제한 | `--storage.tsdb.retention.size` 로 최대 크기 제한 가능 |
| 블록 크기 | 2시간 | 메모리에서 디스크로 플러시되는 단위 |
| 압축 방식 | Gorilla(XOR) + Snappy | 시계열 특성에 최적화된 델타-오브-델타 인코딩 |

**WAL (Write Ahead Log)** — 서버 비정상 종료 시 메모리에 있던 최근 데이터를 WAL에서 복구. 유실 방지용.

---

### 4-4. Rule Engine (규칙 평가 엔진)

Prometheus 내부에서 **주기적으로 PromQL 표현식을 실행**하는 평가 루프.
TSDB에서 데이터를 읽어 규칙을 계산하고, 결과를 다시 TSDB에 쓰거나 Alertmanager에 전송한다.

```
[TSDB] ──읽기──▶ [Rule Engine 평가 루프 (evaluation_interval: 1m)]
                        ├── Recording Rule → 결과를 새 메트릭으로 TSDB에 쓰기
                        └── Alerting Rule  → 조건 충족 시 Alertmanager로 HTTP POST
```

**평가 주기 설정** (`prometheus.yml`)

```yaml
global:
  evaluation_interval: 1m    # 모든 Rule을 1분마다 재평가

rule_files:
  - "rules/*.yml"            # 규칙 파일 경로 (glob 지원)
```

---

#### Recording Rule — 쿼리 사전 계산

복잡하거나 비용이 큰 PromQL을 **미리 계산해 새 메트릭 이름으로 저장**하는 규칙.

**언제 써야 하나**
- Grafana 대시보드가 무거운 집계 쿼리를 반복 실행할 때
- `rate(...)[5m]` 같은 범위 쿼리를 여러 대시보드가 공유할 때
- 쿼리 응답이 느려서 타임아웃이 발생할 때

**네이밍 컨벤션** — `{집계수준}:{메트릭명}:{연산}`

```yaml
groups:
  - name: recording_rules
    interval: 1m
    rules:
      # job 단위로 초당 요청 수 사전 계산
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # instance 단위 CPU 사용률 사전 계산
      - record: instance:cpu_usage:rate1m
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

> 저장된 `job:http_requests:rate5m` 은 일반 메트릭처럼 PromQL에서 바로 조회 가능.

---

#### Alerting Rule — 조건 기반 알림

PromQL 조건이 **`for` 기간 동안 지속**되면 Alertmanager로 알림을 전송하는 규칙.

**필드 설명**

| 필드 | 역할 |
|------|------|
| `alert` | 알림 이름 (Alertmanager에서 식별자로 사용) |
| `expr` | 평가할 PromQL 조건 (결과가 비어 있으면 비활성) |
| `for` | 조건이 이 시간 동안 연속 충족돼야 FIRING (순간 스파이크 방지) |
| `labels` | 알림에 붙이는 레이블 — Alertmanager 라우팅 기준으로 활용 |
| `annotations` | 알림 메시지 본문 — `{{ $value }}`, `{{ $labels.instance }}` 템플릿 사용 가능 |

```yaml
groups:
  - name: alerting_rules
    rules:
      # 에러율 5% 초과 경고
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{code=~"5.."}[5m])
          / rate(http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "에러율 초과: {{ $labels.job }}"
          description: "현재 에러율 {{ $value | humanizePercentage }} (임계값 5%)"

      # 인스턴스 다운 감지
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "인스턴스 다운: {{ $labels.instance }}"
```

---

#### 알림 상태 전이 (State Transition)

```
         조건 충족                  for 기간 경과
INACTIVE ──────────▶ PENDING ──────────────────▶ FIRING
                        │                           │
                        │ 조건 해소 (for 미달)       │ 조건 해소
                        ▼                           ▼
                    INACTIVE                   RESOLVED
                                          (Alertmanager에 통보)
```

| 상태 | 의미 |
|------|------|
| `INACTIVE` | 조건 미충족 — 정상 상태 |
| `PENDING` | 조건 충족됐지만 `for` 기간 아직 미달 — 관찰 중 |
| `FIRING` | `for` 기간 초과 — Alertmanager로 알림 전송 중 |
| `RESOLVED` | 조건 해소 — Alertmanager에 종료 신호 전송 |

> `PENDING` 상태가 존재하는 이유: 순간적인 스파이크(1~2초)에 즉시 반응하면 오탐이 많아짐.
> `for: 2m` 은 "2분 내내 조건이 참이어야 알림을 보낸다"는 의미.

---

#### Rule Group 격리

Rule은 반드시 `group` 안에 정의되며, **같은 그룹 내 규칙은 순서대로 순차 실행**된다.
Recording Rule이 먼저 실행돼야 Alerting Rule에서 그 결과를 참조할 수 있을 때 같은 그룹에 배치.

```yaml
groups:
  - name: latency_group
    interval: 30s
    rules:
      # 1) 먼저 계산해서 저장
      - record: job:request_latency_p99:rate5m
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

      # 2) 위에서 저장한 메트릭을 알림 조건에서 참조
      - alert: HighLatency
        expr: job:request_latency_p99:rate5m > 1.0
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "p99 응답시간 1초 초과 ({{ $value }}s)"
```

---

### 4-5. HTTP API & PromQL 엔진

외부에서 메트릭을 조회할 수 있는 인터페이스. Grafana, 커스텀 도구, CLI 모두 이 API를 사용.

| 엔드포인트 | 설명 |
|-----------|------|
| `GET /api/v1/query` | 특정 시점 즉시 쿼리 (Instant Query) |
| `GET /api/v1/query_range` | 시간 범위 쿼리 (Range Query) |
| `GET /api/v1/targets` | 현재 스크레이프 타겟 상태 확인 |
| `GET /api/v1/rules` | 등록된 Rule 목록 확인 |
| `GET /api/v1/alerts` | 현재 발화 중인 알림 목록 |
| `GET /-/healthy` | 헬스체크 |
| `POST /-/reload` | 설정 파일 무중단 재로드 |

---

### 4-6. Alertmanager (알림 라우팅 서버)

Prometheus와 **별도 프로세스**로 실행. Rule Engine에서 받은 알림을 적절한 수신처로 라우팅.

```
[Prometheus Rule Engine]
        ↓ HTTP POST /api/v2/alerts
[Alertmanager]
  ├── Grouping    : 같은 알림 묶기 (group_by)
  ├── Inhibition  : 상위 알림 발생 시 하위 알림 억제
  ├── Silencing   : 점검 시간대 알림 무시
  └── Routing     : Slack / PagerDuty / Email / Webhook
```

---

### 4-7. Exporter (메트릭 변환기)

Prometheus 포맷을 직접 노출하지 않는 시스템 앞에 붙이는 어댑터 프로세스.

| Exporter | 수집 대상 | 주요 메트릭 |
|----------|-----------|-------------|
| `node_exporter` | 리눅스 서버 | CPU, 메모리, 디스크, 네트워크 |
| `cadvisor` | Docker 컨테이너 | 컨테이너 CPU/메모리/네트워크 |
| `mysqld_exporter` | MySQL | 쿼리 수, 슬로우 쿼리, 커넥션 |
| `redis_exporter` | Redis | 커맨드 수, 메모리, 히트율 |
| `blackbox_exporter` | HTTP/TCP/ICMP | 응답시간, 상태코드, 인증서 만료 |

Spring Boot Actuator (`/actuator/prometheus`)는 Exporter 없이 앱 자체가 메트릭을 직접 노출.

---

### 4-8. Pushgateway (Push 방식 보완)

Pull 방식이 맞지 않는 **배치 잡(Batch Job)** 을 위한 중간 게이트웨이.

```
[Batch Job] ──Push──▶ [Pushgateway] ──Pull──▶ [Prometheus]
```

> 주의: 상시 서비스에는 사용하지 말 것. Pushgateway는 단기 실행 잡의 완료 메트릭 수집 전용.

---

## 5. 주요 명령어 / PromQL 문법

### 기본 셀렉터

```promql
# 메트릭 이름만
http_requests_total

# 레이블 필터 (정확히 일치)
http_requests_total{job="api", code="200"}

# 레이블 필터 (정규식)
http_requests_total{job=~"api.*"}

# 레이블 부정
http_requests_total{code!="500"}
```

### 범위 벡터 (시간 범위 지정)

```promql
# 최근 5분치 데이터 범위
http_requests_total[5m]

# 1시간 전 시점 기준
http_requests_total offset 1h
```

### 핵심 함수

| 함수 | 타입 | 용도 |
|------|------|------|
| `rate(counter[d])` | Counter | 초당 평균 변화율 (그래프용) |
| `irate(counter[d])` | Counter | 순간 변화율 (스파이크 탐지) |
| `increase(counter[d])` | Counter | 기간 내 총 증가량 |
| `delta(gauge[d])` | Gauge | 기간 내 변화량 |
| `avg_over_time(gauge[d])` | Gauge | 기간 평균 |
| `histogram_quantile(0.99, ...)` | Histogram | 분위수 계산 (p99 등) |

### 집계 연산자

```promql
# 전체 합산
sum(rate(http_requests_total[5m]))

# 레이블 기준 그룹 합산
sum by (job) (rate(http_requests_total[5m]))

# 특정 레이블 제외하고 집계
sum without (instance) (http_requests_total)

# 최댓값
max by (container) (container_memory_usage_bytes)
```

### 자주 쓰는 실전 쿼리

```promql
# CPU 사용률 (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

# 메모리 사용률 (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 에러율
rate(http_requests_total{code=~"5.."}[5m]) / rate(http_requests_total[5m])

# p99 응답시간
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

---

## 6. 한 줄 요약

> Prometheus = **"Pull로 긁고, 레이블로 구분하고, PromQL로 질의한다"**
> 복잡한 인프라 없이 단일 프로세스가 신뢰성 있게 동작하는 것이 핵심 설계 의도.
