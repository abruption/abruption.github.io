---
title: LangChain 프롬프트 템플릿 패턴 정리 (TypeScript) — Few-shot, Partial, Pipeline
author: abruption
date: 2026-03-11 10:00:00 +0900
categories: [Programming, LangChain]
tags: [LangChain, TypeScript, LLM, PromptTemplate, Few-shot]
---

LLM 애플리케이션에서 프롬프트는 코드와 동일한 수준으로 관리해야 합니다. 하드코딩된 문자열로 시작했다가 기능이 복잡해지면 프롬프트가 스파게티가 되는 경험을 하게 됩니다.

LangChain의 프롬프트 템플릿 시스템은 이 문제를 구조적으로 해결합니다. 이 글에서는 실무에서 자주 사용하는 세 가지 패턴을 정리합니다.

1. **FewShotPromptTemplate** — 예시를 포함한 프롬프트
2. **Partial 포맷** — 변수를 단계별로 주입
3. **PipelinePromptTemplate** — 프롬프트 모듈화 및 재조합

---

## 1. FewShotPromptTemplate — 예시로 모델 행동 유도

Few-shot prompting은 모델에게 응답 형식과 추론 방식을 예시로 보여주는 기법입니다. 제로샷(zero-shot)보다 일관된 출력을 유도할 때 효과적입니다.

LangChain에서는 `FewShotPromptTemplate`으로 예시를 구조화할 수 있습니다.

### 구성 순서

**Step 1. 예시를 포맷할 템플릿 정의**

```ts
import { PromptTemplate } from "@langchain/core/prompts";

const examplePrompt = PromptTemplate.fromTemplate(
  "Question: {question}\n{answer}"
);
```

**Step 2. 예시 데이터 작성**

각 예시는 템플릿의 변수와 일치하는 키를 가진 객체입니다.

```ts
const examples = [
  {
    question: "Who lived longer, Muhammad Ali or Alan Turing?",
    answer: `
Are follow up questions needed here: Yes.
Follow up: How old was Muhammad Ali when he died?
Intermediate answer: Muhammad Ali was 74 years old when he died.
Follow up: How old was Alan Turing when he died?
Intermediate answer: Alan Turing was 41 years old when he died.
So the final answer is: Muhammad Ali
    `,
  },
  {
    question: "When was the founder of craigslist born?",
    answer: `
Are follow up questions needed here: Yes.
Follow up: Who was the founder of craigslist?
Intermediate answer: Craigslist was founded by Craig Newmark.
Follow up: When was Craig Newmark born?
Intermediate answer: Craig Newmark was born on December 6, 1952.
So the final answer is: December 6, 1952
    `,
  },
];
```

**Step 3. FewShotPromptTemplate 조립**

```ts
import { FewShotPromptTemplate } from "@langchain/core/prompts";

const prompt = new FewShotPromptTemplate({
  examples,
  examplePrompt,
  suffix: "Question: {input}",
  inputVariables: ["input"],
});

const formatted = await prompt.format({
  input: "Who was the father of Mary Ball Washington?",
});

console.log(formatted.toString());
```

포맷 결과는 다음과 같이 구성됩니다.

```
Question: Who lived longer, Muhammad Ali or Alan Turing?
Are follow up questions needed here: Yes.
...

Question: When was the founder of craigslist born?
Are follow up questions needed here: Yes.
...

Question: Who was the father of Mary Ball Washington?
```

`suffix`가 실제 질문이고, 예시들은 그 앞에 자동으로 삽입됩니다. 모델은 이 패턴을 보고 동일한 추론 방식으로 답변을 생성합니다.

### 언제 사용하면 효과적인가

- 출력 형식이 복잡하거나 특수한 경우
- 도메인 특화된 추론 방식을 강제해야 할 경우
- 제로샷으로 일관성이 떨어지는 경우

---

## 2. Partial 포맷 — 변수를 단계별로 주입

프롬프트 템플릿이 여러 변수를 요구하는데, 그 변수들을 체인의 서로 다른 단계에서 얻는 경우가 있습니다. 예를 들어 시스템 설정 변수는 앱 초기화 시 고정되지만, 사용자 입력 변수는 런타임에 결정되는 식입니다.

이럴 때 모든 변수를 한 곳에 모아 전달하는 대신, `.partial()`로 먼저 알려진 변수를 먼저 바인딩할 수 있습니다.

```ts
import { PromptTemplate } from "langchain/prompts";

const prompt = new PromptTemplate({
  template: "{foo}{bar}",
  inputVariables: ["foo", "bar"],
});

// foo를 먼저 바인딩
const partialPrompt = await prompt.partial({
  foo: "foo",
});

// 나중에 bar만 전달
const formattedPrompt = await partialPrompt.format({
  bar: "baz",
});

console.log(formattedPrompt);
// foobaz
```

### 실무 활용 예시

```ts
// 앱 초기화 시 시스템 언어/지역 설정 바인딩
const basePrompt = new PromptTemplate({
  template: "You are a helpful assistant. Language: {language}. User query: {query}",
  inputVariables: ["language", "query"],
});

const koreanPrompt = await basePrompt.partial({ language: "Korean" });

// 이후 요청마다 query만 전달
const result = await koreanPrompt.format({ query: "오늘 날씨 어때?" });
```

체인의 여러 단계에 걸쳐 변수를 전달하는 복잡성을 줄이고, 책임을 명확하게 분리할 수 있습니다.

---

## 3. PipelinePromptTemplate — 프롬프트 모듈화

프롬프트가 길어지면 관리가 어려워집니다. 특히 시스템 지시사항, 예시, 실제 질문처럼 역할이 다른 섹션이 하나의 긴 문자열에 뒤섞이면 수정할 때마다 전체를 읽어야 합니다.

`PipelinePromptTemplate`은 프롬프트를 재사용 가능한 서브 프롬프트로 분리하고 조합합니다.

### 구성 방식

두 가지 핵심 요소가 있습니다.
- `finalPrompt`: 서브 프롬프트들을 조합하는 최종 템플릿
- `pipelinePrompts`: 이름과 서브 프롬프트 쌍의 배열

```ts
import {
  PromptTemplate,
  PipelinePromptTemplate,
} from "@langchain/core/prompts";

// 최종 템플릿 — 서브 프롬프트를 변수처럼 참조
const fullPrompt = PromptTemplate.fromTemplate(`{introduction}

{example}

{start}`);

// 서브 프롬프트 1: 페르소나 설정
const introductionPrompt = PromptTemplate.fromTemplate(
  `You are impersonating {person}.`
);

// 서브 프롬프트 2: 예시 대화
const examplePrompt = PromptTemplate.fromTemplate(
  `Here's an example of an interaction:
Q: {example_q}
A: {example_a}`
);

// 서브 프롬프트 3: 실제 질문
const startPrompt = PromptTemplate.fromTemplate(
  `Now, do this for real!
Q: {input}
A:`
);

const composedPrompt = new PipelinePromptTemplate({
  pipelinePrompts: [
    { name: "introduction", prompt: introductionPrompt },
    { name: "example", prompt: examplePrompt },
    { name: "start", prompt: startPrompt },
  ],
  finalPrompt: fullPrompt,
});
```

### 사용

```ts
const formattedPrompt = await composedPrompt.format({
  person: "Elon Musk",
  example_q: `What's your favorite car?`,
  example_a: "Tesla",
  input: `What's your favorite social media site?`,
});

console.log(formattedPrompt);
```

출력 결과:

```
You are impersonating Elon Musk.

Here's an example of an interaction:
Q: What's your favorite car?
A: Tesla

Now, do this for real!
Q: What's your favorite social media site?
A:
```

### 재사용성의 이점

`introductionPrompt`, `examplePrompt`, `startPrompt` 각각을 다른 파이프라인에서 재사용할 수 있습니다. 예를 들어, 같은 `startPrompt`를 다른 페르소나(`introductionPrompt`)와 조합하거나, 예시만 교체하는 변형 파이프라인을 쉽게 만들 수 있습니다.

---

## 세 패턴 비교

| 패턴 | 핵심 클래스 | 언제 사용하나 |
|------|-----------|-------------|
| Few-shot | `FewShotPromptTemplate` | 출력 형식/추론 방식을 예시로 유도할 때 |
| Partial | `prompt.partial()` | 변수를 체인의 서로 다른 단계에서 주입할 때 |
| Pipeline | `PipelinePromptTemplate` | 긴 프롬프트를 역할별로 분리·재사용할 때 |

---

## 정리

LangChain의 프롬프트 템플릿은 단순한 문자열 포맷팅 도구가 아닙니다. 복잡한 LLM 파이프라인에서 프롬프트를 **모듈화**, **단계적 조립**, **예시 기반 제어**할 수 있는 구조를 제공합니다.

특히 서비스 규모가 커질수록 프롬프트 관리의 복잡성도 함께 커지기 때문에, 초기부터 이 패턴들을 적용하면 나중에 유지보수 비용을 크게 줄일 수 있습니다.

## 참고

- [LangChain - How to use few shot examples](https://js.langchain.com/docs/how_to/few_shot_examples)
- [LangChain - How to partially format prompt templates](https://js.langchain.com/docs/how_to/prompts_partial/)
- [LangChain - How to compose prompts together](https://js.langchain.com/docs/how_to/prompts_composition/)
