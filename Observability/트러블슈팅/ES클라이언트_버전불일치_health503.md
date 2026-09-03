## ES 클라이언트가 서버보다 최신이라 health가 상시 503이었다

### 문제 배경

작업 환경을 Mac으로 옮기고 4개 서비스를 처음부터 다시 띄우는 중이었다.
**찾으려던 게 아니라, 서비스를 하나씩 기동해보다가 로그에서 마주쳤다.**

### 문제

search-service의 `/actuator/health`가 **프로젝트 시작부터 계속 503**이었다.

```
TransportException: node: http://localhost:9200/, status: 200, [es/cluster.health] Failed to decode response
                                                  ^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^
Caused by: MissingRequiredPropertyException: Missing required property 'HealthResponse.unassignedPrimaryShards'
```

**`status: 200`이 핵심 단서다.** 요청은 성공했고 서버는 정상 응답했다.
실패한 곳은 **클라이언트가 그 응답을 객체로 변환하는 단계**다.
이 한 줄로 연결 문제·서버 다운 가설을 즉시 제거할 수 있었다.

### 원인 분석 — 두 버전이 서로 다른 곳에서 정해진다

| | 버전 | 정해지는 곳 |
|---|---|---|
| ES 서버 | **8.13.0** | `Dockerfile.elasticsearch`에 **손으로 고정** |
| elasticsearch-java 클라이언트 | **8.18.8** | **Spring Boot 3.5.13의 BOM이 자동 결정** |

```bash
curl -s http://localhost:9200/_cluster/health
→ "unassigned_shards":1, "delayed_unassigned_shards":0
  (unassigned_primary_shards 없음)     ← 8.13 서버는 이 필드를 안 보낸다
```

8.18 클라이언트는 이 필드를 **필수(required)로 선언**해뒀다. 없으면 예외를 던진다.

> **이 구조가 문제의 본질이다.** 한쪽은 사람이 파일에 박고, 다른 한쪽은 프레임워크가 정한다.
> **둘이 맞는지 아무도 검사하지 않는다.** Spring Boot를 올리는 순간 클라이언트만 조용히 따라 올라간다.

**언제부터였나 — 처음부터**

`gradle.properties`의 `springBootVersion` 변경 이력을 봤더니 **한 번도 바뀐 적 없이 계속 3.5.13**이었다.
최근에 깨진 게 아니라 **처음부터 어긋나 있었다.** Windows 환경에서도 동일하게 503이었을 것이다.

### 왜 아무도 몰랐나 — "조용한 고장"

| | |
|---|---|
| 앱이 안 죽는다 | health indicator만 실패한다. Tomcat도 Kafka도 정상, **Eureka 등록도 정상** |
| 검색 기능은 동작한다 | 깨진 것은 `cluster.health` API 응답 하나. `_search`는 정상 |
| 메트릭도 정상 | `/actuator/prometheus`는 200. **관측 지표상으로도 안 보인다** |
| **`show-details`가 없다** | 응답이 `{"status":"DOWN"}` 한 줄뿐이라 **무엇이 DOWN인지 알 수 없다** |

**아무도 그 화면을 안 보면 고장은 존재하지 않는 것과 같다.**

### 해결 방법

서버를 클라이언트에 맞춰 올렸다.

```diff
- FROM docker.elastic.co/elasticsearch/elasticsearch:8.13.0
+ FROM docker.elastic.co/elasticsearch/elasticsearch:8.18.8    # Dockerfile.elasticsearch

- image: docker.elastic.co/kibana/kibana:8.13.0
+ image: docker.elastic.co/kibana/kibana:8.18.8                # docker-compose.yml
```

**Kibana가 딸려 왔다.** Kibana는 ES와 버전이 맞아야 기동되므로, ES만 올렸으면 이번엔 Kibana가 깨졌을 것이다.
**버전은 혼자 안 움직인다.**

**서버를 올릴지 클라이언트를 내릴지** — 서버를 올렸다.
클라이언트 버전은 **Spring Boot가 결정**하므로 Boot를 올릴 때마다 또 어긋난다.
**자동으로 움직이는 쪽에 손으로 고정한 쪽을 맞추는 것**이 지속 가능하다.

### 검증 — 원인 지점이 직접 바뀌었다

| | 전 | 후 |
|---|---|---|
| ES 버전 | 8.13.0 | **8.18.8** |
| `unassigned_primary_shards` | **응답에 없음** | **`0`으로 존재** |
| search-service `/actuator/health` | **503 `{"status":"DOWN"}`** | **200 `{"status":"UP"}`** |

### 주의사항

- 기존 `es-data` 볼륨은 8.13 기준이라 8.18로 기동하면 **자동 업그레이드되고 되돌릴 수 없다.** 팀 공유 필수
- `show-details: always`를 나머지 서비스에도 넣을지 검토 필요

### 배운 점

1. **health check 실패 ≠ 대상이 죽은 것.** DOWN에 이르는 경로는 세 가지다 — 대상이 죽음 / 연결은 되는데 거절 / **응답은 성공했는데 해석 실패**
2. **에러 메시지의 `status: 200`처럼 "성공했다"는 흔적을 놓치지 말 것.** 가설을 절반으로 줄여준다
3. **버전이 두 곳에서 따로 정해지는 구조를 찾아둘 것.** 사람이 박은 값과 프레임워크가 정하는 값이 만나는 지점은 언젠가 어긋난다
