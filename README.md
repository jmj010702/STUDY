# 작업을 하다가 막히는 부분이 있을 때 어떤 점이 어려웠는지 적는 곳 나중에 같은 문제 발생시에 보고 해결할 수 있도록

---

## 폴더 구성

| 폴더 | 내용 |
|---|---|
| [Observability](Observability/) | Prometheus·PromQL·Actuator 등 관측 체계. 트러블슈팅 5건 포함 |
| [AI_STUDY](AI_STUDY/) | RAG·LangChain·LLM. 재색인 멱등성, 프롬프트 인젝션 방어 |
| [Batch_STUDY](Batch_STUDY/) | Spring Batch, Quartz JDBC 클러스터링, 레디스 분산락의 한계 |
| [Terraform_Study](Terraform_Study/) | IaC 기초, HCL 문법, 동작 흐름 |
| [Tooling](Tooling/) | 개발 도구가 조용히 속인 케이스 + MCP 서버 정리 |
| [Python](Python/) | 파이썬 유틸리티 프로그램 개발 기록 |

## 루트 문서

- [EKS_ingress.md](EKS_ingress.md) — WebSocket/SSE 환경에서 sticky session vs IP 해시(consistent hashing)
- [eks_restClient.md](eks_restClient.md) — EKS에서 `@LoadBalanced`가 일으킨 산발적 `SERVICE_UNAVAILABLE`
- [Elasticsearch_AI.md](Elasticsearch_AI.md) — ES 인덱스·샤드·역인덱스, 벡터 검색 기초(BM25/kNN/RRF)
- [EKS_WebSocket_연결끊김.md](EKS_WebSocket_연결끊김.md) — 배포 환경에서만 WebSocket이 주기적으로 끊긴 문제 ⚠️ 기록 일부 유실
- [테이블_파티셔닝_근태출근.md](테이블_파티셔닝_근태출근.md) — 쿼리 튜닝을 기각하고 파티션 테이블로 재설계한 판단 ⚠️ 기록 일부 유실

---

## 빠르게 찾기 — 증상별 색인

같은 증상을 또 만났을 때 여기서 먼저 찾을 것.

| 증상 | 문서 |
|---|---|
| 쿼리가 **에러 없이 빈 결과**를 반환한다 | [PromQL 함정 모음](Observability/트러블슈팅/PromQL_함정_모음.md) · [Git Bash 경로 변환](Tooling/GitBash_경로변환.md) |
| `/actuator/health`가 503인데 앱은 멀쩡하다 | [mail 하나가 전체를 DOWN](Observability/트러블슈팅/health_mail_하나가_전체를_DOWN.md) · [ES 클라이언트 버전 불일치](Observability/트러블슈팅/ES클라이언트_버전불일치_health503.md) |
| 원인을 고쳤는데 **증상이 똑같이 재현**된다 | [IDE가 옛 클래스 파일로 실행](Tooling/IDE가_옛_클래스파일로_실행.md) · [docker compose는 리로드가 아니다](Tooling/docker_compose는_리로드가_아니다.md) |
| 앱이 죽었는데 **알림이 안 울린다** | [SD 전환 후 죽은 대상이 사라진다](Observability/트러블슈팅/SD전환후_죽은대상이_사라진다.md) |
| 5xx 오류율이 실제보다 높게 나온다 | [404가 500으로 둔갑](Observability/트러블슈팅/404가_500으로_둔갑.md) |
| 파드마다 배치 잡이 중복 실행된다 | [레디스 분산락의 한계](Batch_STUDY/레디스_분산락의_한계.md) |
| 재색인할 때마다 벡터DB에 중복이 쌓인다 | [RAG 재색인 멱등성](AI_STUDY/RAG_재색인_멱등성.md) |
| 로컬은 되는데 **배포하면** WebSocket이 끊긴다 | [EKS WebSocket 연결 끊김](EKS_WebSocket_연결끊김.md) |
| 데이터가 계속 쌓이는 테이블의 조회가 느려진다 | [테이블 파티셔닝](테이블_파티셔닝_근태출근.md) |

---

## 반복해서 나온 교훈

작성한 문서들을 관통하는 패턴. 새 문제를 만났을 때 먼저 떠올릴 것.

### 1. "성공처럼 보이는 실패"가 제일 무섭다

도구는 "틀렸다"고 잘 말해주지 않는다. 문법이 유효하면 실행하고, 결과가 없으면 그냥 없다고 답한다.

| 사례 | 겉보기 |
|---|---|
| Git Bash 경로 변환 | `status: success` + 빈 결과 |
| `promtool check config` | `SUCCESS` (빈 `relabel_configs`인데도) |
| `by (le)` 누락 | `status: success` + 빈 결과 + 경고 |
| `status=~"5"` 정규식 | 빈 결과 → 오류율 0%로 보임 |

**빈 결과를 "데이터가 없나 보다"로 넘기지 말 것.** 필터를 하나씩 떼어보며 어느 조건에서 사라지는지 좁힌다.

### 2. "유효하다"와 "의도대로 동작한다"는 별개다

검증 도구 통과는 최소 조건일 뿐이고, 진짜 검증은 결과 확인이다.

### 3. 고쳤는데 증상이 같으면, 고친 게 적용되는 경로에 있는지부터 본다

도구가 중간에 캐시·산출물을 끼고 있으면 수정이 닿지 않는다. (Gradle 캐시, IDE 빌드 산출물, 컨테이너 재생성)

### 4. 값이 맞는다고 검증이 끝난 게 아니다

`sum` 대신 `avg`를 써도 소수점 끝까지 같은 값이 나온 적이 있다. 우연히 상쇄된 것이었다.

### 5. 수치는 그 자리에서 적는다

⚠️ 표시가 붙은 문서는 **당시 기록을 유실해서 측정치를 복원하지 못한 것**이다.
판단 근거는 남았지만 "얼마나 좋아졌는가"가 없어서 반쪽짜리가 됐다.

설계를 바꿀 때는 **바꾸기 전에 한 번, 바꾸고 나서 한 번** 재고 그 자리에서 적을 것.
"나중에 정리해야지"는 기록을 남기지 않는다는 뜻이다.
