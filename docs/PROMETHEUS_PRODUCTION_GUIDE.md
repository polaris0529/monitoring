# Prometheus 운영환경 설정 매뉴얼

> 이 매뉴얼은 현재 프로젝트(`monitoring/`)의 실제 설정을 기준으로 작성되었습니다.  
> 순서대로 진행하면 운영환경에 바로 적용 가능한 상태가 됩니다.

---

## 목차

1. [현재 상태 점검](#1-현재-상태-점검)
2. [STEP 1 — global 설정 보강](#step-1--global-설정-보강)
3. [STEP 2 — Node Exporter 추가](#step-2--node-exporter-추가)
4. [STEP 3 — Alertmanager 연동](#step-3--alertmanager-연동)
5. [STEP 4 — Alert Rules 작성](#step-4--alert-rules-작성)
6. [STEP 5 — Prometheus 스토리지 옵션 보강](#step-5--prometheus-스토리지-옵션-보강)
7. [STEP 6 — Prometheus UI 접근 보안](#step-6--prometheus-ui-접근-보안)
8. [STEP 7 — 설정 반영 및 검증](#step-7--설정-반영-및-검증)
9. [선택 사항 — 장기 보존 (Remote Write)](#선택-사항--장기-보존-remote-write)
10. [운영 중 자주 쓰는 명령어](#운영-중-자주-쓰는-명령어)
11. [체크리스트 요약](#체크리스트-요약)

---

## 1. 현재 상태 점검

### 현재 구성 (`prometheus.yml` 기준)

| 항목 | 현재 값 | 문제 |
|------|---------|------|
| `scrape_interval` | 15s | 정상 |
| `evaluation_interval` | 1m | ⚠️ 너무 느림 — alert 평가가 1분 간격 |
| `scrape_timeout` | 미설정 | ⚠️ 기본값(10s) 의존 |
| `external_labels` | 미설정 | ⚠️ Alertmanager에서 출처 식별 불가 |
| Alertmanager | 미연동 | 🔴 알림 발송 불가 |
| Node Exporter | 미설정 | 🔴 호스트 메트릭 없음 |
| Alert Rules | `rules/` 비어 있음 | 🔴 경보 조건 없음 |
| Prometheus UI 인증 | 없음 | 🔴 외부 노출 시 무방비 |

---

## STEP 1 — global 설정 보강

**파일:** `prometheus.yml`

```yaml
global:
  # 모든 scrape job의 기본 수집 주기 (개별 job에서 override 가능)
  # 너무 짧으면 타겟 부하 증가, 너무 길면 이상 감지 지연 — 15s가 범용 적정값
  scrape_interval: 15s

  # alert rule 파일의 평가(발화 조건 계산) 주기
  # scrape_interval과 동일하게 맞추는 것이 원칙
  # 1m으로 두면 30초짜리 스파이크성 장애는 탐지 자체가 불가
  evaluation_interval: 15s

  # 단일 타겟에서 메트릭을 수집할 때 기다리는 최대 시간
  # scrape_interval보다 반드시 짧아야 함 (초과 시 설정 오류)
  # 기본값(10s)에 의존하기보다 명시적으로 선언하는 것이 권장됨
  scrape_timeout: 10s

  # 이 Prometheus가 생성하는 모든 메트릭/alert에 자동으로 붙는 레이블
  # Alertmanager, Grafana, 원격 저장소에서 "어느 서버의 데이터인지" 식별에 사용
  # 여러 Prometheus 인스턴스를 운영할 때 특히 중요
  external_labels:
    env: production           # 환경 구분 (production / staging / dev)
    region: ap-northeast-2   # 실제 서버 리전으로 변경
    cluster: my-service       # 실제 서비스명으로 변경
```

**왜 `evaluation_interval: 1m`이 위험한가:**  
Alert rule이 1분에 한 번만 평가됩니다. 30초짜리 스파이크성 장애(OOM, 포트 충돌 등)는 탐지 자체가 안 됩니다. `scrape_interval`과 동일하게 맞추는 것이 원칙입니다.

---

## STEP 2 — Node Exporter 추가

cAdvisor는 **컨테이너** 메트릭만 수집합니다. 호스트 OS 레벨 메트릭(CPU, 메모리, 디스크, 네트워크)은 Node Exporter가 필요합니다.

### `docker-compose.yml`에 추가

```yaml
services:
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    networks:
      - monitoring  # Prometheus와 같은 네트워크 — 외부 포트 노출 불필요

    # 호스트의 PID 네임스페이스를 공유
    # 이 옵션 없이는 컨테이너 내부 프로세스만 보임 — 호스트 전체 프로세스 수집 불가
    pid: host

    volumes:
      # 호스트의 /proc 를 읽기 전용으로 마운트 — CPU/메모리/네트워크 메트릭 수집 경로
      - /proc:/host/proc:ro
      # 호스트의 /sys 를 읽기 전용으로 마운트 — 디바이스/커널 메트릭 수집 경로
      - /sys:/host/sys:ro
      # 호스트 루트 파일시스템 — 디스크 사용량 수집 경로
      - /:/rootfs:ro

    command:
      # 컨테이너 내부에서 호스트 /proc 경로를 사용하도록 지정
      - "--path.procfs=/host/proc"
      # 컨테이너 내부에서 호스트 /sys 경로를 사용하도록 지정
      - "--path.sysfs=/host/sys"
      # 컨테이너 내부에서 호스트 루트 경로를 사용하도록 지정
      - "--path.rootfs=/rootfs"
      # 수집에서 제외할 마운트포인트 패턴 (정규식)
      # /sys, /proc 등 가상 파일시스템은 디스크 사용량 집계에서 제외해야 노이즈 없음
      # $$ 는 docker-compose에서 $ 이스케이프 처리
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"

    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.3"    # Node Exporter는 경량 — 0.3코어면 충분
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### `prometheus.yml` scrape_configs에 추가

```yaml
scrape_configs:
  - job_name: 'node'
    # job_name 은 수집된 메트릭에 job="node" 레이블로 자동 부착됨
    # Grafana에서 Node Exporter Full 대시보드(ID: 1860) 사용 시 이 이름으로 필터링
    static_configs:
      - targets: ['node-exporter:9100']
        # node-exporter 는 컨테이너 이름 — Docker 내부 DNS로 해석됨
        # 9100 은 Node Exporter 기본 포트
```

> Node Exporter는 외부 포트를 열 필요가 없습니다. monitoring 네트워크 내부에서만 통신합니다.

---

## STEP 3 — Alertmanager 연동

Alertmanager 없이는 alert rule을 아무리 잘 만들어도 **알림이 나가지 않습니다.**

### `docker-compose.yml`에 추가

```yaml
services:
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    networks:
      - monitoring  # Prometheus와 같은 네트워크 — 내부 DNS로 통신

    volumes:
      # alertmanager.yml: 알림 라우팅/수신자 설정 파일 (읽기 전용 마운트)
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      # 알림 상태(silence, inhibition 등)를 영구 보존하는 볼륨
      # 컨테이너 재시작 후에도 silence 설정이 유지됨
      - prometheus-alertmanager-data:/alertmanager

    command:
      # 설정 파일 경로 지정
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      # 알림 상태 데이터 저장 경로 (볼륨 마운트 경로와 일치해야 함)
      - "--storage.path=/alertmanager"
      # 외부에서 접근하는 Alertmanager의 공개 URL
      # 알림 메시지 내 "소스 링크"에 이 URL이 사용됨 — 실제 도메인으로 반드시 변경
      - "--web.external-url=http://your-domain/alertmanager"

    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.3"
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  prometheus-alertmanager-data:
    name: prometheus-alertmanager-data
```

### `alertmanager.yml` 생성

```yaml
global:
  # alert가 해소된 후 "resolved" 알림을 보내기까지 대기하는 시간
  # 너무 짧으면 flapping(발화-해소 반복) 시 resolved 알림이 과도하게 발송됨
  resolve_timeout: 5m

  # Slack Webhook URL — .env 파일에서 환경변수로 주입
  # 절대 하드코딩 금지
  slack_api_url: "${SLACK_WEBHOOK_URL}"

route:
  # 동일 그룹으로 묶을 레이블 목록
  # 같은 alertname + env 조합이면 하나의 알림 메시지로 묶어서 발송
  group_by: ['alertname', 'env']

  # 새 그룹의 첫 번째 알림을 보내기 전 대기 시간
  # 같은 시간대에 여러 alert가 동시에 발화할 경우 하나로 묶기 위한 버퍼
  group_wait: 30s

  # 이미 발송된 그룹에 새 alert가 추가됐을 때 다시 알림 보내기까지의 간격
  group_interval: 5m

  # 동일한 alert가 계속 발화 중일 때 반복 알림 주기
  # 4h: 해소되지 않은 장애를 4시간마다 재알림 — 너무 짧으면 알림 피로도 증가
  repeat_interval: 4h

  # 어떤 route에도 매칭되지 않은 alert의 기본 수신자
  receiver: 'slack-critical'

  # 세부 라우팅 규칙 — 위에서 아래 순서로 첫 번째 매칭되는 규칙 적용
  routes:
    - match:
        severity: warning   # severity=warning 레이블이 있으면 warning 채널로
      receiver: 'slack-warning'
    - match:
        severity: critical  # severity=critical 레이블이 있으면 critical 채널로
      receiver: 'slack-critical'

receivers:
  - name: 'slack-warning'
    slack_configs:
      - channel: '#alerts-warning'   # 경고 전용 Slack 채널
        title: '[WARNING] {{ .GroupLabels.alertname }}'
        # .Alerts 는 이 그룹에 묶인 alert 목록 — range로 순회하여 summary 출력
        text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
        # true: alert 해소 시 "resolved" 알림도 발송 (장애 종료 인지에 필수)
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'  # 긴급 전용 Slack 채널
        title: '[CRITICAL] {{ .GroupLabels.alertname }}'
        text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
        send_resolved: true

# 알림 억제 규칙 — 특정 조건 충족 시 다른 알림을 자동으로 묵음 처리
inhibit_rules:
  - source_match:
      severity: 'critical'    # critical alert가 발화 중이면
    target_match:
      severity: 'warning'     # 동일 인스턴스의 warning alert는 억제
    # 두 alert의 이 레이블 값이 같을 때만 억제 적용
    # alertname + instance 가 같은 경우에만 억제 — 다른 서비스의 warning은 유지
    equal: ['alertname', 'instance']
```

> Slack 대신 PagerDuty, OpsGenie, Email 등도 지원합니다. Alertmanager 공식 문서 참고.

### `prometheus.yml`에 Alertmanager 연동 추가

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
          # alertmanager: 컨테이너 이름 (Docker 내부 DNS)
          # 9093: Alertmanager 기본 수신 포트
      # Alertmanager 연결 타임아웃 — 이 시간 내에 응답 없으면 알림 전달 실패로 처리
      timeout: 10s
```

---

## STEP 4 — Alert Rules 작성

**파일 위치:** `rules/` 디렉토리 (현재 비어 있음)

### `rules/base-alerts.yml` — 인프라 기본 경보

```yaml
groups:
  - name: instance_availability   # 그룹 이름 — Alertmanager에서 group_by 기준으로 활용됨
    # 이 그룹만의 평가 주기 (global evaluation_interval 을 override)
    # 인스턴스 다운은 빠르게 감지해야 하므로 별도 지정
    interval: 15s
    rules:
      - alert: InstanceDown         # alert 이름 — Alertmanager의 alertname 레이블 값
        # up 메트릭: Prometheus가 scrape 성공 시 1, 실패 시 0을 자동으로 기록
        # up == 0 이면 해당 타겟이 응답하지 않는 상태
        expr: up == 0
        # 조건이 이 시간 동안 연속으로 참이어야 alert 발화
        # 1m: 일시적 네트워크 순단으로 인한 오탐 방지
        for: 1m
        labels:
          severity: critical        # Alertmanager 라우팅에서 수신자 결정에 사용
        annotations:
          # $labels: 해당 메트릭의 레이블 맵 — 어떤 서비스/인스턴스인지 동적으로 표시
          summary: "인스턴스 다운: {{ $labels.job }} / {{ $labels.instance }}"
          description: "{{ $labels.instance }} 가 1분 이상 응답하지 않습니다."

  - name: host_resources
    rules:
      - alert: HighCPUUsage
        # node_cpu_seconds_total{mode="idle"}: CPU가 idle 상태에서 소비한 누적 시간(초)
        # rate(): 초당 변화량 계산 — 5분 윈도우로 평균 비율 산출
        # avg by(instance): 코어가 여러 개여도 인스턴스 단위로 평균
        # 100 - (idle 비율 * 100) = 실제 CPU 사용률(%)
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        # 5분 지속 초과 시 발화 — 순간적 스파이크(배치 작업 등) 오탐 방지
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU 사용률 85% 초과: {{ $labels.instance }}"

      - alert: HighMemoryUsage
        # node_memory_MemAvailable_bytes: 실제 사용 가능한 메모리 (캐시 해제 가능분 포함)
        # node_memory_MemTotal_bytes: 전체 물리 메모리
        # (1 - 가용/전체) * 100 = 사용률(%) — MemFree 대신 MemAvailable 사용이 정확함
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "메모리 사용률 90% 초과: {{ $labels.instance }}"

      - alert: DiskSpaceLow
        # node_filesystem_avail_bytes: 일반 사용자가 쓸 수 있는 여유 공간 (root 예약분 제외)
        # node_filesystem_size_bytes: 전체 파티션 크기
        # mountpoint="/": 루트 파티션만 대상 — 다른 마운트포인트 추가 시 중괄호 조건 수정
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "디스크 여유 공간 15% 미만: {{ $labels.instance }}"

      - alert: DiskSpaceCritical
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 5
        # 5%는 즉각 대응이 필요한 수준 — for 를 1m으로 짧게 설정
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "디스크 여유 공간 5% 미만 (긴급): {{ $labels.instance }}"

  - name: container_resources
    rules:
      - alert: ContainerHighMemory
        expr: |
          container_memory_usage_bytes{name!=""}
          / container_spec_memory_limit_bytes{name!=""}
          > 0.9
          # container_memory_usage_bytes: 컨테이너 현재 메모리 사용량
          # container_spec_memory_limit_bytes: docker-compose limits.memory 에 설정한 값
          # name!="": 이름 없는 내부 컨테이너(POD 등) 제외
          # 비율이 0.9 초과 = 제한의 90% 이상 사용 중
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "컨테이너 메모리 90% 초과: {{ $labels.name }}"

      - alert: ContainerRestarting
        # container_last_seen: 컨테이너가 마지막으로 관측된 타임스탬프
        # rate() == 0: 5분간 새로운 관측이 없음 = 컨테이너가 반복 재시작 중이거나 소멸
        expr: rate(container_last_seen{name!=""}[5m]) == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "컨테이너 재시작 감지: {{ $labels.name }}"
```

### `rules/recording-rules.yml` — Recording Rules (쿼리 성능 최적화)

자주 쓰는 복잡한 쿼리는 미리 집계해두면 Grafana 대시보드 응답 속도가 빨라집니다.

```yaml
groups:
  - name: node_recordings
    # recording rule 전용 평가 주기
    # 1m: 대시보드 갱신 빈도와 맞춤 — 15s로 줄이면 저장 데이터가 4배 증가
    interval: 1m
    rules:
      # record: 결과를 저장할 새 메트릭 이름
      # 네이밍 규칙: {집계범위}:{원본메트릭}:{집계함수} — Prometheus 공식 권장 형식
      - record: job:node_cpu_usage:avg
        # 미리 계산된 결과가 새 메트릭으로 저장됨
        # Grafana에서 이 메트릭을 바로 조회 → 실시간 복잡한 연산 불필요
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      - record: job:node_memory_usage_percent:avg
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

      - record: job:node_disk_usage_percent:avg
        expr: 1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
```

---

## STEP 5 — Prometheus 스토리지 옵션 보강

**파일:** `docker-compose.yml` — prometheus 서비스의 `command` 섹션

```yaml
command:
  # 사용할 설정 파일 경로 (컨테이너 내부 경로)
  - "--config.file=/etc/prometheus/prometheus.yml"

  # 메트릭 데이터 보존 기간 — 이 기간이 지난 데이터는 자동 삭제
  # 30d: 한 달치 보존. 늘릴수록 디스크 사용량 증가
  - "--storage.tsdb.retention.time=30d"

  # 데이터 보존 최대 용량 — 시간/용량 중 먼저 도달한 기준으로 삭제
  # 디스크가 꽉 차서 Prometheus가 죽는 상황을 방지하는 안전망
  # 서버 디스크의 약 70~80% 수준으로 설정 권장
  - "--storage.tsdb.retention.size=10GB"

  # WAL(Write-Ahead Log) 파일을 압축하여 저장
  # 재시작 시 복구 속도가 약간 느려지지만 디스크를 약 20% 절약
  - "--storage.tsdb.wal-compression"

  # HTTP POST /-/reload 엔드포인트를 활성화
  # 설정 파일 변경 후 컨테이너 재시작 없이 curl 한 줄로 반영 가능
  # 활성화 시 이 엔드포인트가 외부에 노출되지 않도록 주의
  - "--web.enable-lifecycle"
```

> **`--web.enable-admin-api`는 운영환경에서 주의:** 데이터 강제 삭제 API가 열리므로, 인증 없이 외부에 노출되면 위험합니다. 필요한 경우에만 추가하고 반드시 인증을 걸어두세요.

---

## STEP 6 — Prometheus UI 접근 보안

현재 `prometheus`가 `nginx-proxy` 네트워크에 연결되어 있어 외부 접근이 가능한 상태입니다.  
Prometheus는 자체 인증이 없으므로 반드시 nginx에서 차단해야 합니다.

### 방법 A — nginx Basic Auth (권장)

```bash
# htpasswd 유틸리티 설치 (이미 설치되어 있으면 생략)
apt install apache2-utils -y

# -c: 새 파일 생성 (기존 파일 덮어씀 주의)
# prometheus-admin: 생성할 계정 이름
htpasswd -c /etc/nginx/.htpasswd prometheus-admin
```

nginx 설정:

```nginx
location /prometheus/ {
    # Basic Auth 활성화 — 브라우저에서 사용자명/비밀번호 입력창이 뜸
    auth_basic "Prometheus - Restricted";
    # 위에서 생성한 htpasswd 파일 경로
    auth_basic_user_file /etc/nginx/.htpasswd;

    # 인증 통과 후 Prometheus 컨테이너로 역방향 프록시
    proxy_pass         http://prometheus:9090/;
    # 원본 클라이언트 IP를 Prometheus에 전달 (로그 기록용)
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
}
```

### 방법 B — Prometheus 자체 Basic Auth (`--web.config.file`)

```yaml
# web-config.yml
# TLS 설정 — 인증서가 없으면 빈 블록으로 두면 됨
tls_server_config: {}

# Prometheus 자체 Basic Auth 사용자 목록
# 값은 반드시 bcrypt 해시여야 함 (평문 비밀번호 불가)
basic_auth_users:
  admin: $2y$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  # bcrypt
```

```yaml
# docker-compose.yml command에 추가
# web-config.yml 파일을 볼륨으로 마운트한 뒤 아래 옵션으로 경로 지정
- "--web.config.file=/etc/prometheus/web-config.yml"
```

bcrypt 해시 생성:

```bash
# -n: 파일에 저장하지 않고 화면 출력
# -B: bcrypt 알고리즘 사용
# -C 10: cost factor (높을수록 안전하나 느림 — 10이 권장값)
htpasswd -nBC 10 admin
```

### 방법 C — IP 화이트리스트 (내부 접근만 허용)

```nginx
location /prometheus/ {
    # 허용할 IP 대역 — 실제 내부 네트워크 대역으로 변경
    allow 192.168.1.0/24;    # 사내 내부망 대역
    allow 10.0.0.0/8;        # VPN/사설망 대역
    # 위 allow 조건에 해당하지 않는 모든 IP 차단
    deny all;

    proxy_pass http://prometheus:9090/;
}
```

---

## STEP 7 — 설정 반영 및 검증

### 설정 문법 검사 (컨테이너 재시작 전 반드시 실행)

```bash
# --rm: 검사 완료 후 임시 컨테이너 자동 삭제
# -v: 로컬 설정 파일을 컨테이너 내부로 마운트
# promtool check config: YAML 파싱 + scrape/rule 파일 참조 유효성까지 검사
docker run --rm \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  prom/prometheus:v2.48.0 \
  promtool check config /etc/prometheus/prometheus.yml

# alert rules 파일만 별도 검사 (PromQL 표현식 문법 검증 포함)
docker run --rm \
  -v $(pwd)/rules:/etc/prometheus/rules:ro \
  prom/prometheus:v2.48.0 \
  promtool check rules /etc/prometheus/rules/*.yml

# alertmanager.yml 문법 검사
# amtool check-config: 라우팅 트리, 수신자 정의, receiver 참조 유효성 검사
docker run --rm \
  -v $(pwd)/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro \
  prom/alertmanager:v0.26.0 \
  amtool check-config /etc/alertmanager/alertmanager.yml
```

### 컨테이너 재시작 없이 설정 리로드 (`--web.enable-lifecycle` 적용 후)

```bash
# SIGHUP 시그널 대신 HTTP POST로 설정 리로드 트리거
# 수집 중단 없이 prometheus.yml, rules 파일 변경사항 즉시 반영
curl -X POST http://localhost:9090/-/reload
```

### 수집 상태 확인

```bash
# activeTargets: 현재 scrape 중인 타겟 목록
# health: "up" (정상) / "down" (실패) / "unknown" (초기화 중)
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, instance: .labels.instance, health: .health}'

# state: "pending" (for 조건 대기 중) / "firing" (발화 중)
# pending → for 조건 충족 → firing → Alertmanager로 전달
curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {name: .labels.alertname, state: .state}'
```

### Alertmanager 테스트 알림 발송

```bash
# Alertmanager API에 직접 alert를 푸시하여 Slack 등 수신자에게 실제 전달되는지 확인
# labels: Alertmanager 라우팅 규칙에서 매칭에 사용됨
# annotations.summary: 알림 메시지 본문에 표시될 내용
curl -X POST http://localhost:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {"alertname": "TestAlert", "severity": "warning", "env": "production"},
    "annotations": {"summary": "Alertmanager 테스트 알림"}
  }]'
```

---

## 선택 사항 — 장기 보존 (Remote Write)

30일 이상 메트릭을 보존해야 하거나, 여러 Prometheus 인스턴스를 중앙에서 조회해야 할 경우.

### VictoriaMetrics (경량, 권장)

```yaml
# prometheus.yml
remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"
    # 전송 큐 설정 — 네트워크 순단 시 데이터 유실 방지를 위한 버퍼
    queue_config:
      # 한 번에 전송할 최대 샘플 수 — 클수록 처리량 증가, 메모리 사용도 증가
      max_samples_per_send: 10000
      # 전송 대기 큐 최대 크기 — 이 이상 쌓이면 가장 오래된 것부터 드롭
      capacity: 20000
      # 버퍼에 데이터가 있으면 이 시간 내로 강제 전송 (최대 지연 보장)
      batch_send_deadline: 5s
```

### Thanos (HA/글로벌 뷰가 필요한 대규모 환경)

Thanos Sidecar를 Prometheus 옆에 붙이는 구조로, 설정이 복잡합니다. 단일 서버 환경에서는 VictoriaMetrics가 훨씬 간단합니다.

---

## 운영 중 자주 쓰는 명령어

```bash
# Prometheus 컨테이너 로그 실시간 스트리밍
# scrape 오류, rule 평가 오류 발생 시 여기서 확인
docker logs -f prometheus

# 설정 문법 검사 통과 시 바로 리로드 (원라이너)
# && 로 체이닝 — 검사 실패 시 reload 명령 실행 안 됨
docker run --rm -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  prom/prometheus:v2.48.0 promtool check config /etc/prometheus/prometheus.yml \
  && curl -X POST http://localhost:9090/-/reload \
  && echo "리로드 완료"

# TSDB 내부 상태 조회
# headStats.numSeries: 현재 보유 시계열 수
# headStats.chunkCount: 압축된 청크 수
# sizeInBytes: 현재 디스크 사용량
curl -s http://localhost:9090/api/v1/status/tsdb | jq

# 현재 실행 중인 Prometheus 버전, 빌드 날짜, Go 버전 확인
curl -s http://localhost:9090/api/v1/status/buildinfo | jq
```

---

## 체크리스트 요약

적용 순서대로 체크하세요.

### 필수 (운영 전 반드시 완료)

- [ ] `evaluation_interval` → `15s` 로 변경
- [ ] `external_labels` 추가 (`env`, `region`, `cluster`)
- [ ] Node Exporter 서비스 추가 및 scrape 설정
- [ ] Alertmanager 서비스 추가 및 `alertmanager.yml` 작성
- [ ] `prometheus.yml`에 `alerting` 블록 추가
- [ ] `rules/base-alerts.yml` 작성 (InstanceDown, CPU, 메모리, 디스크)
- [ ] `--storage.tsdb.retention.size` 추가
- [ ] `--storage.tsdb.wal-compression` 추가
- [ ] `--web.enable-lifecycle` 추가
- [ ] Prometheus UI 접근 인증 설정 (nginx Basic Auth 또는 web-config.yml)
- [ ] 설정 문법 검사 (`promtool check config`) 통과 확인
- [ ] Alertmanager 테스트 알림 발송 확인

### 권장

- [ ] `rules/recording-rules.yml` 작성 (Grafana 응답 속도 개선)
- [ ] `scrape_timeout: 10s` 명시
- [ ] `.env`에 `SLACK_WEBHOOK_URL` 등 알림 채널 정보 추가

### 선택 (필요 시)

- [ ] VictoriaMetrics 또는 Thanos 연동 (30일 초과 보존)
- [ ] `--web.enable-admin-api` + 인증 적용

---

*마지막 업데이트: 2026-06-07*  
*기준 버전: Prometheus v2.48.0 / Alertmanager v0.26.0 / Node Exporter v1.7.0*