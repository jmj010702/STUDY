# Spring Boot Actuator

**Spring Boot 앱의 내부 상태를 HTTP로 들여다보게 해주는 모듈.** 이름 그대로 *actuator(작동기)* — 기계를 들여다보고 조작하는 손잡이.

## 1. 엔드포인트

| 엔드포인트 | 하는 일 |
|---|---|
| `/actuator/health` | 앱 + DB·Redis·Kafka 연결 상태 |
| `/actuator/metrics` | 측정된 지표 |
| `/actuator/prometheus` | 메트릭을 Prometheus 형식으로 |
| `/actuator/env` | 환경변수와 설정값 **전부** |
| `/actuator/beans` | 등록된 스프링 빈 목록 |
| `/actuator/mappings` | URL 매핑 전체 |
| `/actuator/loggers` | **로그 레벨 조회 + 런타임 변경** |
| `/actuator/threaddump` | 스레드 덤프 |
| `/actuator/heapdump` | **힙 덤프 파일 다운로드** |

**조회만이 아니라 변경도 된다.** `/actuator/loggers`는 재시작 없이 로그 레벨을 바꾼다. 운영 중 장애 시 특정 패키지만 DEBUG로 올렸다 내리는 식으로 유용하지만, **외부인이 건드릴 수 있으면 안 된다.**

## 2. 코드는 라이브러리 안에 있다

```
~/.gradle/caches/.../spring-boot-actuator-3.5.13.jar
  └─ org/springframework/boot/actuate/health/HealthEndpoint.class

~/.gradle/caches/.../spring-boot-actuator-autoconfigure-3.5.13.jar
  ├─ .../jdbc/DataSourceHealthContributorAutoConfiguration.class    ← DB 체크
  ├─ .../data/redis/RedisHealthContributorAutoConfiguration.class   ← Redis 체크
  └─ .../system/DiskSpaceHealthContributorAutoConfiguration.class   ← 디스크 체크
```

**우리 프로젝트 소스에는 health 관련 자바 코드가 한 줄도 없다.** 남이 만든 코드를 jar로 받아 **켠 것**이다.

### 자동 설정 흐름

```
1. build.gradle에 의존성 한 줄
        ↓
2. Gradle이 jar를 받아 클래스패스에 넣음
        ↓
3. 앱이 뜰 때 Spring Boot가 클래스패스를 훑음
   "HealthEndpoint 클래스가 있네"      → 등록
   "DataSource 빈도 있네"             → DB 체크 추가
   "RedisConnectionFactory도 있네"    → Redis 체크 추가
        ↓
4. /actuator/health 주소가 열림
```

**health가 무엇을 체크할지도 자동으로 결정된다.** 설정한 적 없는데 db·redis·mail·diskSpace가 나온 이유다.

---

## 3. 의존성과 설정은 별개다

| 상태 | `/actuator/health` | `/actuator/metrics` |
|---|---|---|
| 의존성 없음 | 없음 | 없음 |
| **의존성만 추가** | **200** `{"status":"UP"}` | **여전히 막힘** |
| **설정 추가** | 구성요소별 상세 | — |

> **의존성 추가는 "쓸 수 있게 만드는 것"이고, 무엇을 열지는 별도로 정해야 한다.**

```
앱 내부에 존재:  health, metrics, env, beans, heapdump, loggers, ...
                    ↓ include: health,prometheus
HTTP로 접근 가능: health, prometheus
```

**기본값은 `health` 하나.** 안 쓰면 `/actuator/prometheus`는 앱 안에 있어도 밖에서 못 부른다 — 404가 아니라 **"존재하지만 닫힘"**.

### ⚠️ `endpoints` vs `endpoint` — 실제로 걸린 함정

```yaml
management:
  endpoints:          # 복수 — 문을 여닫는 관리
    web:
      exposure:
        include: health,prometheus
  endpoint:           # 단수 — 개별 문의 동작 방식
    health:
      show-details: always
```

| 키 | 뜻 | 아래에 오는 것 |
|---|---|---|
| `endpoint**s**` (복수) | 전체 노출 관리 | `web.exposure.include` |
| `endpoint` (단수) | 개별 엔드포인트 동작 | `health`, `info` 등 엔드포인트 이름 |

**`s` 하나 차이이고, 틀리면 에러 없이 조용히 무시된다.**
실제로 `management.endpoint.web.exposure`로 잘못 써서 IDE가 밑줄을 그었다.
**설정을 넣었는데 아무 일도 안 일어나면 여기부터 의심할 것.**

### 주요 설정

| 설정 | 기본값 | 의미 |
|---|---|---|
| `endpoints.web.exposure.include` | `health` | HTTP로 열 엔드포인트 목록 |
| `endpoint.health.show-details` | `never` | `always`면 구성요소별 상태 표시 (운영은 `when-authorized` 권장) |

> `show-details`가 없으면 응답이 `{"status":"DOWN"}` 한 줄뿐이라 **무엇이 DOWN인지 알 수 없다.** 실제로 이 설정이 없던 서비스는 원인 특정에 훨씬 오래 걸렸다.

---

## 4. health의 전체 상태 판정

**하나라도 DOWN이면 전체가 DOWN**이고 HTTP 503이 된다.
그래서 k8s가 이 주소 하나만 보고 파드를 재시작할 수 있다.

이 문장을 개념으로만 알고 있었는데 실제로 겪었다 → [mail 하나가 전체를 DOWN으로 만든다](트러블슈팅/health_mail_하나가_전체를_DOWN.md)

### health group으로 나누는 것이 정석

```yaml
management:
  endpoint:
    health:
      group:
        liveness:
          include: livenessState              # "프로세스가 살아있나"
        readiness:
          include: readinessState,db,redis    # "요청 받을 준비 됐나"
```

| 경로 | 묻는 것 | 실패 시 k8s 동작 |
|---|---|---|
| `/actuator/health/liveness` | 프로세스가 살아있나 | **재시작** |
| `/actuator/health/readiness` | 트래픽 받을 준비 됐나 | **트래픽만 차단** (재시작 안 함) |

**`mail` 같은 외부 의존은 둘 중 어디에도 속하지 않는다.** 메일을 못 보내도 출퇴근 조회는 정상 동작한다.

---

## 5. health check 실패 ≠ 대상이 죽은 것

DOWN에 이르는 경로는 세 가지다.

| 경로 | 예시 |
|---|---|
| 대상이 정말 죽음 | DB 컨테이너 정지 |
| 연결은 되는데 **거절** | SMTP가 초당 10건을 거절 |
| **응답은 성공했는데 해석 실패** | ES 클라이언트가 서버보다 최신이라 필드를 못 찾음 |

세 번째가 제일 헷갈린다 → [ES 클라이언트 버전 불일치](트러블슈팅/ES클라이언트_버전불일치_health503.md)

---

## 6. 엔드포인트 보안 — `include: '*'`를 쓰면 안 되는 이유

> ⚠️ 아래는 실제로 열어보고 확인한 것이 아니라 원리에 근거한 설명이다.

| 엔드포인트 | 노출되는 것 | 위험도 |
|---|---|---|
| `/actuator/env` | 설정값 전체 | 중 — 이름 기준 마스킹이 있으나 완전하지 않음 |
| `/actuator/heapdump` | **메모리 전체를 파일로** | **최상** — 마스킹 개념 자체가 없음 |
| `/actuator/loggers` | 로그 레벨 런타임 변경 | 중 |

### env의 마스킹은 완전하지 않다

Spring Boot는 키 이름에 `password`·`secret`·`key`·`token` 등이 들어가면 `******`로 가린다. 그러나 **이름 기준**이라 이런 건 안 가려진다:

- `discord.batch-webhook` → URL 자체가 인증 수단인데 마스킹 안 됨
- `spring.datasource.url` → 내부 DB 주소 노출
- `coolsms.from-number` → 발신번호 노출

### heapdump가 가장 위험한 이유

힙 = 앱이 쓰는 **메모리 영역 전체**. 지금 다루는 모든 객체가 들어 있다.
인사 시스템이라면 주민번호·계좌·급여 이력이 거기 있다.

### `.env`로도 못 막는다 — 위협 모델을 구분할 것

```
.env 파일 → 환경변수 → String 객체(힙) → HikariCP가 DB 접속에 사용
                          ↑ 여기서 평문으로 존재
```

| 위협 | `.env`가 막나 |
|---|---|
| git 저장소에 비밀번호 커밋 | ✅ |
| 소스코드 공유 | ✅ |
| **힙 덤프 유출** | ❌ |
| 실행 중 서버 침입 | ❌ |

**`.env`는 "코드와 비밀을 분리"하는 도구이지 "실행 중 메모리를 보호"하는 도구가 아니다.**
마스킹도 **응답을 만들 때 가리는 것**이지 메모리에서 지우는 게 아니다. env는 가려도 heapdump에는 원본이 나온다.

게다가 자바 `String`은 불변이라 사용 후에도 GC 전까지 남는다. 보안 민감한 곳에서 `char[]`를 쓰는 이유(`PasswordAuthentication.getPassword()`가 `char[]`를 반환하는 것도 이 때문). 다만 **Spring 설정값은 전부 `String`이라 이 방어가 불가능하다.**

### 실무 대응 — 값을 숨기는 대신 접근을 막는다

| 대응 | 내용 |
|---|---|
| **heapdump를 아예 안 엶** | `include`에 절대 넣지 않음 |
| **Actuator 별도 포트** | `management.server.port: 9090` → 방화벽에서 그 포트만 차단 |
| 내부망 한정 | k8s에서 Actuator 포트를 Service로 노출 안 함 |
| 인증 필수화 | Spring Security로 `/actuator/**` 보호 |

> `include: '*'`가 위험한 진짜 이유는 "정보가 좀 새서"가 아니라
> **모든 비밀 관리 노력을 한 번에 무력화하기 때문**이다.

---

## 7. MSA에서는 서비스마다 넣어야 한다

Actuator는 **앱 프로세스 안에서 도는 것**이라 서비스 하나에 넣었다고 다른 서비스가 관측되지 않는다.
4개 서비스면 4개 전부에 의존성 + 설정이 들어가야 한다. `show-details` 같은 설정이 서비스마다 다르면 **원인 특정 난이도가 서비스마다 달라진다.**
