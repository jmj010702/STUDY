## Git Bash 경로 변환이 PromQL 쿼리를 조용히 망가뜨린다

### 문제 배경

Prometheus에 셸(`curl`)로 PromQL을 던져 특정 엔드포인트의 요청 수를 확인하려 했다.

### 문제

```
http_server_requests_seconds_count{job="hr-service"}                        → 4개
http_server_requests_seconds_count{job="hr-service",uri="/actuator/health"} → 0개   ← ?!
http_server_requests_seconds_count{job="hr-service",uri=~".*health"}        → 2개   ← 정규식은 됨
```

**에러가 나지 않는다.** `{"status":"success", ... "result":[]}` — **성공 응답에 빈 결과다.**

실제로 이 문제 때문에 "시계열이 사라졌다 → 앱이 죽었나?"까지 의심하며 시간을 썼다.

### 원인 분석

1. **라벨 값에 보이지 않는 문자가 있는지 hex로 확인**
   `2f 61 63 74 75 61 74 6f 72 2f 68 65 61 6c 74 68` = 정확히 `/actuator/health`. **깨끗함**
2. **정규식(`.*health`)은 매칭되는데 정확 일치는 안 됨**
   → **정규식에는 `/`가 없다**는 점에 착안
3. `MSYS_NO_PATHCONV=1` 설정 후 재시도 → **해결**

```
[기본]                  uri="/actuator/health"     → 0개
                        uri="/actuator/prometheus" → 0개
[MSYS_NO_PATHCONV=1]    uri="/actuator/health"     → 2개
                        uri="/actuator/prometheus" → 1개
```

**Git Bash(MSYS)는 인자에 `/`로 시작하는 문자열이 있으면 Windows 경로로 변환한다.**
`/actuator/health`가 `C:/Program Files/Git/actuator/health` 같은 값으로 바뀌어 Prometheus에 전달됐다.

### 왜 위험한가

| | |
|---|---|
| 문법 오류 | 에러 메시지가 뜬다 → 금방 고친다 |
| **이 함정** | **성공 응답 + 빈 결과** → "데이터가 없나 보다"로 오해하고 지나간다 |

### 해결 방법

```bash
export MSYS_NO_PATHCONV=1
```

- **브라우저(`localhost:9090/query`)에서는 발생하지 않는다** — 셸을 거치지 않으므로
- 같은 셸에서 여러 번 조회한다면 스크립트 상단에 한 번 export

### 배운 점

> **빈 결과가 나오면 "데이터가 없다"고 결론짓기 전에,
> 필터를 하나씩 떼어보며 어느 조건에서 사라지는지 좁힌다.**
>
> 이번엔 "정규식은 되는데 정확 일치는 안 된다"는 비대칭이 결정적 단서였다.
> **되는 케이스와 안 되는 케이스의 차이를 찾는 것**이 원인 추적의 기본이다.
