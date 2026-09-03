# Observability — 관측 체계

PeopleCore(MSA, 4개 서비스)에 Prometheus + Grafana 관측 체계를 붙이면서 정리한 내용.
개념 정리 4건 + 실제로 막혔던 트러블슈팅 5건.

## 목차

### 개념 정리
| 문서 | 내용 |
|---|---|
| [PromQL.md](PromQL.md) | target/job/instance, `up`, `by`/`without`, instant vs range vector, `rate`/`increase`/`irate`, RED·USE |
| [Actuator.md](Actuator.md) | 자동 설정 흐름, `endpoints` vs `endpoint`, health 판정 규칙, `include:'*'`가 위험한 진짜 이유 |
| [Histogram_p95.md](Histogram_p95.md) | histogram이 무엇을 버리는가, `le`의 의미, p95 손계산 검증, 버킷 비용 |
| [ServiceDiscovery_Relabel.md](ServiceDiscovery_Relabel.md) | `__meta_` 라벨, `__address__`, relabel 규칙 구조, Eureka SD 실전 설정 |

### 트러블슈팅
| 문서 | 한 줄 |
|---|---|
| [health_mail_하나가_전체를_DOWN.md](트러블슈팅/health_mail_하나가_전체를_DOWN.md) | health 요청의 18~29%가 503. 원인은 매 요청마다 SMTP에 새로 연결하는 health check |
| [ES클라이언트_버전불일치_health503.md](트러블슈팅/ES클라이언트_버전불일치_health503.md) | HTTP 200으로 성공한 응답을 클라이언트가 해석 못 해서 health가 상시 503 |
| [SD전환후_죽은대상이_사라진다.md](트러블슈팅/SD전환후_죽은대상이_사라진다.md) | 앱이 오래 죽어 있을수록 알림이 조용해진다 |
| [404가_500으로_둔갑.md](트러블슈팅/404가_500으로_둔갑.md) | 없는 경로 요청이 500으로 집계되어 오류율을 오염시킨다 |
| [PromQL_함정_모음.md](트러블슈팅/PromQL_함정_모음.md) | 정규식 전체 일치, `by (le)` 누락, `avg`가 우연히 맞은 사건 |

---

# 관측이 왜 필요한가

## 관측이 없는 상태의 정의

> **장애를 사용자에게 배우는 상태.**

새벽에 서비스가 죽으면 사용자가 먼저 안다. 출근 체크가 안 돼서 연락이 오고, 그제야 대응이 시작된다.

```
관측 없음:  장애 발생 → (몇 시간) → 사용자 신고 → 원인 찾기 시작
관측 있음:  장애 발생 → (수십 초) → 알림      → 원인은 이미 화면에
```

좋은 관측은 한 단계 더 나아가 **터지기 전에** 알려준다 — "메모리가 3일째 차오른다", "응답시간이 평소의 3배다".

## 모니터링 vs 관측성

| | 모니터링 | 관측성 |
|---|---|---|
| 답하는 질문 | **"내가 아는 문제가 일어났나?"** | **"무슨 일이 일어나고 있나?"** |
| 방식 | 미리 정한 것만 감시 | 상태를 남겨두고 나중에 아무거나 질의 |
| 한계 | 예상 못 한 장애는 못 잡음 | — |

> 관측성 = 시스템이 남긴 데이터만으로 **미리 예상하지 못했던 질문에 답할 수 있는 성질**

### 실제로 겪은 사례 — Redis 정지 실험

`redis1` 컨테이너를 멈췄더니:

```
예상: redis 상태만 DOWN, 나머지는 정상
실제: health 응답 자체가 안 옴 (40초 타임아웃)
      + 일반 API도 20초 넘게 응답 없음
```

**Redis 하나가 앱 전체를 멈췄다.** 예상하지 못한 결과였다.

`health`는 "Redis가 죽었다"는 알려줬지만 **"왜 전체가 멈췄는지"는 답하지 못했다.** 그걸 알려면 응답시간 변화·스레드 상태·커넥션 대기를 봐야 하는데 그런 데이터를 남기고 있지 않았다. 이것이 모니터링의 한계다.

## 측정 가능한 목표

| 지표 | 뜻 |
|---|---|
| **MTTD** (Mean Time To Detect) | 장애 발생 → 알아채기까지 |
| **MTTR** (Mean Time To Recover) | 알아챈 후 → 복구까지 |

"관측을 도입했다"보다 **"MTTD를 몇 시간에서 몇 분으로 줄였다"**가 강한 이유.

---

# 메트릭 타입 4종

숫자라고 다 같은 숫자가 아니다. **어떻게 변하는지**에 따라 다루는 법이 달라진다.

## ① Counter — 늘어나기만 함

```
http_server_requests_seconds_count{...} 1523
```

요청 수·에러 수. 앱이 재시작하면 0으로 리셋된다.

**이 숫자 자체는 의미가 거의 없다.** "누적 1523건"이 많은지 적은지 모른다. 알고 싶은 건 "초당 몇 건씩 들어오나"이고, 그래서 `rate()`로 변환해서 쓴다.

## ② Gauge — 오르내림

```
jvm_memory_used_bytes{area="heap",id="G1 Eden Space"} 3.7748736E7
hikaricp_connections_active 3
```

메모리·커넥션 수. 현재값이라 그대로 읽으면 된다.

### 함정 — 순간값이라 짧은 작업을 놓친다

`hikaricp_connections_active`가 계속 0으로 보였다. 원인:

```
usage_seconds_sum 41.912초 ÷ usage_seconds_count 12,305회 = 평균 3.4ms
```

커넥션을 12,305번 썼는데 한 번 잡는 시간이 3.4ms였다. 시간의 약 4%만 활성인데 15초마다 찍으니 포착이 안 된 것. **버그가 아니라 gauge의 구조적 한계.**

> **규칙**: gauge 순간값만 보면 짧고 빈번한 작업을 놓친다. counter/summary 누적값의 `rate()`를 함께 봐야 한다.

## ③ Histogram — 분포

값 하나가 아니라 **구간별 개수**를 센다.

```
10ms 이하로 끝난 요청: 950건
50ms 이하: 990건
100ms 이하: 998건
```

이렇게 쌓아야 p95를 계산할 수 있다. 상세는 [Histogram_p95.md](Histogram_p95.md).

## ④ Summary — 미리 계산된 분위수

```
jvm_gc_pause_seconds{quantile="0.95"} 0.012
```

Histogram과 목적은 같은데 **앱에서 미리 계산해서 내보낸다.**

**차이**: Histogram은 여러 서비스를 **합칠 수 있고**, Summary는 못 합친다(각자 계산해버려서).
**MSA에서는 Histogram이 유리하다.**

## 실측 타입 분포 (hr-service, 188종)

```
gauge     133개   ← 가장 많음
counter    46개
summary    11개
histogram   0개   ← 없음!
```

**histogram이 0인 이유**: `http_server_requests_seconds_bucket`이 기본으로 꺼져 있다(버킷마다 시계열이 늘어 저장 비용 증가). `count`와 `sum`만 있어 **평균만 계산 가능하고 p95는 불가능**했다.

켜는 설정: `management.metrics.distribution.percentiles-histogram`

활성화 후 조합 1개당 버킷 **69개**가 생겼고 총 시계열이 12,371 → 12,757로 늘었다(트래픽 0 조건).
