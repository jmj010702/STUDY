## LangChain에서 LLM 교체 — Claude → Ollama, 정말 두 줄만 바뀌나

### 문제 배경

RAG 모듈을 Claude로 완성하고 실행했더니 이 에러가 났다.

```
anthropic.BadRequestError: 400 - 'Your credit balance is too low to access the Anthropic API.'
```

**코드는 정상이었다.** 요청이 서버까지 정상 도달했고(`request_id`가 찍힘),
단지 계정에 크레딧이 없었을 뿐이다.

충전 대신 **내 PC에서 도는 무료 오픈소스 모델**로 전환하기로 했다.
학습 단계라 호출 횟수가 많고, **LangChain의 추상화가 실제로 성립하는지 확인해보고 싶었다.**

### 결과 — 정말 두 줄이었다

`search` · `format_docs` · 체인 구성 · 보안 프롬프트는 **전부 그대로**.
`llm`이 가리키는 목적지만 바뀐다.

```python
# Claude
from langchain_anthropic import ChatAnthropic
llm = ChatAnthropic(model="claude-...", temperature=0)

# Ollama
from langchain_ollama import ChatOllama
llm = ChatOllama(model="gemma4:12b", temperature=0)
```

**왜 이게 가능한가** — 체인의 나머지 부분(`prompt | llm | parser`)이
"LLM"이라는 **인터페이스**에만 의존하고 구현체를 몰라도 되게 짜여 있기 때문이다.
LCEL의 `|`는 "앞 단계 출력을 다음 단계 입력으로" 연결할 뿐, 그 안이 뭔지 신경 쓰지 않는다.

### 설정으로 분리하기

두 줄이라도 코드를 고치는 건 여전히 배포가 필요하다. `.env`로 뺐다.

```bash
# .env
LLM_PROVIDER=ollama     # ollama=무료 로컬 / claude=유료
```

```python
# 이 함수 하나로 Claude ↔ Ollama 전환을 흡수. 안 쓰는 쪽은 import도 안 함.
def get_llm():
    if settings.LLM_PROVIDER == "ollama":
        from langchain_ollama import ChatOllama
        return ChatOllama(model=settings.OLLAMA_MODEL, temperature=0)
    from langchain_anthropic import ChatAnthropic
    return ChatAnthropic(model=settings.CLAUDE_MODEL, temperature=0)
```

**안 쓰는 쪽은 import도 안 하는 게 포인트.** 로컬만 쓸 거면 `langchain-anthropic`이 없어도 돈다.

### 통로 — 원리는 같고 목적지만 다르다

```
[Claude]  ChatAnthropic → 인터넷 → https://api.anthropic.com/v1/messages   (유료, 외부)
[Ollama]  ChatOllama   → 루프백 → http://127.0.0.1:11434/api/chat          (무료, 내 PC)
```

- `127.0.0.1`(루프백) = "내 컴퓨터 자기 자신" → 요청이 **랜선으로 안 나간다**(외부 유출 X, 오프라인 가능)
- `ChatOllama`는 기본 `base_url`이 `http://localhost:11434`라 주소를 안 적어도 자동 접속
- **원리는 Claude와 동일**(JSON을 HTTP로 주고받음), 목적지만 내 PC라 공짜

> **Ollama = 모델(두뇌 파일)을 메모리에 올려 돌려주고, 내 PC 안에 작은 API 서버로 열어주는 실행 엔진.**
> 모델은 "재료", Ollama는 "구동기".
> Ollama ≠ 구글. 독립 회사이고 Gemma(구글)·Llama(Meta)·Qwen(Alibaba) 등 여러 회사 모델을 다 돌려준다.

### 설치 ~ 실행

```bash
# 1) Ollama 설치: https://ollama.com/download
ollama --version                 # 버전 뜨면 설치 완료

# 2) 모델 다운로드 (~7.6GB)
ollama pull gemma4:12b
ollama list                      # 목록에 보이면 OK

# 3) 모델만 단독 테스트 (코드 없이)
ollama run gemma4:12b "안녕? 한국어로 한 문장."

# 4) 파이썬 패키지
pip install langchain-ollama
```

**3번이 중요하다.** 코드 없이 모델만 먼저 돌려보면
"모델 문제"와 "내 코드 문제"를 분리할 수 있다.

확인: `curl http://localhost:11434/api/tags` → 들고 있는 모델 목록 JSON

### 겪은 에러 3종

| 에러 | 원인 | 해결 |
|---|---|---|
| `400 credit balance too low` | Anthropic 크레딧 0 (**코드는 정상**) | `.env`에 `LLM_PROVIDER=ollama` |
| 설정을 바꿨는데 **여전히 Claude 호출됨** | `.env`에 `LLM_PROVIDER` 줄을 안 넣어 기본값 `claude`로 감 | `.env`에 줄 추가 |
| `KeyError: 'sources'` | 담을 땐 `"source"`(단수), 꺼낼 땐 `"sources"`(복수) — 이름표 불일치 | 이름 통일 |
| 가짜 import (`from click import prompt` 등) | IntelliJ 자동 import | 해당 줄 삭제 + 자동 import 끄기 |

**두 번째가 전형적인 함정이다.** 코드에는 분기가 있는데 `.env`에 값이 없어서
**기본값으로 조용히 넘어갔다.** 에러가 아니라 "설정이 안 먹은 것처럼 보이는" 상태.

**네 번째는 IDE 설정 문제**다. Settings → Editor → General → Auto Import에서
"붙여넣기 시 임포트 삽입 = 안 함", "즉시 명확한 임포트 추가" 해제.

### 로컬 모델이 느린 이유 (정상)

매 실행마다 **2GB 이상의 weights(모델이 학습으로 익힌 숫자들)를 디스크 → 메모리로 올린다.**
실행할 때마다 하는 게 정상이다.

**운영에서는 서버가 켜질 때 한 번만 올리고 계속 들고 있어서 안 느리다.**
개발 중 체감 속도로 운영 성능을 판단하면 안 된다.

### 배운 점

> **추상화가 실제로 값을 하는지는 갈아끼워 봐야 안다.**
> "LangChain을 쓰면 LLM을 쉽게 바꿀 수 있다"는 문장은 많이 봤지만,
> 직접 바꿔보니 체인·프롬프트·검색 로직이 **정말로 하나도 안 바뀌었다.**
>
> 반대로 말하면, 만약 여기서 프롬프트까지 고쳐야 했다면
> 그 추상화는 광고만큼 값을 못 하는 것이다. **확인 없이 믿지 말 것.**
