# Harness는 어떻게 확장하는가：skill、설정 디렉토리와 hook

이전 글 《Harness란 정확히 무엇인가》에서 harness의 경계를 명확히 했다—플랫폼 층으로, 컨텍스트 관리、메모리、subagent 편성、도구 호출、hook、실행 클로즈드 루프를 담당하며, 비즈니스 엔지니어링은 그 위에 구축되고 harness를 수정하지 않는다. 이 글에서는 한 단계 더 내려간다：harness 자체는 어떻게 확장하는가?

구체적으로 세 가지를 다룬다：skill에 무엇을 써야 하는가、설정 디렉토리에 무엇을 넣는가、hook은 어떻게 연결하는가. CC와 Codex 모두 이 세 가지 능력을 제공하며, 많은 사람들이 어떤 능력이 특정 제품 전용이라고 생각하지만 실제로 양쪽 모두 대응하는 것을 가지고 있다. 진정한 차이는 "누가 무엇을 가지고 없느냐"가 아니라, **같은 확장 문제에 직면했을 때 두 회사의 엔지니어링 해법이 다른 방향을 택했다**는 점에 있다.

---

## 1. Skill —— 무엇을 써야 하는가、어떻게 로드하고 트리거하는가


### 어떤 문제를 해결하는가

LLM의 기본 동작은 일반적이다. 하지만 당신은 구체적인 프로젝트에서 작업하며, 재사용 가능한 워크플로를 많이 쌓았다：특정 도메인의 코드 리뷰 기준、어떤 프레임워크의 베스트 프랙티스、고정된 커밋 메시지 형식、매번 확인해야 하는 체크리스트.

이런 내용을 처리하는 일반적인 방법이 세 가지 있다：매번 손으로 입력하기（반복 작업）、시스템 프롬프트에 넣기（전역 오염, 모든 대화에 딸려 다님）、shell 스크립트로 래핑하기（agent 컨텍스트에서 벗어남）. Skill은 네 번째 해법이다：재사용 가능한 워크플로、리소스、전문 지식을 자기 완결적 단위로 패키징하여, 적절한 시점에 자동 또는 수동으로 로드하고, 그 외의 시간에는 대화를 방해하지 않는다.

### Codex는 어떻게 하는가

Codex의 skill은 자기 완결적 폴더로, specialized workflows、tool integrations、domain expertise、bundled resources를 제공한다（`codex-rs/skills/src/assets/samples/skill-creator/SKILL.md`에서）.

각 skill의 진입점은 `SKILL.md`이며, frontmatter에는 세 가지 필드가 있다：

```yaml
name: my-skill              # 필수, 호출 시 식별자
description: |              # 필수, 내용은 "이 skill을 언제 사용해야 하는가"
  This skill should be used when...
metadata:
  short-description: 한 줄 요약  # 선택, 목록 표시에 사용
```

`description` 필드의 작성 방식이 트리거 품질을 결정한다. 그 의미는 "어떤 상황에서 나를 호출하는가"이지, "내가 무엇을 할 수 있는가"가 아니다—전자는 트리거 조건이고 후자는 능력 설명이다. Agent는 전자를 사용해 시나리오를 매칭하므로, 잘못 쓰면 트리거율이 매우 낮아진다.

시스템 내장 skill은 `include_dir!` 매크로로 바이너리에 컴파일된다（`skills/src/lib.rs:10`）. 시작 시 `$CODEX_HOME/skills/.system`에 설치된다（`:24`）. 매번 시작할 때 다시 설치하는 것을 피하기 위해 fingerprint marker 파일로 판단한다—내장 내용의 해시가 변경되지 않았으면 설치를 건너뛴다（`:32`）. 이것은 배포 안정성을 보장하는 것으로, 업그레이드 전후에 시스템 skill의 동작을 일관되게 유지한다.

사용자 정의 skill은 `$CODEX_HOME/skills/`에 넣으며, 추가 등록 없이 재시작 후 적용된다.

### CC는 어떻게 하는가

CC의 skill은 `.claude/skills/<name>/SKILL.md`에 있으며, frontmatter에는 세 가지 필드가 있다：`name`、`description`、`disallowed-tools`.

트리거 방식은 두 가지다：사용자가 명시적으로 `/skill-name`을 호출하거나, Claude가 대화 컨텍스트를 기반으로 자동으로 판단해 호출한다. 두 번째 방식은 완전히 `description`의 설명 품질에 의존한다—명확할수록 자동 트리거가 더 정확하다.

로컬 skill 외에도 CC에는 plugin skill 메커니즘이 있다：plugin 마켓을 통해 설치하고 `/plugin`으로 관리한다. 세션 내에서 `/reload-skills`로 다시 스캔할 수 있으며（v2.1.152 지원）, 또는 `SessionStart` hook에서 `reloadSkills` 파라미터를 통해 다시 스캔을 트리거할 수 있다.

`disallowed-tools`는 CC 측의 고유한 필드다：skill 수준에서 특정 도구를 끌 수 있다. 예를 들어 "코드 리뷰" skill은 파일을 읽기만 하면 되고 파일을 쓸 필요가 없으므로, `disallowed-tools`에서 파일 쓰기 도구를 끄면 이 skill 실행 중에는 쓰기 작업이 트리거되지 않아 잘못된 조작 리스크를 줄인다.

### 비교

| 차원 | Codex | CC |
|------|-------|----|
| 저장 위치 | `$CODEX_HOME/skills/` | `.claude/skills/<name>/` |
| frontmatter 필드 | name / description / metadata.short-description | name / description / disallowed-tools |
| 트리거 방식 | 컨텍스트 자동 트리거 | 명시적 `/skill-name` + 컨텍스트 자동 트리거 |
| 도구 수준 제한 | 전용 필드 없음 | `disallowed-tools` |
| 내장 skill | 바이너리에 컴파일, fingerprint 재설치 최적화 | plugin 마켓 |
| 핫 리로드 | 파일 시스템 업데이트 시 | `/reload-skills` 또는 `SessionStart` hook |

### 실용 조언

skill 작성에서 가장 쉽게 빠지는 함정은 `description`을 능력 소개로 쓰는 것이지, 호출 조건이 아닌 것이다. 반례：

> 능력 소개 형식："This skill provides Python code review capabilities, including style checking and performance analysis."

트리거율이 매우 낮다. Agent 입장에서는 "이 능력이 있다"는 것이지, 언제 호출해야 하는지 모른다.

조건 트리거 형식으로 바꾸면：

> "This skill should be used when reviewing Python code, checking for style issues, or analyzing performance bottlenecks in Python files."

다른 몇 가지 포인트：
- CC의 `disallowed-tools`를 잘 활용하라：각 skill에 도구 범위를 명확히 제한하여 skill 간의 상호 간섭을 방지한다
- 시스템 내장 skill은 유지 보수 부담이며, Codex가 바이너리에 컴파일하는 것은 배포 일관성을 위한 것이다. 일상 사용은 사용자 디렉토리에 바로 넣으면 된다
- 로컬 skill을 레포지토리에 커밋하여 팀이 공유한다—skill 자체가 재사용 가능한 워크플로 자산이며, 개인 기기에만 있어서는 안 된다

---

## 2. 설정 디렉토리 —— .codex vs .claude, 무엇을 넣는가


### 어떤 문제를 해결하는가

Agent의 동작에는 설정이 필요하다：어떤 모델을 사용할지、어떤 도구를 허용할지、프로젝트 수준의 코드 규범 제약、기업 수준의 보안 경계. 여기에는 자연적인 충돌이 있다：개인 선호、프로젝트 제약、기업 정책 세 가지의 설정 권한이 다르며, 모두 한 파일에 넣어 "나중 것이 이전 것을 덮어씀"으로 간단히 처리할 수 없다—기업의 보안 경계가 프로젝트 설정 파일에 의해 덮어씌워져서는 안 된다.

설정 디렉토리의 핵심 설계 문제는：**누구의 설정이 누구의 설정을 덮어쓸 수 있는가, 어디서 고정시키는가**.

### Codex는 어떻게 하는가

Codex의 프로젝트 수준 설정은 `.codex/config.toml`이며, 형식은 TOML이다. 분층 로드 순서는 `config/src/loader/mod.rs:79-100`에 완전한 주석이 있으며, 두 가지 로직으로 나뉜다：

**제약 층**（constraint, 기업/보안 정책에 사용）：

```
cloud → admin → system (/etc/codex/requirements.toml)
```

이 층은 "이른 층 고정" 의미론을 사용한다：`a constraint defined in an earlier layer cannot be overridden by a later layer`（원문）. 시스템 관리자가 `/etc/codex/requirements.toml`에 쓴 제약은 프로젝트 층이 덮어쓸 수 없고, 사용자 층도 덮어쓸 수 없다.

**설정 층**（일반 설정, 나중 층이 이른 층을 덮어씀）：

```
admin → system (/etc/codex/config.toml)
     → user ($CODEX_HOME/config.toml)
     → profile ($CODEX_HOME/<name>.config.toml)
     → cwd (./config.toml)
     → tree (./.codex/config.toml, 단계적으로 위로 찾음)
     → repo (git root/.codex/config.toml)
     → runtime (--config 파라미터 등)
```

프로젝트 루트의 판단은 `project_root_markers`에 의해 결정되며, 기본값은 `.git`이다. 설정 파일이 프로젝트 루트에 가까울수록 우선순위가 높다（runtime이 최고）.

보호 경로는 또 다른 하드 제약이다：`PROTECTED_METADATA_PATH_NAMES = [".git", ".agents", ".codex"]`（`protocol/src/permissions.rs:27`）. danger-full-access 샌드박스 모드에서도 agent는 이 세 디렉토리에 쓸 수 없다. 이것은 시스템 수준 보호이며, 선택적 설정이 아니다.

`.agents/`는 subagent 정의를 넣고；`.codex/hooks.toml`은 hook 설정을 넣는다（다음 섹션에서 자세히 설명）.

### CC는 어떻게 하는가

CC의 설정 디렉토리는 `.claude/`이며, 다음을 포함한다：

- `settings.json`：JSON 형식, agent 동작 설정
- `CLAUDE.md`：Markdown, 시스템 프롬프트와 컨텍스트 설명
- `skills/`：로컬 skill 디렉토리
- `agents/`：서브 agent 정의
- `commands/`：커스텀 슬래시 명령
- `rules/`：경로 범위 로드 규칙 파일
- `.mcp.json`：`MCP` 도구 설정

우선순위 고에서 저：managed（기업 IT 배포）> 커맨드라인 파라미터 > local > project（`.claude/`）> user（`~/.claude/`）. 그 중 managed settings는 기업 층 강제 정책이며 우선순위가 가장 높고, 프로젝트 층이 덮어쓸 수 없다.

`rules/`에는 따로 언급할 가치가 있는 특성이 있다：frontmatter의 `paths:` 필드로 경로 범위를 지원한다. 예를 들어 `rules/python-style.md`에 `paths: ["**/*.py"]`를 쓰면, 이 규칙은 Claude가 Python 파일을 읽을 때만 컨텍스트에 추가되고, 그 외의 시간에는 토큰을 차지하지 않는다. 대형 레포지토리에서 이 특성은 관련 없는 규칙의 토큰 소비를 눈에 띄게 줄일 수 있다.

### 비교

| 차원 | Codex | CC |
|------|-------|----|
| 프로젝트 설정 파일 | `.codex/config.toml`（TOML） | `.claude/settings.json`（JSON）+ `CLAUDE.md`（MD） |
| 기업 강제 정책 | requirements.toml, 제약 층 고정, 나중 층 덮어쓸 수 없음 | managed settings, 최고 우선순위 |
| 분층 로직 | 제약 층（고정）+ 설정 층（덮어쓰기）두 가지 독립 로직 | 통합 우선순위 체인, managed가 최고 |
| 보호 경로 | `.git` / `.agents` / `.codex`, 샌드박스도 쓸 수 없음 | 유사한 메커니즘 없음 |
| 경로 범위 규칙 | 없음 | `rules/` frontmatter `paths:` 필드 |
| subagent 정의 | `.agents/` | `.claude/agents/` |

두 회사는 "기업 정책은 덮어씌울 수 없다"는 방향에서 일치하지만, 구현이 다르다：Codex는 제약 층과 설정 층의 병합 로직을 분리해서 쓰며, 제약 층은 코드에서 명확한 "이른 층 고정" 의미론을 가진다；CC는 우선순위 체인에 의존하며, managed가 최고 우선순위를 가져 보장한다.

### 실용 조언

분층의 핵심 가치는 **설정 귀속을 명확히 하는 것**이다：

- 개인 선호（어떤 모델을 사용할지、어떤 스타일인지）는 사용자 층（`~/.codex/config.toml` 또는 `~/.claude/settings.json`）에 넣고, 레포지토리에 커밋하지 않는다
- 프로젝트 제약（도구 권한 범위、코드 규범 설명）은 프로젝트 층에 넣고 레포지토리에 커밋하여, 팀원들이 같은 제약을 공유한다
- 기업 보안 경계（특정 도구 호출 금지、네트워크 접근 제한）는 관리 층을 사용하고, 프로젝트 층으로 덮어쓰려 하지 않는다
- CC의 `rules/` 경로 범위를 충분히 활용하라：모든 규칙을 하나의 큰 `CLAUDE.md`에 쌓는 것보다, 언어나 디렉토리별로 분리하여 매번 관련된 것만 로드하는 것이 낫다

---

## 3. Hook —— 이벤트、유형、신뢰


### 어떤 문제를 해결하는가

Agent가 실행하는 동안 많은 핵심 노드가 있다：도구 호출 전후、세션 시작과 종료、사용자가 프롬프트를 제출할 때、컨텍스트 압축 시、서브 agent 시작과 정지 시. 이 노드들에 커스텀 로직을 삽입하면 많은 것을 할 수 있다：

- 파일 쓰기 전에 확인하기
- 도구 호출 후에 감사 로그 기록하기
- 세션 종료 시에 완성도 확인하기
- 권한 신청 시에 자동으로 결정하기

두 회사 모두 이벤트 구동 생명주기 모델을 사용하지만, 마운트 설정、핸들러 유형、신뢰 메커니즘이 각각 다르다.

### Codex는 어떻게 하는가

Codex의 hook 설정은 `.codex/hooks.toml`에 있다（TOML 형식）.

완전한 이벤트 목록（`config/src/hook_config.rs`에서, 총 10개）：

```
PreToolUse         도구 호출 전
PostToolUse        도구 호출 후
PermissionRequest  권한 신청 시
PreCompact         컨텍스트 압축 전
PostCompact        컨텍스트 압축 후
SessionStart       세션 시작
UserPromptSubmit   사용자 프롬프트 제출
SubagentStart      서브 agent 시작
SubagentStop       서브 agent 정지
Stop               agent 정지
```

핸들러 유형（`HookHandlerConfig` enum, `hook_config.rs:137-156`）：

```rust
enum HookHandlerConfig {
    command { command, commandWindows, timeout, async, statusMessage },
    prompt {},
    agent {},
}
```

세 가지 유형—`command`（shell 명령）、`prompt`（LLM 평가）、`agent`（서브 agent 검증）.

신뢰 메커니즘은 `HookStateToml`의 `trusted_hash: Option<String>` 필드를 통해 구현된다（`hook_config.rs:29`）. 각 hook은 `state` 섹션에서 hash를 선언해야 하며, hash가 현재 파일 내용과 일치할 때만 hook이 자동으로 실행된다；일치하지 않으면 사용자 승인이 필요하다.

```toml
[state.my-audit-hook]
trusted_hash = "sha256:abc123"

[hooks.PostToolUse]
[[hooks.PostToolUse]]
matcher = "write_file"
[[hooks.PostToolUse.hooks]]
type = "command"
command = "echo 'file written' >> /tmp/audit.log"
```

`trusted_hash`의 설계 동기는 악의적인 hook을 방어하기 위한 것이다：여러 사람이 공유하는 레포지토리에서 `.codex/hooks.toml`이 변조되면 hash가 무효화되어, hook이 조용히 실행되지 않고 승인을 트리거한다—hook 자체를 잠재적인 공격 면으로 취급하는 것이다.

### CC는 어떻게 하는가

CC의 hook 설정은 `settings.json`에 있다. 공식 문서에 32개의 이벤트가 나열되어 있으며, 다음을 포함한다：`SessionStart` / `SessionEnd` / `UserPromptSubmit` / `PreToolUse` / `PostToolUse` / `PostToolUseFailure` / `Stop` / `PermissionRequest` / `Notification` / `MessageDisplay` 등.

핸들러 유형：`command`（shell）/ `http`（HTTP 요청）/ `mcp_tool`（`MCP` 도구）/ `prompt`（LLM 평가）/ `agent`（서브 agent 검증）.

몇 가지 실행 규칙을 기억할 가치가 있다：
- `matcher` 필드는 정규식을 지원하며, 도구 이름이나 이벤트 내용을 매칭한다
- exit code 0 = 성공, 2 = 차단（Claude에게 현재 작업을 중단하라고 알림）
- JSON은 stdin/stdout을 통해 컨텍스트와 결과를 전달한다

`prompt` 유형의 실제 사용법：`Stop` hook에 프롬프트를 써서 소규모 모델이 "작업이 정말 완료되었는가"를 판단하게 하고, 완료되지 않으면 비 0을 반환하여 Claude가 계속하게 할 수 있다. 이것은 hook의 역할을 "모니터링+차단"에서 "모니터링+판단+역제어"로 확장하여, agent loop 말미에 자동 QA 층을 추가하는 것과 같다.

`http` 유형은 CC의 고유한 것이다：shell 스크립트를 통한 간접 구현 없이 직접 HTTP 엔드포인트를 호출한다. 외부 승인 시스템이나 로그 플랫폼에 hook을 연결하기에 적합하다.

### 비교

| 차원 | Codex | CC |
|------|-------|----|
| 설정 파일 | `.codex/hooks.toml`（TOML） | `settings.json`（JSON） |
| 이벤트 총 수 | 10개 | 32개 |
| 핸들러 유형 | command / prompt / agent | command / http / mcp_tool / prompt / agent |
| 고유 이벤트 | `SubagentStart` / `SubagentStop` / `PreCompact` / `PostCompact` | `SessionEnd` / `PostToolUseFailure` / `Notification` / `MessageDisplay` / `Setup` / `PermissionDenied` 등 |
| 신뢰 메커니즘 | `trusted_hash`：hash 일치 시만 자동 실행 | 파일 시스템 권한에 의존 |
| HTTP 유형 | 없음（command를 통한 간접 구현 필요） | 있음, 직접 HTTP 엔드포인트 호출 |

두 회사 모두 `prompt`와 `agent` 유형의 hook을 가지고 있으며, 이것은 많은 사람들의 인식과 다르다—소스 코드의 `HookHandlerConfig` enum에 이 두 가지 변형이 명확히 있다. 차이는 주로：Codex에는 `trusted_hash`의 파일 수준 신뢰 메커니즘이 있고（보안 중시）, CC에는 HTTP 유형과 더 많은 이벤트 노드가 있다（통합 중시）.

### 실용 조언

사용 빈도로 보면, 가장 자주 사용하는 조합 몇 가지：

- `PreToolUse` + `matcher`로 위험한 도구 매칭 + `command` 유형：`rm -rf`、`git push --force` 전에 로그나 확인을 삽입
- `PostToolUse` + `command`：도구 호출 기록을 감사 파일에 써서 나중에 재확인
- `Stop` + `prompt` 유형：LLM이 "이번 작업 목표가 달성되었는가"를 판단하게 하고, 달성되지 않으면 exit code 2를 반환하여 Claude가 계속하도록 트리거
- Codex 사용자는 `trusted_hash` 유지에 주의하라：hook 스크립트를 수정할 때마다 hash가 무효화되며, 다시 신뢰해야 한다. 이것은 예상된 동작이지 버그가 아니다. 하지만 hook을 자주 수정한다면 hash 업데이트를 워크플로에 포함시켜야 한다

---

## 소결

세 가지 확장 능력에서 두 측의 설계 방향을 각각 한 문장으로 요약할 수 있다：

**Codex**：빌드 시에 고정（시스템 skill을 바이너리에 컴파일）、런타임에 검증（trusted_hash）, 제약 층의 이른 층 고정은 덮어씌울 수 없다—보안 경계 우선, 제약을 앞으로 당긴다.

**CC**：파일 시스템을 중심으로, 경로 범위 규칙、plugin 마켓、HTTP hook이 확장점을 더 유연하게 만든다—조합 가능성 우선, 유연성을 외부로 연장한다.

두 회사 모두 "어떤 능력이 우리만 가진 것"은 없다. 진정한 분기는 같은 확장 요구에 대한 엔지니어링 태도다：Codex는 "제약이 일찍 확정될수록 안전하다"를 더 믿는 경향이 있고, CC는 "조합이 유연할수록 유용하다"를 더 믿는 경향이 있다.

어떤 harness를 사용하고 어떤 확장 방식을 쓸지는 결국 당신의 시나리오에 달려 있다—팀 규모、보안 요구사항、외부 시스템과의 통합 깊이. 두 가지 철학에는 옳고 그름이 없고, 적합한지 아닌지만 있을 뿐이다.

---

## Harness Engineering 시리즈

coding agent의 「플랫폼 층(harness)과 비즈니스 엔지니어링」을 주제로, 편마다 분해한다：

1. [Harness란 정확히 무엇인가](./01-what-is-harness.md) —— 플랫폼 층과 비즈니스 엔지니어링의 경계
2. [복잡한 작업의 Spec은 어떻게 쓰는가](./02-how-to-write-specs.md) —— 멀티 Agent、편성자 진입점、rules / docs / skills 구성
3. [Harness는 어떻게 확장하는가](./03-extending-the-harness.md) —— skill、설정 디렉토리와 hook：CC와 Codex의 두 가지 메커니즘
4. [Harness는 어떻게 agent를 제어하는가](./04-permissions-and-effort.md) —— 권한과 effort：CC와 Codex의 제어 면
5. Harness는 어떻게 긴 작업을 버티는가 —— compact、memory、goal（작성 중）

> 다른 언어：[English](../en/03-extending-the-harness.md) · [한국어](../ko/03-extending-the-harness.md) · [日本語](../ja/03-extending-the-harness.md) · [中文](../zh/03-extending-the-harness.md)
