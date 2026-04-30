---
title: "자연어 한 줄로 소프트웨어를 만드는 6-Agent 파이프라인 설계기"
author: abruption
date: 2026-04-15 09:00:00 +0900
categories: [Programming, AI]
tags: [multi-agent, langgraph, typescript, langchain, architecture]
---

"주문 관리 시스템을 만들어줘."

이 한 줄을 넣으면 요구사항 정의서, ERD, 백엔드 명세, 프론트엔드 컴포넌트 명세가 쏟아져 나오는 시스템을 만들었다. 6개의 에이전트가 릴레이처럼 산출물을 넘기면서 동작하는 구조인데, 만드는 과정에서 생각보다 많은 삽질이 있었다.

```
사용자 자연어 입력
    ↓
BA 에이전트       → 요구사항 정의서 (PRD)
    ↓
Planner 에이전트  → 정보 구조 (IA), 화면 목록
    ↓
DBA 에이전트      → ERD, DDL, 테이블/컬럼 명세
    ↓
Backend 에이전트  → 비즈니스 오브젝트 명세
    ↓
Frontend 에이전트 → 컴포넌트/컨트롤/서비스 명세
    ↓
Supervisor       → 산출물 통합 및 최종 보고서
```

이 글은 그 과정에서 부딪힌 문제들과, 나름대로 찾아낸 해결책을 정리한 것이다.

---

## BaseAgent — 6개 에이전트의 공통 골격

에이전트가 6개가 되면서 첫 번째로 부딪힌 문제는 코드 중복이었다. 역할은 다르지만 하는 일의 흐름은 똑같다. 시스템 프롬프트 설정하고, LLM 호출하고, 출력 검증하고, 산출물 저장하고. BA 에이전트에서 만든 재시도 로직을 DBA 에이전트에도 복붙하고 있는 자신을 발견했을 때 뭔가 잘못됐다는 걸 깨달았다.

결국 `BaseAgent`라는 추상 클래스를 만들었다. GoF의 Template Method 패턴인데, 공통 흐름은 건드리지 않고 에이전트마다 달라지는 부분만 오버라이드하도록 했다.

```ts
abstract class BaseAgent {
  async run(state: AgentState): Promise<AgentState> {
    const prompt = this.getSystemPrompt();
    const tools = this.getToolsCreator();
    const preprocessed = this.handlePreModel(state);
    const result = await this.callLLM(prompt, preprocessed, tools);
    const postprocessed = this.handlePostModel(result);
    return postprocessed;
  }

  abstract getSystemPrompt(): string;
  abstract getToolsCreator(): Tool[];
  abstract getAdditionalStateFields(): ZodSchema;
  protected handlePreModel(state: AgentState): AgentState { return state; }
  protected handlePostModel(result: LLMResult): AgentState { return result; }
  protected shouldLimitToolCalls(): boolean { return false; }
}
```

오버라이드 포인트는 6개다. `getSystemPrompt()`로 역할을 정의하고, `getToolsCreator()`로 전용 도구를 반환하고, `getAdditionalStateFields()`로 Zod 스키마를 확장한다. `handlePreModel()`은 Provider별 메시지 포맷 변환, `handlePostModel()`은 빈 응답 복구와 무한 루프 감지를 담당한다.

이 구조의 진짜 가치는 나중에 Frontend 에이전트를 추가할 때 체감했다. 6개 메서드만 구현하니까 스트리밍, 재시도, 상태 복구가 전부 따라왔다. 하루 만에 통합 완료.

---

## 상태가 날아가는 문제 — interrupt()로 해결

초기 설계에서 Supervisor가 하위 에이전트를 직접 HTTP로 호출했다. 이게 DBA 에이전트가 ERD 생성하다 3분째에 터지기 전까지는 잘 동작했다. 문제는 하위 에이전트가 죽으면 Supervisor의 전체 상태가 같이 날아간다는 거였다. BA와 Planner가 이미 성공적으로 만들어놓은 PRD, IA까지 다 없어지고 처음부터 재실행. 비용도 비용이지만 시간이 치명적이었다.

LangGraph의 `interrupt()` 메커니즘으로 전환하면서 이 문제가 해결됐다.

```ts
function supervisorNode(state: SupervisorState) {
  const nextAgent = determineNextAgent(state);

  const result = interrupt({
    agent: nextAgent,
    input: state.currentArtifacts,
  });

  return {
    ...state,
    artifacts: [...state.artifacts, result],
  };
}
```

`interrupt()`가 실행을 일시 중지하고, Redis 체크포인트에 현재 상태를 저장한다. 하위 에이전트가 실패해도 Supervisor는 마지막 성공 지점에서 재개할 수 있다. TTL은 12시간으로 잡았는데, 에이전트 6개가 순차로 돌면 전체 파이프라인이 꽤 오래 걸리기 때문에 넉넉하게 잡은 것이다.

---

## 미들웨어 — 방어 코드를 비즈니스 로직에서 분리하기

LLM은 생각보다 자주 말을 안 듣는다. API가 갑자기 429를 뱉기도 하고, 같은 프롬프트에 전혀 다른 형식으로 답하기도 한다. 처음에는 이런 방어 코드를 에이전트 안에 넣었는데, 금방 스파게티가 됐다. try-catch 안에 retry가 있고 그 안에 또 fallback이 있고...

해법은 Express.js의 미들웨어에서 힌트를 얻었다. 각 방어 로직을 독립적인 미들웨어로 만들고 조립하는 방식이다.

```ts
const agent = applyMiddleware(baseAgent, [
  modelRetryMiddleware({ maxRetries: 3, backoff: "exponential" }),
  modelFallbackMiddleware({ fallbackProvider: "anthropic" }),
  modelCallLimitMiddleware({ maxCalls: 50 }),
  promptCachingMiddleware(),
  humanInTheLoopMiddleware({ requireApproval: ["dba", "backend"] }),
  hooksMiddleware({ checkpointTTL: "12h" }),
]);
```

`modelRetryMiddleware`는 LLM API 일시 장애 시 지수 백오프로 재시도한다. `modelFallbackMiddleware`는 재시도를 해도 안 되면 아예 다른 Provider(예: OpenAI → Anthropic)로 전환한다. `modelCallLimitMiddleware`는 에이전트가 도구를 50번 넘게 호출하면 강제로 끊는다 — LLM이 도구 호출 루프에 빠지는 경우가 의외로 잦다.

여기서 배운 게 하나 있는데, 미들웨어 순서가 중요하다. `retry`가 `fallback` 앞에 있어야 같은 Provider에서 먼저 재시도하고, 그래도 안 되면 대체 Provider로 넘어간다. 순서를 반대로 했다가 OpenAI가 잠깐 느려질 때마다 Anthropic으로 전환되는 바람에 비용이 폭발한 적이 있다.

---

## Semantic Validation — JSON Schema만으로는 부족하다

이건 실제로 겪어보지 않으면 와닿지 않는 문제다. DBA 에이전트가 만든 ERD가 JSON Schema는 깔끔하게 통과하는데, 테이블 이름이 camelCase고 FK가 존재하지 않는 컬럼을 참조하고 있었다. 이 ERD를 받은 Backend 에이전트가 엉뚱한 BO를 만들고, Frontend 에이전트는 그걸 토대로 또 엉뚱한 컴포넌트를 만든다. 파이프라인이 길수록 초반의 작은 오류가 후반에서 눈덩이처럼 불어난다.

그래서 Semantic Validator를 15종 만들었다. `NamingConventionValidator`는 snake_case 준수를 확인하고, `ReferenceIntegrityValidator`는 FK가 실제로 존재하는 테이블·컬럼을 참조하는지 확인한다. `TypeConsistencyValidator`는 FK 양쪽의 데이터 타입이 일치하는지까지 본다.

검증 흐름은 이렇다:

```
LLM 출력 → SemanticValidator.validate()
    score ≥ 9.0   → 통과
    score < 9.0   → 피드백 포함 재생성 (최대 3회)
    3회 후 ≥ 8.0  → 강제 통과
```

마지막 줄이 핵심이다. 3번 재시도해도 9.0이 안 나오면 8.0 이상에서 그냥 통과시킨다. 처음에는 무조건 9.0 이상을 요구했는데, 에이전트가 같은 실수를 반복하면서 무한 루프에 빠지는 일이 생겼다. 완벽한 출력을 기다리다 시스템 전체가 멈추는 것보다는, 80점짜리라도 다음 단계로 넘기는 게 현실적이었다.

---

## 에이전트마다 다른 LLM 쓰기

에이전트 6개가 전부 같은 LLM을 쓸 이유가 없다. BA 에이전트는 긴 요구사항을 이해하고 구조화해야 하니 강력한 모델이 필요하고, 도구 내부에서 간단한 변환을 하는 보조 호출은 가볍고 빠른 모델이 효율적이다.

```ts
const agentConfig = {
  ba: {
    mainLLM: "claude-sonnet-4-6",
    toolLLM: "claude-haiku-4-5",
  },
  dba: {
    mainLLM: "gpt-5",
    toolLLM: "gpt-4.1-mini",
  },
};
```

환경 변수 하나로 전체 Provider를 전환할 수도 있어서, "이번 파이프라인은 전부 Anthropic으로 돌려보자" 같은 비교 실험이 간단하다. 실제로 DBA 에이전트는 GPT-5가 ERD 품질이 더 좋았고, BA 에이전트는 Claude가 요구사항 구조화를 더 잘했다. 이런 차이는 직접 돌려봐야 알 수 있다.

---

## 프롬프트 분리 — 배포 없이 프롬프트 교체

에이전트 시스템을 운영하면 프롬프트 수정이 정말 잦다. 근데 프롬프트가 코드에 하드코딩되어 있으면 수정할 때마다 빌드-배포를 해야 한다. 하루에 프롬프트를 열 번 고칠 때도 있는데, 매번 배포하는 건 말이 안 된다.

프롬프트를 외부 저장소에서 버전 관리하고, 캐싱 레이어를 거쳐 읽어오도록 바꿨다. `/refresh-prompts` 엔드포인트를 호출하면 캐시가 갱신되면서 다음 에이전트 실행부터 새 프롬프트가 적용된다. 배포 없이.

이건 특히 프롬프트 엔지니어링 담당자가 개발자한테 "이거 좀 바꿔주세요" 하고 기다리는 병목을 없애는 데 효과적이었다.

---

## 돌아보면

에이전트 하나를 만드는 건 어렵지 않다. 프롬프트 잘 쓰고 LLM 호출하면 된다. 근데 6개를 파이프라인으로 엮으면 전혀 다른 종류의 문제가 나온다.

5단계에서 터지면 처음부터 다시 돌려야 했던 초기의 고통이 체크포인트 설계로 이어졌고, LLM이 내뱉는 "형식은 맞지만 의미는 틀린" 출력이 Semantic Validation의 출발점이 됐다. 완벽한 9.0점을 요구하다 시스템이 멈춘 경험이 "8.0이면 그냥 가자"는 실용적인 판단으로 바뀌었다.

결국 Multi-Agent 시스템은 "LLM을 얼마나 잘 쓰느냐"보다 "LLM이 실패할 때를 얼마나 잘 대비하느냐"에 가깝다는 생각이 든다.

---

## 참고 링크

- [LangGraph - Multi-Agent Systems](https://langchain-ai.github.io/langgraphjs/concepts/multi_agent/)
- [LangGraph - Human-in-the-loop](https://langchain-ai.github.io/langgraphjs/concepts/human_in_the_loop/)
- [Template Method Pattern - Refactoring Guru](https://refactoring.guru/design-patterns/template-method)
