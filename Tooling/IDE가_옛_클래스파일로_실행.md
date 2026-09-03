## IDE가 실패한 빌드의 옛 클래스 파일로 앱을 실행한다

### 문제 배경

새 Mac에 개발 환경을 옮기는 중이었다. GitHub Packages 인증이 없어 `common` 모듈을 못 받았다.

```
Could not resolve com.peoplecore:common:1.0.2-SNAPSHOT
  > Unable to load Maven meta-data from https://maven.pkg.github.com/...
     > Username must not be null!
```

원인이 명확했다. `~/.gradle/gradle.properties`에 `gpr.user` / `gpr.token`을 넣어 **고쳤다.**

### 문제

그런데 다시 실행하니 **이번엔 다른 에러**가 났다.

```
Caused by: java.lang.NoClassDefFoundError: com/peoplecore/entity/BaseTimeEntity
```

`BaseTimeEntity`는 **바로 그 `common` 모듈 안의 클래스**다.
인증을 고쳤는데 여전히 `common`을 못 찾는 것처럼 보였다.
→ **"인증 수정이 실패했나?"로 되돌아가기 딱 좋은 상황.**

### 원인 분석 — 단서는 실행 명령 자체에 있었다

IntelliJ가 출력한 `-classpath`를 훑어보니 수백 개 jar 중에
**`common-1.0.2-SNAPSHOT.jar`가 아예 없었다.**
그리고 클래스패스 맨 앞에 이것이 있었다.

```
/Users/.../hr-service/build/classes/java/main    ← 이전 컴파일 산출물
```

**컴파일이 다시 돌지 않았다.**
예전에 만들어진 `.class` 파일을 그대로 실행했고, JVM은 클래스를 **필요할 때 지연 로딩**하므로
`AutoCloseJobConfig`를 들여다보는 순간에야 `BaseTimeEntity`가 없다는 걸 알았다.
그래서 **컴파일 에러가 아니라 런타임 에러의 모습**으로 나타났다.

### 왜 헷갈리나 — 같은 원인의 두 얼굴

| | 실제 | 보이는 것 |
|---|---|---|
| 1차 | 인증 실패로 의존성 해결 실패 | `Username must not be null!` — **원인이 명확** |
| 2차 | 인증은 고쳐졌으나 **IDE가 재컴파일·재해결을 안 함** | `NoClassDefFoundError` — **여전히 common이 없는 것처럼 보임** |

### 해결 방법 — IDE 밖에서 먼저 검증한다

```bash
./gradlew :hr-service:dependencies --configuration compileClasspath --refresh-dependencies | grep peoplecore
→ +--- com.peoplecore:common:1.0.2-SNAPSHOT      # FAILED 없음 = 인증 성공
```

이걸로 **"인증 문제"와 "IDE 상태 문제"를 분리**할 수 있었다.
그다음 IntelliJ에서 Gradle sync를 하면 해결된다.

**`--refresh-dependencies`가 왜 필요한가**
Gradle은 **실패한 조회 결과도 캐시한다.**
캐시를 무시하고 원격에 다시 물어보게 해야 **새 자격증명이 실제로 먹히는지** 알 수 있다.

### 배운 점

> **원인을 고쳤는데 증상이 같으면, 고친 것이 실제로 적용되는 경로에 있는지부터 확인한다.**
> 도구가 중간에 캐시·산출물을 끼고 있으면 수정이 닿지 않는다.
>
> 그리고 **문제를 두 층으로 나눠 각각 확인할 수단을 확보하는 것**이 핵심이었다.
> CLI(`./gradlew`)와 IDE를 분리하지 않았다면 계속 인증 설정만 만졌을 것이다.
