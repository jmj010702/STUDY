# Service Discovery와 relabel

## 1. 흐름 — 중간 단계가 하나 더 있다

```
[static_configs]
  targets에 적은 주소  ──────────────────▶  스크레이프

[Service Discovery]
  레지스트리(Eureka/k8s API)에 질의
      ↓
  인스턴스마다 target 생성 + __meta_ 라벨 부착   ← 원재료
      ↓
  relabel_configs 로 가공                        ← 요리
      ↓
  최종 __address__ 확정  ─────────────────▶  스크레이프
```

`static_configs`에서는 주소를 **우리가 직접 적었다.** SD에서는 **SD가 조립한 주소를 우리가 고쳐 쓴다.**

## 2. 언더스코어 2개로 시작하는 라벨은 버려진다

| 라벨 | 스크레이프 후 | 역할 |
|---|---|---|
| `__meta_*` | **사라짐** | SD가 제공하는 원재료 |
| `__address__` | **사라짐** | **실제 접속 주소** ★ |
| `__metrics_path__` | **사라짐** | 스크레이프 경로 |
| `__tmp_*` | **사라짐** | 임시용 예약 접두사 |
| `job`, `instance`, 그 외 | **남음** | 메트릭에 붙는 라벨 |

**`__meta_` 라벨을 그냥 두면 아무 데도 남지 않는다.** relabel로 일반 라벨에 복사해야 최종 메트릭에 붙는다.

## 3. `__address__` — 가장 중요한 특수 라벨

라벨이 20개 붙어 있어도 **"어디로 갈지"를 정하는 건 `__address__` 하나**다. 나머지는 참고 정보일 뿐이다.

```
__meta_eureka_app_instance_ip_addr  = 192.168.45.118     ← 안 봄
__meta_eureka_app_instance_hostname = DESKTOP-5TOO5J4    ← 안 봄
__address__ = DESKTOP-5TOO5J4:8080                       ← 여기로 간다 ★
```

`static_configs`의 `targets`도 사실은 이 라벨에 값을 넣는 것이었다.
그리고 `metrics_path: '/actuator/prometheus'`라고 쓴 YAML 설정이 `__metrics_path__` 라벨로 나타난다.

> **모든 스크레이프 설정은 결국 라벨이다.** 주소도, 경로도, 스킴도.
> 그래서 **라벨을 바꾸면 동작이 바뀐다** — 이것이 relabel이 강력한 이유다.

## 4. relabel 규칙의 구조

```yaml
- source_labels: [라벨A, 라벨B]   # ① 읽을 라벨들
  separator: ':'                  # ② 여러 개면 이걸로 이어붙임 (기본 ";")
  regex: '(.*)'                   # ③ 매칭할 정규식 (기본 "(.*)")
  target_label: __address__       # ④ 쓸 대상 라벨
  replacement: '$1'               # ⑤ 쓸 값. $1은 캡처 그룹 (기본 "$1")
  action: replace                 # ⑥ 동작 (기본 replace)
```

동작 순서:
```
① [ip_addr, port] 를 읽는다        "192.168.45.118"  "8080"
② separator ':' 로 이어붙인다       "192.168.45.118:8080"
③ regex "(.*)" 로 매칭 → $1
⑤ replacement "$1" 을
④ target_label "__address__" 에 쓴다
                                    __address__ = "192.168.45.118:8080"
```

기본값이 맞아떨어지면 `regex`·`replacement`·`action`은 생략한다 — **실제로 쓸 건 3줄뿐인 경우가 많다.**

### ⚠️ regex는 양끝이 자동 앵커링된다

`regex: 'hr'`는 `hr`을 **포함**하는 것이 아니라 **정확히 `hr`인 것**만 매칭한다.
부분 매칭은 `.*hr.*`처럼 명시해야 한다. `keep`/`drop`을 쓸 때 걸리기 쉬운 지점.

### action 종류

| 분류 | action |
|---|---|
| 변환 | `replace`(기본) · `lowercase` · `uppercase` |
| 필터 | `keep` · `drop` · `keepequal` · `dropequal` |
| 라벨 조작 | `labelmap` · `labeldrop` · `labelkeep` |
| 기타 | `hashmod` |

> `labeldrop`/`labelkeep` 사용 시 **제거 후에도 메트릭이 고유하게 식별되는지** 확인해야 한다.
> 라벨을 지워 두 시계열이 같아지면 충돌한다.

## 5. Eureka SD가 주는 라벨 (실측)

```
__address__                          : 192.168.45.118:57488   ← SD가 조립 (hostname:port)
__metrics_path__                     : /actuator/prometheus
__meta_eureka_app_name               : COLLABORATION-SERVICE
__meta_eureka_app_instance_hostname  : 192.168.45.118
__meta_eureka_app_instance_ip_addr   : 192.168.45.118
__meta_eureka_app_instance_port      : 57488
__meta_eureka_app_instance_id        : DESKTOP-5TOO5J4:collaboration-service:0
__meta_eureka_app_instance_status    : UP
__meta_eureka_app_instance_healthcheck_url : http://192.168.45.118:57488/actuator/health
__meta_eureka_app_instance_vip_address     : collaboration-service
(그 외 homepage_url, statuspage_url, country_id, datacenterinfo, secure_port 등)
```

> **라벨 이름을 추측하지 말 것.** 공식 문서에 목록이 안 나오는 경우가 있으므로,
> **relabel 없는 job을 먼저 붙여 `/api/v1/targets`의 `discoveredLabels`로 실물을 확보**하는 편이 확실하다.

## 6. 실제로 쓴 규칙 2개

```yaml
    relabel_configs:
      # ① __address__ 를 hostname:port 대신 ip_addr:port 로
      - source_labels: [__meta_eureka_app_instance_ip_addr, __meta_eureka_app_instance_port]
        separator: ':'
        target_label: __address__
      # ② job 라벨을 Eureka 앱 이름(소문자)으로
      - source_labels: [__meta_eureka_app_name]
        target_label: job
        action: lowercase
```

### ① 없이는 api-gateway가 실패한다

SD가 만드는 `__address__`는 `hostname:port`인데, api-gateway만 `prefer-ip-address` 설정이 없어 hostname이 PC 이름이었다.

```
dial tcp: lookup DESKTOP-5TOO5J4 on 127.0.0.11:53: no such host
                                    ^^^^^^^^^^^^ Docker 내장 DNS
```

### ② 없이는 4개 서비스가 한 job으로 뭉친다

```
job_name: 'peoplecore-services'   →  SD가 4개 발견  →  4개 전부 job="peoplecore-services"
```

**`job` 라벨 값은 `job_name`에 쓴 그 문자열**이다. SD가 4개를 발견하든 40개를 발견하든 전부 같은 job이 된다.
`static_configs`에서는 서비스마다 job을 따로 썼기 때문에 자연히 갈려 있었을 뿐이다.

## 7. 주소 설계 — 무엇을 고정하고 무엇을 동적으로 둘 것인가

```
내가 파일에 직접 적는 값     → host.docker.internal   (IP가 바뀌어도 안 흔들림)
레지스트리가 동적으로 주는 값 → ip_addr                (IP가 바뀌면 자동 갱신)
```

Eureka **접속** 주소에 IP를 박으면 공유기가 IP를 바꿀 때 SD 자체가 죽는다.
반대로 target 주소는 실시간 값이라 IP 변경을 자동으로 따라간다.

## 8. SD가 닿지 않는 곳은 static으로 남긴다

```yaml
  # Eureka 서버 — 자기를 자기한테 등록하지 않으므로 SD로는 절대 못 찾는다
  - job_name: 'eureka-server'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8761']
```

> **서비스 디스커버리는 레지스트리에 등록된 것만 찾는다. 레지스트리 자신은 거기 없다.**
> **가장 중요한 단일 장애점을 관측 밖에 두면 안 된다.**
> k8s에서 kube-apiserver를 `kubernetes_sd`로 찾을 수 없는 것과 같은 구조다.

## 9. `hostName`과 `instanceId`는 별개다

`prefer-ip-address: true` 적용 후:

| 필드 | 전 | 후 |
|---|---|---|
| `hostName` | `DESKTOP-5TOO5J4` | **`192.168.45.118`** |
| `instanceId` | `DESKTOP-5TOO5J4:api-gateway:8080` | **변화 없음** |

**`prefer-ip-address`는 hostName만 바꾼다.**
Prometheus의 `instance` 라벨은 instanceId에서 오므로, **재시작해도 `instance`가 유지되어 시계열이 이어진다.**

## 10. SD의 대가

→ [SD 전환 후 죽은 대상이 사라진다](트러블슈팅/SD전환후_죽은대상이_사라진다.md)

| | `static_configs` | `eureka_sd_configs` |
|---|---|---|
| 앱 정지 시 | **target이 남아 `up=0` 유지** | `up=0` 잠시 → **소멸** |
| `up == 0` 알림 | 계속 감지됨 | **일정 시간 후 무력화** |
| 포트 변경 대응 | 수동 수정 | 자동 |
| 재시작 시 시계열 | `instance` 변경으로 **끊김** | instanceId 유지로 **이어짐** |

**SD가 일방적으로 우월한 것이 아니다.** 운영 부담을 줄인 대신 "사라짐"의 감지가 어려워졌다.

## 11. 검증 순서 — `promtool`은 문법만 본다

```
① promtool check config      → YAML이 깨졌나 (최소 조건)
② docker kill -s HUP          → 반영
③ up / job 라벨 / scrapeUrl   → 의도대로인가 (진짜 검증)
```

**빈 `relabel_configs:`가 SUCCESS로 통과한 적이 있다.** YAML에서 키 뒤에 값이 없으면 null이 되고
Prometheus는 "규칙 0개"로 정상 처리한다. **"유효하다"와 "의도대로 동작한다"는 별개다.**

**리로드 직후 확인은 최소 한 번의 `scrape_interval`을 기다린 뒤에 한다.**
삭제된 target의 마지막 샘플이 최대 5분간 instant query에 남아 있어, 성급히 조회하면 **삭제한 job이 그대로 보인다.**
