# 개발용 MCP 서버 정리

Claude Code에 붙여서 쓸 만한 MCP 서버를 조사해 정리한 것. (2026-08 기준)

## MCP가 뭔가

**Model Context Protocol.** LLM이 외부 시스템에 접근하는 방식을 표준화한 프로토콜.
2024년 11월 Anthropic이 공개했고, 2025년 초 OpenAI·Google DeepMind가 채택하면서 사실상 표준이 됐다.

핵심은 **"AI가 쓸 수 있는 도구를 꽂아 넣는 규격"**이라는 점이다.

```
Grafana MCP 붙임    → Claude가 Grafana에 직접 쿼리를 날린다
PostgreSQL MCP 붙임 → 실제 테이블 스키마를 보고 답한다
안 붙이면           → 추측으로 답한다
```

이 차이가 생각보다 크다.

---

## 등록하면 자동으로 쓰이나 → 그렇다

**매번 "MCP 써서 해줘"라고 지시할 필요 없다.**

MCP 서버를 등록하면 그 서버의 도구들이 **내장 도구(파일 읽기, Bash, 웹 검색 등)와 같은 목록에 합쳐진다.**
Claude 입장에서는 둘의 구분이 없고, 작업에 필요하다고 판단하면 알아서 호출한다.

### 다만 자동이 완벽하지는 않다

| 상황 | 무슨 일이 생기나 |
|---|---|
| **첫 호출 시** | 권한 프롬프트가 뜬다. 한 번 허용하면 이후엔 안 뜸 |
| **도구를 안 쓰고 답해버림** | "이 메트릭 타입 뭐야?"에 Grafana MCP로 확인 안 하고 그럴듯하게 답할 수 있다 |
| **비슷한 도구가 많을 때** | 엉뚱한 걸 고른다 |

**두 번째가 제일 위험하다.** "LLM이 없는 메트릭을 지어낸다"는 문제와 **정확히 같은 실패**이기 때문이다.
**도구가 있는데도 안 쓰면 도구가 없는 것과 결과가 같다.**

### 자동 사용률을 높이는 방법 — CLAUDE.md에 못 박기

```markdown
- PromQL을 제안하기 전에 반드시 Grafana MCP로 실행해서
  데이터가 반환되는지 확인한다. 확인하지 않은 쿼리는 제안하지 않는다.
- 메트릭 타입은 추측하지 말고 Grafana MCP의 메타데이터 조회로 확인한다.
```

CLAUDE.md는 매 세션 컨텍스트에 들어가므로 Claude가
**"이 상황에서는 저 도구를 써야 한다"를 알고 시작한다.** 도구만 꽂아두는 것보다 훨씬 확실하다.

> **정리**: 꽂아두면 알아서 쓴다. 다만 *반드시* 써야 하는 순간은 CLAUDE.md에 적어둬야 확실해진다.

---

## ⚠️ 선정 기준 — 개수를 늘리면 오히려 나빠진다

> **3~6개가 적정. 6개를 넘으면 도구 선택 신뢰도가 떨어진다.**

이유 두 가지:

1. MCP 도구 정의는 전부 컨텍스트를 차지한다. 도구가 많을수록 실제 작업에 쓸 컨텍스트가 줄어든다
2. 비슷한 기능의 도구가 여러 개면 모델이 엉뚱한 걸 고른다

**"좋아 보이는 걸 전부 깐다"가 아니라 "지금 하는 일에 맞는 걸 골라 넣고, 안 쓰면 뺀다"가 맞다.**

---

## 1순위 — 지금 하는 일에 직접 맞물리는 것

| MCP | 무엇을 하나 | 왜 필요한가 |
|---|---|---|
| **Grafana** (공식) | 대시보드 조회·생성, **Prometheus/Loki 쿼리 및 메타데이터 조회**, 알림 규칙 | 아래 별도 설명 |
| **PostgreSQL** | 스키마 조회, 읽기 전용 쿼리 | JPA 엔티티 설계 시 실제 테이블 구조 확인 |
| **Context7** | 라이브러리 **버전별** 최신 문서·코드 예제 | Spring Boot 3.x, Spring Batch, Quartz처럼 버전 간 API가 갈리는 것들 |
| **GitHub** | 이슈·PR·Actions·저장소 관리 | 포트폴리오 저장소 운영 |

### Grafana MCP가 왜 1순위인가

"검증하지 않은 쿼리로 산출물을 만들지 않는다"는 원칙을 지키려면 지금은 왕복이 필요하다.

```
Claude가 PromQL 제안 → 내가 Grafana에 붙여넣어 실행 → 결과를 Claude에게 알려줌 → 판단
```

Grafana MCP를 붙이면 이게 사라진다.

```
Claude가 직접 쿼리 실행 → 데이터 나오는 것만 골라서 제안
```

**문서상의 규칙이 실제로 강제되는 절차로 바뀐다.** 이게 핵심이다.

메트릭 타입 조회(`/api/v1/metadata`)도 같다. counter인지 gauge인지 histogram인지를 직접 확인하고 쿼리 형태를 맞출 수 있다.

- counter → `rate()`로 감싸야 함
- histogram → `histogram_quantile()` 사용 가능
- gauge → 그대로 사용

**단, 읽기 전용 권한으로 쓰는 걸 권장.** 검증 없이 만들어진 패널이 늘어날 위험이 있다.

---

## 2순위 — 범용 개발

| MCP | 용도 | Claude Code에서의 판단 |
|---|---|---|
| **Chrome DevTools** | 성능 트레이싱, Web Vitals, 네트워크 디버깅 | 프론트엔드 작업 시 유용 |
| **Playwright** | 브라우저 자동화, E2E 테스트 | 프론트엔드 작업 시 유용 |
| **Filesystem** (공식) | 지정 경로 파일 접근 | ⚠️ 내장 도구와 **중복** |
| **Git** (공식) | 커밋 히스토리·diff 조회 | ⚠️ 내장 도구와 **중복** |
| **Memory** (공식) | 지식 그래프 기반 영속 메모리 | ⚠️ 파일 기반 메모리와 **중복** |
| **Fetch** (공식) | 웹 페이지 → 마크다운 변환 | ⚠️ WebFetch 내장과 **중복** |
| **Sequential Thinking** (공식) | 단계적 추론 보조 | 최신 모델은 자체 추론이 있어 체감 이득 적음 |

**중복 표시한 것들은 Claude Code에 넣지 말 것.**
공식 레퍼런스 서버들은 원래 "MCP 기능을 보여주는 교육용 예제"로 만들어진 것이고,
Claude Code에는 같은 기능이 이미 내장 도구로 들어 있다.
넣으면 도구 목록만 늘어나서 위의 선택 정확도 문제가 생긴다.
**Claude Desktop처럼 내장 도구가 없는 클라이언트에서는 의미가 있다.**

### Playwright vs Chrome DevTools — 역할이 다르다

- **Playwright** = 브라우저를 **운전**한다 (자동화, 테스트 시나리오 실행)
- **Chrome DevTools** = 브라우저를 **진단**한다 (성능 트레이싱, 렌더링 지연 분석)

성능 트레이싱은 Chrome DevTools MCP에만 있다.

---

## 3순위 — 스택상 있지만 지금은 이른 것

| MCP | 언제 필요해지나 |
|---|---|
| **Elasticsearch** | 로그 분석 단계 진입 시 |
| **Kafka** | 토픽·컨슈머 그룹 조회. 커뮤니티 구현체라 품질 편차 있음 |
| **Kubernetes** | EKS 배포 단계. 로컬 Docker Compose면 불필요 |
| **Docker** | Bash로 `docker ps` 하는 것과 큰 차이 없음 |

---

## 등록 방법

```bash
# 1. 원격 서버 (URL만 등록) — 2026년 기본값
claude mcp add --transport http <이름> <URL>

# 2. 로컬 서버 (내 PC에서 프로세스를 띄움)
claude mcp add <이름> [--env KEY=VALUE] -- <실행명령>
```

`--` 뒤가 실제로 실행될 명령이다. 그 앞은 Claude Code에 주는 옵션.

### 스코프 — 어디에 등록할 것인가

| 스코프 | 플래그 | 저장 위치 | 적용 범위 |
|---|---|---|---|
| **user** | `-s user` | `~/.claude.json` | **모든 프로젝트** |
| **project** | `-s project` | 프로젝트 루트 `.mcp.json` | 그 프로젝트만. **git에 커밋되어 팀과 공유됨** |
| **local** (기본) | 없음 | `~/.claude.json`의 프로젝트별 항목 | 그 프로젝트에서 나만 |

판단 기준:
- Context7, GitHub처럼 **어디서나 쓰는 건 `-s user`**
- Grafana, PostgreSQL처럼 **이 프로젝트 전용은 `-s project`**
- ⚠️ **`-s project`는 `.mcp.json`이 커밋된다. API 키·토큰을 직접 넣지 말 것.**
  환경변수 참조(`${GRAFANA_API_KEY}`)로 두고 `.env`로 주입한다

### 관리 명령

```bash
claude mcp list                    # 등록 목록 + 연결 상태
claude mcp get <이름>              # 상세 (스코프, URL, 헤더, 연결 실패 원인)
claude mcp remove <이름> -s user   # 제거 (등록한 스코프를 지정해야 함)
```

`/mcp` 슬래시 명령으로도 대화 중에 상태를 볼 수 있고, OAuth 인증도 여기서 한다.

### 설치 예시

> ⚠️ MCP 생태계는 변화가 빠르다. 실제 설치 전 각 공식 저장소의 최신 문서를 확인할 것.

```bash
# Grafana — Service Account Token 먼저 발급
claude mcp add grafana \
  --env GRAFANA_URL=http://localhost:3000 \
  --env GRAFANA_API_KEY=<token> \
  -- uvx mcp-grafana

# Context7 (원격, HTTP)
claude mcp add --transport http context7 https://mcp.context7.com/mcp

# GitHub (원격, OAuth)
claude mcp add --transport http github https://api.githubcopilot.com/mcp/

# Playwright
claude mcp add playwright -- npx @playwright/mcp@latest

# Chrome DevTools
claude mcp add chrome-devtools -- npx chrome-devtools-mcp@latest
```

**원격(remote) MCP가 2026년 기본값이다.**
GitHub, Vercel, Linear, Notion, Supabase, Stripe, Figma 등이 OAuth 기반 호스팅 엔드포인트를 제공해서
로컬 설치 없이 URL만 등록하면 된다.

---

## 트러블슈팅 — 로컬 MCP `ConnectionRefused`

### 문제

Obsidian MCP가 등록은 되어 있는데 연결이 안 됐다. 설정(user 스코프, HTTP, Bearer 토큰)은 정상이었다.

### 원인

```powershell
Get-Process -Name Obsidian                             # → 결과 없음 (앱이 안 떠 있음)
Get-NetTCPConnection -LocalPort 27123 -State Listen    # → 리스닝 없음
```

**Obsidian 앱이 실행 중이 아니라서 27123 포트를 아무도 듣고 있지 않았다.**

### 해결

앱을 켠 뒤 `claude mcp list` → `✔ Connected`. **플러그인 설정은 건드릴 필요 없었다.**

> **교훈: 로컬 MCP가 `ConnectionRefused`면 설정을 의심하기 전에 대상 앱이 떠 있는지부터 본다.**
>
> 로컬 MCP 서버는 **내 PC의 다른 프로그램에 붙는 것**이라 그 프로그램이 꺼져 있으면 연결이 안 된다.
> 원격 MCP는 항상 떠 있으니 이런 문제가 없다.

> ⚠️ `claude mcp get <이름>`을 실행하면 **Authorization Bearer 토큰이 그대로 출력된다.**
> 화면 공유·스크린샷·노트에 붙여넣을 때 주의할 것.

---

## 출처

- [modelcontextprotocol/servers (공식 레퍼런스 서버)](https://github.com/modelcontextprotocol/servers)
- [grafana/mcp-grafana](https://github.com/grafana/mcp-grafana)
- [Grafana MCP 공식 문서](https://grafana.com/docs/grafana/latest/developer-resources/mcp/introduction/)
- [Chrome DevTools MCP — Chrome for Developers](https://developer.chrome.com/blog/chrome-devtools-mcp)
- [PulseMCP 서버 디렉터리](https://www.pulsemcp.com/servers)
