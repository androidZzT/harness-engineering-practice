# Harness는 어떻게 agent를 제어하는가：권한과 effort

이전 글 《Harness는 어떻게 확장하는가：skill、설정 디렉토리와 hook》은 "harness에 무언가를 추가하는 것"에 관한 것이었다—재사용 가능한 워크플로를 skill로 패키징하고、제약을 설정 디렉토리에 쓰고、모니터링 로직을 hook에 연결하는 것. 이 글은 각도를 바꾼다：agent에 도구를 장착한 후, 어떻게 관리하는가?

두 가지 구체적인 문제：첫째, agent가 파일을 변경하고、명령을 실행하고、네트워크 요청을 보낼 때, 어떤 작업은 물어보고 어떤 것은 자동으로 허용할지 어떻게 결정하는가；둘째, 같은 모델에서 시나리오에 따라 "얼마나 깊이 생각할지"를 어떻게 조절하여 토큰을 낭비하지 않으면서도 복잡한 작업을 대충 처리하지 않을 수 있는가.

CC와 Codex 모두 이 두 가지 문제에 해법을 가지고 있다. 이전 글들과 마찬가지로, 차이는 누가 무엇을 가지고 없느냐가 아니라, **같은 제어 문제에 직면했을 때 두 회사의 엔지니어링 해법이 다른 방향을 택했다**는 점에 있다.

---

## 1. 권한


### 어떤 문제를 해결하는가

Agent는 실행 중에 파일을 변경하고、명령을 실행하고、네트워크 요청을 보낸다. 완전히 개방하면 위험하고, 매번 물어보면 효율이 떨어진다. 핵심 모순은：**"방해 최소화"와 "제어 가능" 사이에서 조절 가능한 균형점을 어떻게 찾을 것인가**, 그리고 이 균형점을 "개인 / 프로젝트 / 기업"별로 계층적으로 고정할 수 있게 하는 것이다.

### Codex는 어떻게 하는가

Codex는 권한 부여를 두 개의 직교 차원으로 분리하여 권한 행렬을 구성한다.

**차원 1：샌드박스 정책**（`SandboxPolicy`）, agent의 작업 경계를 정의한다. 열거형（`codex-rs/protocol/src/protocol.rs:878`）, 4가지 변형：

- `DangerFullAccess`（직렬화 `"danger-full-access"`）—— 무제한
- `ReadOnly { network_access: bool }` —— 읽기 전용, 네트워크 기본 비활성화
- `ExternalSandbox { network_access }` —— 이미 외부 샌드박스에 있고, 디스크는 개방하되 전달된 네트워크 설정을 따름
- `WorkspaceWrite { writable_roots, network_access, exclude_tmpdir_env_var, ... }` —— ReadOnly 기반으로 현재 작업 공간 + 나열된 `writable_roots`에 대한 쓰기를 추가로 허용

**차원 2：승인 정책**（`AskForApproval`）, 언제 사용자에게 멈추고 물어볼지를 결정한다. 열거형（`codex-rs/protocol/src/protocol.rs:784`）, 5가지 변형：

- `UnlessTrusted`（`"untrusted"`）—— "이미 안전하고 읽기 전용"인 명령만 자동 승인, 나머지는 모두 물어봄
- `OnFailure`（DEPRECATED 표시）—— 샌드박스 내 전자동 승인, 실패 시 사용자에게 상위 전달
- `OnRequest`（`#[default]`, 기본 档）—— 모델이 언제 물어볼지 결정
- `Granular(GranularApprovalConfig)` —— 카테고리별로 세밀하게 켜고 끄기, 어떤 카테고리는 `true`로 허용, `false`로 자동 거부（사용자에게 팝업 없음）
- `Never` —— 절대 물어보지 않음, 실패 시 모델에게 바로 반환, 상위 전달 없음

`Granular` 변형 내의 `GranularApprovalConfig`（같은 파일 `protocol.rs`, `AskForApproval` 정의 직후）에는 5개의 카테고리 필드가 있다：`sandbox_approval`（shell 명령 승인, 인라인 권한 상승 포함）、`rules`（execpolicy `prompt` 규칙 트리거 프롬프트）、`skill_approval`（skill 스크립트 실행）、`request_permissions`（`request_permissions` 도구 트리거）、`mcp_elicitations`（`MCP` elicitation 프롬프트）.

두 차원의 분업：샌드박스 정책은 "경계"를 관리하고（파일 읽기쓰기 범위 + 네트워크 스위치）, 승인 정책은 "방해 리듬"을 관리한다（언제 멈추고 물어볼지）. 두 가지는 독립적이며 임의로 조합할 수 있다.

### CC는 어떻게 하는가

CC는 권한 부여를 세 층으로 겹쳐서 만든다：모드（baseline）+ 규칙（allow/ask/deny）+ 분류기（auto mode 실시간 심사）.

**6가지 권한 모드**（permission-modes 페이지）, Shift+Tab으로 `default→acceptEdits→plan` 사이를 순환：

- `default` —— 읽기 전용 작업만 물어보지 않음
- `acceptEdits` —— 읽기 + 파일 편집 + 작업 디렉토리 내 `mkdir`/`touch`/`mv`/`cp`/`rm`/`rmdir`/`sed`는 물어보지 않음
- `plan` —— 읽기 전용, 연구하고 방안을 제시하며 소스 코드는 변경하지 않음
- `auto` —— 거의 물어보지 않지만, 독립적인 분류기 모델이 동작 실행 전에 실시간 심사；research preview로 표시；Claude Code v2.1.83+ 필요, 모델은 Opus 4.6+ 또는 Sonnet 4.6 필요
- `dontAsk` —— 사전 승인된 도구만 실행, 나머지는 모두 자동 거부；고정된 CI 또는 스크립트에 적합
- `bypassPermissions` —— 모든 검사를 건너뛰고 전혀 물어보지 않음, `--dangerously-skip-permissions`와 동일, 격리된 컨테이너/VM에서만 사용；`rm -rf /`와 `rm -rf ~`는 여전히 퓨즈 프롬프트로 표시됨

`bypassPermissions` 외에, 보호 경로（`.git`/`.claude`/`.vscode`/`.idea` 등）에 대한 쓰기는 절대 자동 승인되지 않는다.

**규칙 세 종류**（permissions 페이지）：`allow`（물어보지 않고 바로 사용）、`ask`（매번 확인）、`deny`（금지）. 평가 순서는 deny → ask → allow이며, 첫 번째 매칭이 승리하고 deny는 항상 우선이다.

기억할 가치가 있는 세부 사항：bare 도구 이름 deny（예：`Bash`）는 도구를 컨텍스트에서 완전히 제거하고；scoped deny（예：`Bash(rm *)`）는 도구를 유지하고 매칭되는 호출만 차단한다.

**settings 5단계 우선순위**：Managed（기업 IT 배포, 최고, 어떤 층도 덮어쓸 수 없음）> 커맨드라인 파라미터 > Local（`.claude/settings.local.json`）> Project（`.claude/settings.json`）> User（`~/.claude/settings.json`）. 어떤 층에서든 deny하면 다른 층에서 allow할 수 없다.

**auto mode 분류기**（permission-modes 페이지）에는 몇 가지 세부 사항이 있다：

독립적인 분류기 모델이 동작 실행 전에 심사하여 세 가지 시나리오를 차단한다：요청 범위를 초과하는 상위 전달、알 수 없는 인프라를 향하는 것、읽은 적대적 내용에 의해 구동되는 동작. 결정 순서는 먼저 allow/deny 규칙을 거치고, 읽기 전용 및 작업 디렉토리 내 편집은 바로 허용하며, 나머지는 분류기에 맡기고, 차단 시 이유를 모델에게 반환한다.

auto mode 진입 시 광범위한 allow 규칙을 버린다（`Bash(*)`、와일드카드 인터프리터、패키지 관리 run、`Agent` 규칙）, 나갈 때 복원한다. 분류기는 user 메시지、도구 호출、`CLAUDE.md`만 본다. 도구 결과는 제거된다（주입 방지）.

롤백 임계값：연속 3번 또는 누적 20번 차단되면 auto mode가 일시 중지되고 개별 질문으로 전환된다（임계값은 설정 불가）.

세션에서 구두로 말한 경계（"일단 push하지 마세요"）는 block 신호로 처리되지만 지속되지 않으며, compaction이 그 메시지를 삭제하면 무효화될 수 있으므로, 하드 보장은 deny 규칙으로 작성해야 한다.

### 비교

| 차원 | Codex | CC |
|------|-------|----|
| 핵심 모델 | 두 개의 직교 차원：샌드박스 정책 × 승인 정책 | 세 층 겹침：모드 + 규칙 + 분류기 |
| 档위 세분도 | 4 샌드박스 × 5 승인, 조합 행렬 | 6가지 모드（dontAsk 전사전승인 포함） |
| 최세밀 제한 | `Granular`：5개 카테고리 필드 독립 스위치 | scoped deny：`Bash(rm *)` 수준 |
| 의미 심사 | 없음（확정적 규칙 조합） | auto mode 분류기, 의미적 판단 |
| 기업 정책 고정 | 샌드박스 정책 + 승인 정책 제약 조합 | Managed settings, 5단계 최고 우선순위 |

Codex는 "정책 심사" 쪽이다：두 개의 확정적 차원의 직교 조합, 오픈소스로 증명 가능하며 동작이 예측 가능하다. CC는 "모델 심사 + 정책 안전망" 쪽이다：auto mode는 독립 분류기로 의미 판단을 하고（"요청을 초과했는가 / 외부 인프라인가 / 주입에 의해 구동되는가" 판단）, deny 규칙 층이 안전망 역할을 한다.

### 실용 조언

**Codex**：일상은 `WorkspaceWrite` + `OnRequest`（기본 조합）；무인 배치 실행에는 `Never`를 사용하되, 반드시 좁은 샌드박스와 함께 사용해야 한다（네트워크 끄기、`writable_roots` 제한）；`Granular`은 "shell은 허용하지만 `MCP` / skill 스크립트는 막는" 같은 세밀한 시나리오에 적합하다.

**CC**：회고식 코딩에는 `acceptEdits`를 사용하고 편집기나 `git diff`로 사후 확인；긴 작업에서 방해를 줄이려면 `auto`를 사용하되, research preview이므로 민감한 작업은 여전히 사람이 봐야 한다；CI에는 `dontAsk` + 사전승인 allow 목록 사용；`bypassPermissions`는 격리된 컨테이너에서만 사용.

**공통**：기업 보안 경계는 Managed 또는 제약 층으로 고정하고, 프로젝트 층으로 덮어쓰려 하지 않는다；하드 경계는 deny 규칙으로 작성하고, 대화에서 구두로만 말하는 것에 의존하지 않는다（compaction이 그 메시지를 삭제할 수 있다）.

---

## 2. effort


### 어떤 문제를 해결하는가

같은 모델이 "적게 생각하고 빠르게 답할" 수도 있고, "많이 생각하고 깊게 답할" 수도 있다. 단순한 작업에 너무 많이 생각하면 토큰을 낭비하고 overthinking이 생길 수 있으며；복잡한 작업에 너무 적게 생각하면 충분하지 않다. 그래서 "얼마나 깊이 생각할지"를 조절할 수 있는 档위가 필요하며, 모델별、시나리오별로 기본값을 설정할 수 있어야 한다.

### Codex는 어떻게 하는가

**`ReasoningEffort` 열거형 6档**（`codex-rs/protocol/src/openai_models.rs:45`）：

`None` / `Minimal` / `Low` / `Medium`（`#[default]`）/ `High` / `XHigh`, 직렬화 `"none"/"minimal"/"low"/"medium"/"high"/"xhigh"`, 기본값 `Medium`.

**클라이언트는 단 하나만 한다：그대로 전달**. `build_reasoning`（`codex-rs/core/src/client.rs:715`）이 effort를 요청에 조합하며, 핵심 행은 `effort: effort.or(model_info.default_reasoning_level)`（`:722`）이고, 최종적으로 OpenAI Responses API 요청 본문의 `{"reasoning":{"effort":"high"}}`가 된다.

Codex 클라이언트는 system prompt를 변경하지 않고、도구 목록을 변경하지 않고、컨텍스트 창을 조정하지 않고、로컬 추론 강화를 하지 않는다. 문자열을 서버 측에 보내고, `"high"`를 실제 연산량으로 변환하는 것은 클라우드 모델의 일이다.

**3단계 fallback**（`effective_reasoning_effort`, `codex-rs/core/src/session/turn_context.rs:125`）：현재 turn에서 사용자가 설정한 값 → 모델 기본档（`default_reasoning_level`）→ 모델이 reasoning을 지원하지 않으면 전체 `reasoning` 필드를 보내지 않음（참고：`"none"`을 보내는 것이 아니라 필드 자체를 보내지 않는 것）.

**모델 전환 시 档위 매핑**：`effort_rank`（`openai_models.rs:557-562`, None=0 / Minimal=1 / Low=2 / Medium=3 / High=4 / XHigh=5）와 `nearest_effort`（`:566`）를 사용하여, rank 차이 절대값이 가장 작은 것으로 대상 모델이 지원하는 가장 가까운 档위에 매핑한다.

**Plan 모드 독립 effort**：`plan_mode_reasoning_effort`（`codex-rs/core/src/config/mod.rs:873`）와 실행 단계는 독립적인 필드이며, Plan 단계에 더 높은 档위를 별도로 설정할 수 있다.

### CC는 어떻게 하는가

**档위는 모델에 따라 다름**（model-config 페이지）：

- Opus 4.8 / Opus 4.7：`low` / `medium` / `high` / `xhigh` / `max`
- Opus 4.6 / Sonnet 4.6：`low` / `medium` / `high` / `max`
- 기본 档：Opus 4.8、Opus 4.6、Sonnet 4.6은 모두 `high`；Opus 4.7은 `xhigh`
- 현재 모델이 지원하지 않는 档위를 설정하면, 그 档위를 초과하지 않는 최고 지원 档위로 fallback（예：Opus 4.6에서 `xhigh` 설정 시 실제로는 `high`로 실행）

**지속성 차이**：`low`/`medium`/`high`/`xhigh`는 세션 간 지속；`max`는 현재 세션에서만 유효하다（`CLAUDE_CODE_EFFORT_LEVEL` 환경 변수를 통하지 않는 경우）. `max`는 가장 깊은 추론이며 토큰 상한이 없는 档位이고, 공식 문서의 원문은 "prone to overthinking, test before adopting broadly"다.

**두 가지가 더 있는데, 모델 档위가 아니라 CC가 자체적으로 추가한 것이다**：

`ultracode`는 따로 더 설명할 가치가 있다. "Opus 4.8의 추가 추론 档位"로 자주 오해되는데, 실제로는 그렇지 않다. 이것은 CC의 세션 수준 설정이다：활성화하면 모델에 `xhigh`를 보내고, 동시에 Claude가 각 실질적인 작업에 대해 자동으로 **dynamic workflow**를 편성하도록 한다—Claude가 직접 작성하고 백그라운드에서 실행되는 JavaScript 편성 스크립트로, 작업을 수십에서 수백 개의 subagent에게 분배하여 병렬로 실행한다（동적 워크플로는 v2.1.154에서 도입）.

일반 subagent 편성과의 핵심 차이：스크립트 자체가 루프、분기、중간 결과를 가지며, Claude의 컨텍스트에는 최종 답변만 남는다. 따라서 "하나의 대화로 조율하기 어려운" 대규모 작업에 적합하다—전체 레포지토리 버그 스캔、수백 개 파일 마이그레이션、다중 소스 교차 검증 조사；스크립트는 품질 루틴을 고정할 수도 있다. 예를 들어 여러 agent가 서로 대결식으로 각자의 결론을 검토한 후 보고하게 할 수 있다.

트리거 방법은 두 가지다：프롬프트에서 직접 요청하거나（자신의 말로, 또는 `ultracode` 키워드 사용）, `/effort ultracode`로 Claude가 전체 세션에서 각 실질적인 작업에 대해 자동으로 계획을 수립하도록 한다. `xhigh`를 지원하는 모델에서만 `/effort` 메뉴에 나타나며（즉 Opus 4.8 / 4.7）, 나머지 모델에서는 제공되지 않는다. 현재 세션에서만 유효하며, `effortLevel` 설정、`--effort` flag、`CLAUDE_CODE_EFFORT_LEVEL`에 속하지 않는다.

`ultrathink` 키워드：프롬프트 어디에서든 `ultrathink`를 쓰면 현재 turn에 더 깊은 추론을 요청한다. 세션 effort를 변경하지 않으며, API에 보내는 effort 값도 변경되지 않고, 컨텍스트에 in-context 지침 하나만 추가된다. "think" / "think hard" 등은 키워드로 인식되지 않는다.

**설정 진입점**：`/effort`（파라미터 없이 슬라이더 열기 / `/effort <档名>`으로 직접 설정 / `/effort auto`로 모델 기본값으로 돌아가기）；`/model`에서 좌우 키로 슬라이더 조정；`--effort` flag；`CLAUDE_CODE_EFFORT_LEVEL` 환경 변수；`effortLevel` 설정（low~xhigh만 받으며, `max`와 `ultracode`는 받지 않음）；skill / subagent frontmatter의 `effort` 필드. 우선순위：환경 변수 > 설정档 > 모델 기본값.

### 비교

| 차원 | Codex | CC |
|------|-------|----|
| 档위 수 | 6档（None/Minimal/Low/Medium/High/XHigh） | 모델에 따라, 최대 5档（low/medium/high/xhigh/max） |
| 기본档 | `Medium` | 대부분 모델 `high`（Opus 4.7은 `xhigh`） |
| 클라이언트 처리 여부 | 처리 없음（순수 전달, API 한 필드만 변경） | 추가 캡슐화 있음（ultracode 트리거 시 백그라운드 워크플로 편성） |
| 단일 turn 임시 심화 | 전용 키워드 없음 | `ultrathink` 키워드（세션档 변경 없음） |
| max 지속성 | 해당 없음 | 현재 세션만（환경 변수 설정 아닐 때） |
| 모델 전환 档위 매핑 | `nearest_effort`：절대값 최가까운档（升档 가능） | 대상档 초과하지 않는 최고 지원档 취함（하강만, 상승 없음） |
| Plan 단계 독립 effort | 있음（`plan_mode_reasoning_effort` 독립 필드） | 전용 필드 없음 |

두 회사는 low/medium/high/xhigh의 중간 단계 명칭을 공유하며, 모두 모델 기본档과 fallback 메커니즘을 가진다. 차이는：Codex는 effort를 API에 전달하기만 하고 로컬에서 처리하지 않는 필드로 취급하며, 소스 코드로 증명 가능하다；CC는 같은 档위 외에 자체적인 것을 몇 가지 추가했다—위로는 `max`（상한 없음）、`ultracode`（effort와 함께 백그라운드 워크플로 편성）、`ultrathink`（단일 turn 임시 심화, 세션档 변경 없음）.

fallback 알고리즘도 다르다：Codex `nearest_effort`는 rank 차이 절대값이 가장 작은 것으로 매핑하며 대상档보다 높아질 수 있다；CC는 "대상档을 초과하지 않는 최고 지원档"을 취하며, 하강만 하고 상승하지 않는다.

### 실용 조언

**Codex**：기본 `Medium`은 대부분 시나리오에 적합；깊은 추론이 필요하면 `High` / `XHigh`로 조정；Plan 모드에 더 높은档위를 별도 설정（계획은 많이 생각할 가치가 있고, 실행 단계는 같은档位일 필요 없다）；档위 조정은 서버에 보내는 문자열 하나만 변경할 뿐, 로컬에서 "더 열심히 하지" 않는다는 것을 기억하라.

**CC**：일상은 `high`（Opus 4.8 기본값）로 충분；어려운 문제를 임시로 심화하려면 프롬프트에 바로 `ultrathink`를 쓰면 되고, 세션 설정을 변경할 필요 없다；Claude가 작업을 자동으로 분해하고 병렬 편성하게 하려면 `ultracode`를 사용；`max`는 신중하게 사용하며, overthinking이 생길 수 있으므로 먼저 소규모로 테스트한다.

**공통**：effort는 모델별로 보정되어 있으며, 같은 档위 이름이 다른 모델에서 같은 양의 연산량을 의미하지 않는다. 모델 전환 후 실제 적용된档위를 확인하라.

---

## 소결

두 섹션을 거쳐, 두 측 제어 면의 방향을 각각 한 문장으로 요약할 수 있다：

**Codex**：권한은 "정책 행렬"（두 개의 확정적 차원의 직교 조합, 오픈소스로 증명 가능）이고, effort는 "전달만 하고 처리하지 않는档위"（어떤档위를 설정하면 그대로 보내고 로컬 처리 없음）. 예측 가능성 우선, 동작 검사 가능.

**CC**：권한은 "모델 심사 + 정책 안전망"（auto mode 분류기 의미 심사, deny 규칙 하드 안전망）이고, effort는 "档위 + 몇 가지 부가"（max 상한 없음、ultracode 백그라운드 편성 연동、ultrathink 임시 심화）. 유연성 우선, 제어 면을 두텁게 만든다.

두 가지 해법에는 옳고 그름이 없고, 다른 시나리오에 적합하다. 동작이 강력하게 예측 가능하고 보안 경계가 기계적으로 검증 가능해야 하면 Codex의 조합 행렬이 더 직접적이다；인적 개입을 줄이고 의미 판단으로 복잡한 경계를 처리하고 싶으면 CC의 분류기 경로가 더 편리하다.

---

## Harness Engineering 시리즈

coding agent의 「플랫폼 층(harness)과 비즈니스 엔지니어링」을 주제로, 편마다 분해한다：

1. [Harness란 정확히 무엇인가](./01-what-is-harness.md) —— 플랫폼 층과 비즈니스 엔지니어링의 경계
2. [복잡한 작업의 Spec은 어떻게 쓰는가](./02-how-to-write-specs.md) —— 멀티 Agent、편성자 진입점、rules / docs / skills 구성
3. [Harness는 어떻게 확장하는가](./03-extending-the-harness.md) —— skill、설정 디렉토리와 hook：CC와 Codex의 두 가지 메커니즘
4. [Harness는 어떻게 agent를 제어하는가](./04-permissions-and-effort.md) —— 권한과 effort：CC와 Codex의 제어 면
5. Harness는 어떻게 긴 작업을 버티는가 —— compact、memory、goal（작성 중）

> 다른 언어：[English](../en/04-permissions-and-effort.md) · [한국어](../ko/04-permissions-and-effort.md) · [日本語](../ja/04-permissions-and-effort.md) · [中文](../zh/04-permissions-and-effort.md)
