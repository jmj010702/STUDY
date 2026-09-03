## `mail` health check 하나가 서비스 전체를 DOWN으로 만든다

### 문제 배경

`rate()` 사용법을 익히려고 `/actuator/health`에 초당 10건의 부하를 넣었다.
**학습이 목적이었지 장애를 찾을 의도가 아니었다.**

### 문제

메트릭에 이런 게 잡혔다.

```
uri="/actuator/health"  status="200"  →  308건
uri="/actuator/health"  status="503"  →  124건    ← 29%가 실패
```

health 요청의 **18~29%가 503**으로 실패하고 있었다. 그동안 아무도 몰랐다.

### 원인 분석

**가설 1: DB 커넥션 풀 고갈 → 기각**

부하 시점의 과거 데이터를 조회했다.

```
hikaricp_connections_active   : 0 0 0 0 0 0 0 0
hikaricp_connections_pending  : 0 0 0 0 0 0 0 0
hikaricp_connections_idle     : 5 5 5 5 5 5 5 5
hikaricp_connections_timeout  : 0 0 0 0 0 0 0 0
```

대기도 타임아웃도 0. **DB는 결백하다.**

**가설 2: 외부 의존 구성요소 → 적중**

503 응답 본문을 저장해서 확인했다.

```json
"mail": {
  "status": "DOWN",
  "details": {
    "location": "smtp.naver.com:465",
    "error": "jakarta.mail.MessagingException: Could not connect to SMTP host: smtp.naver.com, port: 465"
  }
}
```

**나머지 구성요소는 전부 UP**이었다 — db · redis(6개 팩토리) · diskSpace · ping · discoveryComposite(Eureka) · ssl · refreshScope.

**하나라도 DOWN이면 전체가 DOWN이고 HTTP 503이 된다.**

**왜 mail만 실패하나 — 연결 방식의 차이**

| | 연결 방식 | 초당 10건에서 |
|---|---|---|
| DB (HikariCP) | **커넥션 풀 5개를 재사용** | 여유 (`active 0`, `pending 0`, `timeout 0`) |
| **SMTP health check** | **매 요청마다 새로 연결** | 외부 서버가 거절 |

**재현성** — 두 번의 독립 실험에서 재현됐다.

| 실험 | 조건 | 요청 | 503 | 비율 |
|---|---|---|---|---|
| 부하 테스트 | 초당 10건 × 60초(30초 부하 2회) | 432건 | 124건 | **29%** |
| 재현 실험 | 초당 10건 × 5초 | 50건 | 9건 | **18%** |

재현 실험에서는 **실패 9건 전부가 `mail` DOWN**이었다.

### 왜 심각한가 — k8s에서는 자동 재시작으로 이어진다

```
네이버 SMTP가 잠깐 느려짐
    ↓
/actuator/health 가 503
    ↓
k8s livenessProbe 실패 → 파드 재시작
    ↓
메일 서버 때문에 HR 서비스가 재시작된다
```

**`/actuator/health`가 외부 인터넷 구간(네이버 SMTP)에 의존하고 있다는 뜻이다.**
"자동 조치는 하지 않는다"는 원칙을 세워도 **k8s는 그 원칙 밖에 있다.** 우리가 조치하지 않아도 플랫폼이 조치한다.

### 해결 방법

**정석은 health group으로 liveness/readiness를 나누는 것.**

```yaml
management:
  endpoint:
    health:
      group:
        liveness:
          include: livenessState              # "프로세스가 살아있나"
        readiness:
          include: readinessState,db,redis    # "요청 받을 준비 됐나" — mail 없음
```

| 경로 | 묻는 것 | 실패 시 k8s 동작 |
|---|---|---|
| `/actuator/health/liveness` | 프로세스가 살아있나 | **재시작** |
| `/actuator/health/readiness` | 트래픽 받을 준비 됐나 | **트래픽만 차단** (재시작 안 함) |

**`mail`은 둘 중 어디에도 속하지 않는다.** 메일을 못 보내도 출퇴근 조회는 정상 동작한다.

다른 방법들:

| 방법 | 설정 | 성격 |
|---|---|---|
| 그룹에서 제외 | `management.endpoint.health.group.custom.exclude: mail` | 특정 그룹에서만 뺌 |
| 지표 자체 비활성화 | `management.health.mail.enabled: false` | ⚠️ 정확한 키 미확인 |

### 현재 조치 — 고치지 않고 남겨뒀다

| 이유 | 내용 |
|---|---|
| 팀 저장소 | health 구성 변경은 **앱 동작을 바꾸는 일**이라 관측 설정 변경과 성격이 다르다. 공유 후 결정할 사안 |
| 장애 주입 재사용 | 이건 **실제로 발생한 장애 시나리오**다. 인위적 시나리오를 만들 것 없이 그대로 쓸 수 있다 |

---

### 재현 절차

**전제**: hr-service 기동 + Prometheus 수집 중. `<PORT>`는 Eureka에서 조회한 hr-service 포트.

**① 503 발생시키고 응답 본문 저장**

```bash
URL="http://localhost:<PORT>/actuator/health"
OUT="./health503"; rm -rf "$OUT"; mkdir -p "$OUT"

for round in $(seq 1 5); do
  for i in $(seq 1 10); do
    (
      resp=$(curl -s -m 5 -w "|%{http_code}" "$URL")
      code="${resp##*|}"; body="${resp%|*}"
      if [ "$code" = "200" ]; then echo "$code" >> "$OUT/ok.log"
      else echo "$code" >> "$OUT/fail.log"; echo "$body" > "$OUT/fail_${round}_${i}.json"; fi
    ) &
  done
  wait; sleep 1
done
echo "성공: $(wc -l < "$OUT/ok.log")  실패: $(wc -l < "$OUT/fail.log")"
```

**② DOWN 구성요소 집계**

```bash
grep -ohE '"[a-zA-Z]+":\{"status":"DOWN"' "$OUT"/fail_*.json | sort | uniq -c | sort -rn
# 기대: 9 "mail":{"status":"DOWN"
```

**③ DB가 원인이 아님을 확인** (부하 시점을 과거 조회)

```bash
export MSYS_NO_PATHCONV=1     # ← 없으면 빈 결과가 나온다 (Git Bash 경로 변환 문서 참조)
curl -s --get "http://localhost:9090/api/v1/query_range" \
  --data-urlencode 'query=hikaricp_connections_pending{job="hr-service"}' \
  --data-urlencode "start=<부하시작 ISO8601 UTC>" \
  --data-urlencode "end=<부하종료 ISO8601 UTC>" \
  --data-urlencode "step=30s"
# 기대: 전부 0  → 커넥션 대기 없음 → DB 무관
```

**④ 503 비율 확인** (메트릭 기준)

```promql
sum(rate(http_server_requests_seconds_count{job="hr-service",uri="/actuator/health",status="503"}[5m]))
/
sum(rate(http_server_requests_seconds_count{job="hr-service",uri="/actuator/health"}[5m]))
```

> 이 쿼리는 **RED의 E(오류율) 형태 그대로**다. `rate`를 **집계보다 먼저** 적용한 것에 유의.

---

### 배운 점

> 관측 체계를 붙이는 것이 목적이었는데, **붙이는 과정에서 기존 시스템의 문제가 드러났다.**
> "관측이 왜 필요한가"에 대해 이보다 직접적인 답은 없다.
> **부하 없이 유휴 상태였다면 영원히 몰랐을 문제다.**
