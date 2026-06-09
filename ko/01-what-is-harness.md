# Harness란 정확히 무엇인가

## 1. Harness의 정의

### 업계의 권위 있는 정의들

"Harness Engineering"이라는 표현이 2026년에 주목받기 시작했지만, "harness"라는 단어는 점점 더 넓은 의미로 쓰이고 있다. 먼저 일차 출처 몇 가지를 정리하고, 그 다음 내 이해를 이야기하겠다.

**OpenAI**는 [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)（2026-02，Ryan Lopopolo）에서 harness를 다음과 같이 정의한다：

> "the full environment of scaffolding, constraints, and feedback loops that surrounds the agent."

OpenAI의 이 정의에서 Codex harness는 `codex-core`라는 Rust 공유 라이브러리다—agent loop、thread lifecycle、config / auth、sandboxed tool execution이 모두 여기에 포함된다. UI 외피（VS Code 플러그인、CLI）는 harness에 해당하지 않는다.

**Anthropic**은 [Building agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)에서 다음과 같이 말한다：

> "the agent harness that powers Claude Code (the Claude Code SDK) can power many other types of agents, too."

2025-11의 [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)는 harness를 독립적인 엔지니어링 대상으로 논의한다：context compaction、structured handoff、tool sandboxing이 모두 harness의 핵심 설계 요소로 열거된다.

**Simon Willison**은 [How coding agents work](https://simonwillison.net/guides/agentic-engineering-patterns/how-coding-agents-work/)에서 가장 넓은 의미로 사용한다：

> "A coding agent is a piece of software that acts as a harness for an LLM, extending that LLM with additional capabilities that are powered by invisible prompts and implemented as callable tools."

그는 Claude Code、Cursor、Codex CLI 전체를 harness로 본다—UI 외피까지 포함하여.

**LangChain**은 [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)에서 다음 공식을 제시한다：

> "Agent = Model + Harness."

모델은 엔진이고, harness는 그 엔진을 지속적으로 작동하는 운반체로 감싸는 것이다.

이 정의들의 **공통점**은 명확하다：harness는 모델 바깥의 런타임 메커니즘 층이다—scaffolding、제약、피드백 루프、도구 호출、컨텍스트·메모리 관리、agent loop. **차이**는 범위에 있다：OpenAI는 harness를 가장 좁게（SDK / core만）, Simon은 가장 넓게（UI surface 포함）, Anthropic은 그 중간 어딘가로 본다.

### 내 이해

이 글에서는 OpenAI 쪽에 가까운 좁은 정의를 채택하며, 다음 한 문장으로 정의한다：

> **Harness는 Coding Agent의 플랫폼 층(platform layer)이다—컨텍스트 관리、메모리、subagent 편성、skill 메커니즘、도구 호출、hook、실행 클로즈드 루프—이 층을 가리킨다. 비즈니스 엔지니어링(business engineering)은 그 위에 구축되며, harness를 수정해서는 안 된다.**

좁은 정의를 쓰는 이유는 무엇인가? 이 글에서 논의하려는 경계 문제—"무엇이 harness이고, 무엇이 비즈니스 엔지니어링인가"—는 좁은 정의에서만 명확하게 말할 수 있기 때문이다. CLI 전체를 harness로 보면 비즈니스 엔지니어링의 경계가 CLI 밖으로 나가버려 논의할 거리가 없어진다；반대로 harness를 LLM 호출 인터페이스만으로 보면 context / memory / tool이 전부 비즈니스 엔지니어링 영역이 되어 모든 엔지니어링 규율을 비즈니스 측에 떠넘기게 되며, 이것도 현실적이지 않다. 절충점은 바로：**Coding Agent의 SDK / core 층**이다.

### SDD ≠ Harness Engineering，Spec ≠ Harness

이 두 쌍의 개념은 자주 혼동되지만, 위치가 완전히 다르다.

**SDD（Spec-Driven Development）**는 OpenAI의 Sean Grove가 [The New Code](https://www.darekm101.com/articles/the-new-code-sean-grove-openai) 등에서 알린 것으로, **비즈니스 측의 엔지니어링 패러다임**에 관한 것이다：코드 대신 spec을 인간과 AI 협업의 주요 산출물로 삼는다—요구사항 spec、설계 spec、인수 spec이 순서대로 이어지며, 코드는 spec의 파생물이다. SDD는 **what to build**를 다룬다.

**Harness Engineering**은 OpenAI의 Ryan Lopopolo가 2026-02 블로그 포스트에서 정식으로 명명했으며, **agent 플랫폼 층의 엔지니어링 패러다임**에 관한 것이다：context compaction을 어떻게 설계할지、agent loop를 어떻게 관리할지、sandbox를 어떻게 만들지、긴 세션에서 어떻게 안정성을 유지할지. Harness Engineering은 **how the agent runs**를 다룬다.

두 관계：**SDD는 상위 패러다임이고, Harness Engineering은 하위 패러다임이다**. SDD는 "비즈니스 산출물로 어떤 spec을 써야 하는가"를, Harness Engineering은 "Coding Agent가 어떻게 그 spec들을 안정적으로 실행할 것인가"를 다룬다. Harness 없이는 spec을 실행할 수 없고；spec 없이는 harness가 아무리 안정적이어도 공회전에 불과하다.

**Spec과 Harness도 서로 다른 것이다**. Spec은 비즈니스 엔지니어링의 산출물이다—워크플로 주간선、단계 계약、원자적 도구、도메인 지식, 이 비즈니스가 직접 작성하는 것들이 모두 spec의 다양한 형태다. Harness는 spec이 실행되는 데 의존하는 플랫폼이다. 두 관계는 "비즈니스의 spec이 harness의 능력을 호출하는 것"이지, **"spec이 harness의 일부"도 아니고 "harness가 spec의 한 종류"도 아니다**.

이 두 층을 뒤섞으면, "무엇이 비즈니스가 해야 할 일이고 무엇이 플랫폼이 해야 할 일인가"를 영원히 명확히 할 수 없다.

## 2. Harness는 Coding Agent 플랫폼 층이다

나는 현재 Claude Code와 Codex를 사용하고 있다. 이 두 가지는 엔지니어링 관점에서 큰 차이가 없다：로컬 CLI 진입점 + 원격 모델 + 모델이 통제 가능하게 작업을 완료하도록 하는 런타임 메커니즘 세트. 이 런타임 메커니즘 세트가 바로 harness다.

아래 7가지 항목은 Claude Code / Claude Agent SDK를 주요 참조 프레임으로 삼아 정리한 것이다. "skill"과 "hook"은 Claude 생태계의 명칭이고, Codex 측의 대응 개념은 "permissions / sandbox / agent loop hooks"라고 부르며 본질적으로 같고 명칭만 다르다. 이 7가지는 공식 분류가 아니라 실제 편성 경험에서 정리한 것으로, 대부분의 agent 동작을 설명할 수 있다.


**1. 컨텍스트 윈도우 관리.** 어떤 정보가 현재 대화 윈도우에 들어갈지, 언제 압축(compaction)하고 언제 자르고 언제 이전 상태를 복원할지를 결정한다. 하나의 run이 여러 단계에 걸쳐 있을 때, harness는 오래된 세부 사항을 밀어내고 현재 단계에서 봐야 할 사실을 남기는 역할을 한다. 모델 자체는 현재 윈도우만 볼 뿐, 그 뒤의 트레이드오프는 볼 수 없다.

**2. 메모리 관리.** run을 가로지르는 영속 정보：프로젝트 규칙、사용자 선호、재검토 가능한 증거. `CLAUDE.md`、`AGENTS.md`、`.inbox`、`run state` 등이 모두 메모리 인프라에 속한다. harness는 언제 이것들을 윈도우에 주입할지를 결정하고, 비즈니스 엔지니어링은 그것들 안에 무엇을 쓸지를 결정한다.

**3. subagent spawn.** 진입점 agent는 직접 구체적인 작업을 하지 않고, 독립적인 컨텍스트를 가진 subagent에게 작업을 나누어 주며, 자신은 편성·증거 수렴·마무리만 한다. 이것은 중요한 포인트다：subagent는 harness가 제공하는 원시 요소(primitive)이며, 비즈니스 엔지니어링은 "파견"할 수 있을 뿐, 파견 메커니즘을 다시 발명해서는 안 된다.

**4. Skill 메커니즘.** Claude Code의 skill, Codex의 plugin / preset은 본질적으로 같은 것이다—트리거 조건、입력 제약、출력 계약、리스크 표시가 있는 로드 가능한 행동 단위(skill). harness는 skill이 어떻게 발견되고、로드되고、조합되는지를 결정하며, 비즈니스 엔지니어링은 skill 안에 무엇을 쓸지를 결정한다.

**5. 도구 호출.** `git`、`worktree`、shell、빌드 명령、테스트 명령、`MCP` 서버、서드파티 SaaS—모든 부작용은 도구 호출을 통해 발생한다. harness는 프로토콜 층（어떻게 등록하고、어떻게 스케줄링하고、타임아웃 처리、출력 잘라내기）을 담당하고, 비즈니스 엔지니어링은 "어떤 도구를 등록할지"를 담당한다.

**6. Hook 메커니즘.** `SessionStart`、`Stop`、`PreCompact`、`PostCompact` 같은 hook은 harness가 비즈니스 측에 남겨둔 "생명주기 핵심 시점에 개입할 수 있는" 구멍이다. `.inbox` 주입、종료 전 워크플로 클로즈드 루프 검사、압축 전 핵심 증거 보존—이런 것들이 모두 hook에 의존한다.

**7. 실행 클로즈드 루프.** 이것은 앞의 여섯 가지를 연결하는 메타 능력이다：상태 읽기 → 단계 판단 → subagent 파견 → 증거 수렴 → 상태 동기화 → 계속할지 블록할지 결정. 성숙한 harness는 이 클로즈드 루프가 항상 수렴하도록 보장하며, "절반 하다가 조용히 나가버리는" 상황이 발생하지 않는다.

이 일곱 가지의 공통점은：**이것들은 모두 Coding Agent 팀（Anthropic / OpenAI）이 계속 다듬고 있는 것들이며, 비즈니스를 하는 당신이 다시 작성해야 할 것들이 아니다.** 당신의 프로젝트는 이 능력들을 어떻게 "사용"할지 결정할 수 있지만, 이것들을 우회하여 자체 구현을 만들어서는 안 된다.

## 3. 비즈니스 엔지니어링이란 무엇인가：harness 위에 구축된 spec 세트

첫 번째 섹션의 실마리로 돌아가면：비즈니스 엔지니어링의 산출물은 본질적으로 spec 세트다—실행 가능하고, agent에게 파견할 수 있는 엔지니어링 계약. Spec은 harness를 통해 실행되며；harness가 무엇을 실행할지는 spec에 명확히 적혀 있다.

이 spec 세트가 레포지토리 안에서 어떻게 계층화되고、조직되고、파일이 어디에 놓이는지는 별도 글에서 다룰 내용이며, 여기서는 펼치지 않는다. 이 섹션에서는 추상적으로만 명확히 하고 싶다：완전한 비즈니스 spec 세트는 보통 어떤 종류의 내용을 커버해야 하는가, 즉 다음 몇 가지 질문에 답해야 하는가.

**이번 산출물은 어디서 시작하고、어떤 단계를 거치며、언제 끝나는가?** 이것이 워크플로 주간선이다. 그것 자체는 "생각"하지 않고, 단지 선언만 한다：현재 어떤 단계에 있는지、다음에 누구를 파견해야 하는지、어떤 조건에서 멈추는지. 프로젝트 전체에는 보통 하나의 주간선 spec만 필요하며, 그것이 진입점 편성자이지만 그 자체는 agent가 아니다—실제로 "생각"하는 것은 harness가 제공하는 subagent다.

**각 단계는 무엇을 하고、무엇을 산출하며、언제 통과시키는가?** 이것이 단계 계약이다. 각 단계는 명시적인 입력、출력、통과 조건、블록 규칙을 가진다. 단계 사이는 선언적 연결이며, 서로 몰래 달리지 않는다—이것이 SDD와 전통적인 agent loop의 엔지니어링 규율에서 가장 큰 차이점이다. 정확히 몇 단계로 나눌지, 어떻게 나눌지는 프로젝트마다 다르다. "요구사항 분석 / 설계 / 구현 / 리뷰 / 테스트 / 릴리스"는 흔한 골격이지만, 유일한 조합은 아니다.

**단계에서 작업할 때 어떤 원자적 능력을 사용하는가?** 이것이 도구 상자다：레포지토리 스캔、증거 추출、빌드 명령、E2E 테스트 실행、시각적 diff、태스크 시스템 연동……각 원자적 능력은 자신의 입출력에만 책임을 지며, **자신이 어떤 단계에서 호출되는지 알아서는 안 된다**—그래야 여러 단계에서 재사용 가능하며 특정 단계에 묶이지 않는다.

**spec이 판단할 때 어떤 배경에 의존하는가?** 이것이 도메인 지식이다：비즈니스 아키텍처、피해야 할 함정、참고 구현. 이 종류의 내용은 직접 invoke되지 않지만, 다른 모든 spec이 암묵적으로 이것에 의존하여 "이것이 좋은 설계인가"、"함정에 빠지는가"、"참고 구현이 어떻게 생겼는가"를 판단한다.

이 네 종류의 내용을 합치면, "비즈니스 엔지니어링이 한 프로젝트에서 무엇을 써야 하는가"를 정의한다. 그것들에는 몇 가지 공통적인 특징이 있다：

- **전부 비즈니스가 직접 쓰는 것이다**—프로젝트 레포지토리의 일부이며, 프로젝트와 함께 진화한다.
- **전부 harness가 노출하는 원시 요소를 통해 실행된다**—비즈니스 측은 agent loop를 직접 구현하지 않고、컨텍스트 관리를 구현하지 않고、subagent 스케줄링을 구현하지 않으며, 이것들은 모두 harness를 호출한다.
- **서로 참조 관계로 조직된다**—주간선은 단계를 참조하고, 단계는 도구를 참조하며, 모든 것은 배경 지식을 참조한다.

여기까지 보면 명확해진다：**비즈니스 엔지니어링 = 이 spec 세트를 잘 쓰는 것；harness = 이 spec 세트를 실행하는 런타임.** 두 관계는 "호출과 피호출"이지, "수정과 피수정"이 아니다.

## 4. 왜 harness를 수정하면 안 되는가

몇몇 프로젝트에서 문제가 생겼을 때 첫 번째 반응이 "Claude Code의 ×× 메커니즘을 수정할 수 없을까"인 경우를 본 적이 있다：직접 컨텍스트 압축을 인수하거나、자체 subagent 스케줄링을 만들거나、hook을 우회해서 외부 주입기를 작성하는 것. 이런 생각들은 모두 순박한 엔지니어링 직감에서 나온다：나는 더 구체적인 비즈니스 요구가 있고, 플랫폼의 기본 동작으로는 충분하지 않다.

하지만 harness를 우회하는 것은 거의 항상 득보다 실이 크다. 세 가지 이유가 있다.

**첫째, harness 자체가 계속 반복 개발 중이다.** Claude Code가 지난 1년 동안 얼마나 많은 것을 바꿨는지 스스로 알 것이다：컨텍스트 압축 전략、skill graph、subagent 격리、hook 호출 시점、도구 결과 슬리밍……매번 업그레이드에서 공짜로 혜택을 받을 수 있는데, 그 전제는 그 위에 패치를 붙이지 않은 것이다. 일단 패치를 붙이면, 매번 업그레이드마다 호환성을 다시 평가해야 하며, 원래 3초면 되던 업그레이드가 3시간짜리 디버깅이 된다.

**둘째, harness의 복잡도는 직관을 훨씬 초과한다.** 컨텍스트 관리는 단순해 보이지만, 실제로는 윈도우 예산、cache hit、압축 알고리즘、정보 충실도、크로스 run 일관성이 관련된다. "내가 단순화 버전을 직접 쓰겠다"는 200줄 코드처럼 들리지만, 실행하면 코너 케이스가 하나씩 나온다. 이것은 플랫폼 층과 애플리케이션 층의 영원한 비대칭이다—당신은 플랫폼 기본 동작만 보고, 그것이 우회한 수백 가지 엣지 케이스는 볼 수 없다.

오늘 딱 하나의 예를 봤다. 어떤 사람이 자신의 프로젝트에서 "메모리 메커니즘"을 만들었다：레포지토리에서 `memory/` 디렉토리를 유지하고, 매번 agent 세션 시작 시 hook이 `memory/` 안의 모든 파일을 컨텍스트에 전량 주입했다. 표면적으로는 "자체 메모리 시스템 구축"이지만, 몇 번 실행하고 나면 context window가 바로 폭발하고, 중간에 잘리며, 중요한 정보가 밀려나간다.

이것이 전형적인 "harness 무분별 수정"이다.

메모리 관리는 원래 harness의 일곱 가지 능력 중 하나다. Anthropic의 [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)는 글 전체를 통해 그들이 어떻게 하는지 설명한다：진행 파일 구조화 인계、compaction 전략、필요한 정보 보존과 중복 제거, 핵심 목표는 윈도우가 폭발하지 않도록 하는 것. Claude Code도 `CLAUDE.md`、`.inbox`、run state 같은 원시 요소를 제공하며, 주입에는 트레이드오프가 있다：무엇이 윈도우에 들어가고、무엇이 압축되고、무엇이 잘리고、무엇이 외부 저장소에서 지연 로드될지 모두 규칙이 있다.

`memory/` 전량 주입은 이 모든 트레이드오프를 건너뛰고, 플랫폼 팀이 반복적으로 다듬은 능력을 hook 한 줄로 "윈도우가 얼마나 크든 다 쑤셔넣어"로 단순화한 것이다. 결과는 harness가 이미 처리해 준 윈도우 예산 문제를 다시 자기 손에 끌어당기게 되고, 플랫폼 층의 압축/복원 메커니즘이라는 안전망도 없다.

같은 것을 harness의 원시 요소로 하면（`CLAUDE.md`에 프로젝트 규칙 작성、적절한 시점에 hook으로 `.inbox` 주입、run state로 단계 간 필요한 사실 동기화）비용은 몇 줄의 설정이다；직접 다시 쓰면 비용은 context window 폭발 + 플랫폼 업그레이드 혜택 상실 + 이후 유지 보수 부담이다. 전문적인 일은 전문가에게 맡기는 것이며, 비즈니스는 harness의 메커니즘을 수정해서는 안 된다—이것은 설교가 아니라 엔지니어링 경제학이다.

**셋째, harness를 우회하면 SDD의 궤도에서 벗어나게 된다.** SDD의 엔지니어링 규율은 harness가 노출하는 몇 가지 하드 제약에서 나온다：subagent는 컨텍스트를 가로질러 정보를 몰래 가져올 수 없고、단계는 gate를 건너뛰어 몰래 달릴 수 없으며、도구 호출은 증거를 남겨야 한다. 일단 비즈니스 요구를 위해 harness를 우회하면, 이 제약들이 모두 무너진다. 단기적으로는 하나의 제한을 우회한 것 같지만, 장기적으로는 전체 엔지니어링 규율의 기반을 해체한 것이다.

올바른 방법은：**플랫폼 층이 부족한 부분을 만났을 때, 먼저 비즈니스 spec 측에서 커버할 수 있는지 생각해 보라.**

긍정적인 예도 하나 들겠다. 내가 harness 실천을 할 때, 이런 시나리오를 만난 적이 있다：모델이 단계를 넘을 때 가끔 "한 걸음 더 나가려 한다"—분명히 요구사항 증거만 내야 하는데, 모델이 손 놓지 않고 설계까지 하려 한다. 처음에는 "subagent 스케줄링을 패치해서 현재 단계의 일만 하도록 강제할 수 없을까"라고 생각했다. 나중에 이 일이 실제로는 비즈니스 spec 측에서 해결 가능하다는 것을 깨달았다：단계 계약에서 현재 단계의 경계를 명확히 고정하고, gate 검사를 추가하며, subagent가 마무리해야 할 때 hook으로 강제 수렴시키는 것. Harness를 건드리지 않고 동작이 제약되었다.

이 "비즈니스 엔지니어링으로 플랫폼 제한을 커버하는" 능력이, SDD 엔지니어가 가장 연습해야 할 근육이다.

## 5. 한 문장 기억하기

> **비즈니스 엔지니어링의 작업 = 이 spec 세트를 잘 쓰고, harness가 노출하는 원시 요소를 조합하는 것；harness 자체를 수정하지 말라.**

이 문장에는 두 가지 의미가 있다.

위를 보면：harness가 주는 원시 요소를 모두 활용하라. 컨텍스트 관리、subagent、skill、hook、도구 호출, 이것들은 무료 "엔지니어링 능력 애드온"이며, 플랫폼 팀은 당신보다 그것들을 어떻게 구현할지 더 잘 안다.

아래를 보면：당신의 비즈니스 산출물의 차별화는 spec을 어떻게 쓰고 어떻게 조합하는지에 있지, harness를 얼마나 깊이 수정했는지에 있지 않다. 두 팀이 같은 종류의 프로젝트를 한다면, spec이 다르게 생긴 것은 각자의 단계 분할、원자적 도구 선택、도메인 지식 축적의 차이에 있는 것이며—어느 쪽이 Claude Code의 코어를 마음대로 수정했기 때문이 아니다.

이것이 내가 Harness Engineering을 하면서 가장 강하게 느끼는 직감이다：**절제가 엔지니어링 규율의 핵심이다.** 수정하면 안 되는 것을 수정하지 않고, 남긴 에너지를 전부 해야 할 일에 투자한다：비즈니스 측 spec을 잘, 정확하게 쓰는 것.

다음 글에서는 두 가지 관련 문제를 펼쳐서 다룰 것이다：**spec은 정확히 무엇을 써야 하는가? skill은 어떻게 계층화하는가?** 워크플로 주간선、단계 계약、원자적 도구、도메인 지식이 각각 어디에 놓이고、서로 어떻게 참조하는지를, 구체적인 프로젝트의 실행 가능한 구조로 떨어뜨려 설명할 것이다.

---

## Harness Engineering 시리즈

coding agent의 「플랫폼 층(harness)과 비즈니스 엔지니어링」을 주제로, 편마다 분해한다：

1. [Harness란 정확히 무엇인가](./01-what-is-harness.md) —— 플랫폼 층과 비즈니스 엔지니어링의 경계
2. [복잡한 작업의 Spec은 어떻게 쓰는가](./02-how-to-write-specs.md) —— 멀티 Agent、편성자 진입점、rules / docs / skills 구성
3. [Harness는 어떻게 확장하는가](./03-extending-the-harness.md) —— skill、설정 디렉토리와 hook：CC와 Codex의 두 가지 메커니즘
4. [Harness는 어떻게 agent를 제어하는가](./04-permissions-and-effort.md) —— 권한과 effort：CC와 Codex의 제어 면
5. Harness는 어떻게 긴 작업을 버티는가 —— compact、memory、goal（작성 중）

> 다른 언어：[English](../en/01-what-is-harness.md) · [한국어](../ko/01-what-is-harness.md) · [日本語](../ja/01-what-is-harness.md) · [中文](../zh/01-what-is-harness.md)
