## PromQL 함정 모음 — "성공처럼 보이는 실패"

> 셋 다 공통점이 있다. **에러가 나지 않는다.**
> 문법이 유효하면 Prometheus는 실행하고, 결과가 없으면 그냥 없다고 답한다.

---

## 1. 정규식은 부분 일치가 아니라 전체 일치다

### 문제

`status=~"5"`가 500을 잡을 것 같지만 **0개**가 나온다.

### 실측 (당시 존재하던 status 값은 200과 204뿐)

```
status=~"2"     →  0개    ← 200, 204에 '2'가 들어있는데도
status=~"0"     →  0개    ← 200, 204에 '0'이 있는데도
status=~"2.*"   →  9개
status=~"2.."   →  9개
```

**라벨 값 전체가 패턴에 맞아야 한다.**

### 왜 위험한가

5xx 오류율을 만들면서 `status=~"5"`라고 쓰면 **에러 없이 빈 결과**가 나온다.
**오류율이 0%로 보이고, 진짜 장애가 나도 대시보드는 초록불이다.**

### 함께 걸리는 것 두 가지

- **`=~`는 "정규식으로 읽어라"는 스위치일 뿐 와일드카드가 아니다.**
  실제 매칭은 `.` `*` `|` 같은 **패턴 기호가 따옴표 안에서** 한다.
  `status="2.."`(스위치 없음)는 0개, `status=~"200"`(패턴 기호 없음)은 `=`과 동일하다.
- **`{}` 안의 콤마는 "그리고"다.**
  "GET 또는 POST"는 콤마로 못 쓰고 `method=~"GET|POST"`를 써야 한다.
  `|`를 따옴표 밖에 두면 `unexpected character inside braces: '|'` 에러가 난다.

### 해결

HTTP 상태코드는 항상 세 자리이므로 **`5..`가 `5.*`보다 의도를 정확히 표현한다.**

```promql
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
```

---

## 2. `by (le)`를 빼면 에러가 아니라 경고 + 빈 결과가 나온다

### 문제

```promql
histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])))
                         └── by (le)가 없다 ──┘
```

```json
{"status":"success", "data":{"resultType":"vector","result":[]},
 "warnings":["PromQL warning: bucket label \"le\" is missing or has a malformed value of \"\""]}
```

### 원인

`histogram_quantile()`은 **`le` 라벨을 보고 칸의 경계를 안다.**
`sum()`은 지정하지 않은 라벨을 전부 버리므로 `le`까지 날아가면 계산할 근거가 없다.
그런데 Prometheus는 이를 **쿼리 실패가 아니라 경고**로 처리한다.

### 해결

```promql
histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))
```

> 이건 그나마 `warnings` 필드에 단서가 있다.
> **브라우저 UI나 `curl` 출력에서 `warnings`를 습관적으로 확인할 것.**

---

## 3. sum 대신 avg를 써도 답이 같았다 — "정답처럼 보이는 우연"

### 문제

`histogram_quantile`에 `sum` 대신 `avg`를 넣었는데 **소수점 끝까지 같은 값**이 나왔다.

```
histogram_quantile(0.97, sum(rate(..._bucket[5m])) by (job, le))  →  hr-service 0.04781506375
histogram_quantile(0.97, avg(rate(..._bucket[5m])) by (job, le))  →  hr-service 0.04781506375
```

### 원인 — 실측으로 확인

`(job, le)` 그룹마다 합쳐지는 시계열 개수가 **모든 le에서 균일**했다(hr-service 2개, 나머지 1개).
따라서 `avg = sum ÷ 2`가 **모든 칸에 똑같이** 적용됐다.

```
sum:  le=0.001398101  0.017544 ... le=0.050331646  0.421054
avg:  le=0.001398101  0.008772 ... le=0.050331646  0.210527
      (전부 정확히 절반)
```

`histogram_quantile()`은 **전체(+Inf) 대비 비율만** 보므로,
모든 칸을 같은 수로 나눠도 비율이 안 바뀌어 답이 같다.

### 왜 그래도 틀렸나

`sum`은 "이 시간 이하로 끝난 요청이 총 몇 건"이라는 의미가 있지만,
`avg`는 "평균 몇 건"이라 **의미 자체가 없다.**
시계열 개수가 칸마다 달라지는 순간(엔드포인트마다 버킷 설정이 다른 경우 등) 분포 모양이 왜곡된다.

### 배운 점

> **값이 맞는다고 검증이 끝난 게 아니다.**

---

## 정리 — 같은 패턴이 반복된다

| | 증상 | 겉보기 |
|---|---|---|
| Git Bash 경로 변환 | 라벨 값이 Windows 경로로 바뀜 | `success` + 빈 결과 |
| `promtool check config` | 빈 `relabel_configs`가 통과 | `SUCCESS` |
| `by (le)` 누락 | quantile 계산 근거 소실 | `success` + 빈 결과 + 경고 |
| `status=~"5"` | 전체 일치 실패 | 빈 결과 → 오류율 0% |
| `avg` 오용 | 우연히 상쇄 | **값이 정확히 일치** |

> **관측 도구는 "틀렸다"고 잘 말해주지 않는다.**
> **빈 결과를 "데이터가 없나 보다"로 넘기지 않는 습관이 필요하다.**
> 필터를 하나씩 떼어보며 **어느 조건에서 사라지는지 좁힌다.**
