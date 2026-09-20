---
title: LangChain 프롬프트 템플릿 패턴 정리 (TypeScript) — Few-shot, Partial, Pipeline
author: abruption
date: 2026-03-11 10:00:00 +0900
categories: [Programming, LangChain]
tags: [LangChain, TypeScript, LLM, PromptTemplate, Few-shot]
---

프롬프트를 문자열 하나로 시작하는 건 쉽다. 예시 하나를 붙이고 변수를 하나 더 받는 것도 처음에는 별일 아니다. 문제는 기능이 늘어난 뒤다. 어떤 값이 어디서 들어오는지 찾는 데 코드보다 프롬프트를 더 오래 읽게 된다.

LangChain의 프롬프트 템플릿은 이 문제를 꽤 깔끔하게 나눠준다. 여기서는 자주 쓰게 되는 세 가지 패턴을 예제로 비교한다.

1. **FewShotPromptTemplate** — 예시를 포함한 프롬프트
2. **Partial 포맷** — 변수를 단계별로 주입
3. **PipelinePromptTemplate** — 프롬프트 모듈화 및 재조합

---

## 1. FewShotPromptTemplate — 예시로 모델 행동 유도

Few-shot prompting은 모델에게 응답 형식과 추론 방식을 예시로 보여주는 방식이다. 제로샷(zero-shot)보다 일정한 출력을 얻고 싶을 때 유용하다.

LangChain에서는 `FewShotPromptTemplate`으로 이 예시들을 구조화할 수 있다.

### 구성 순서

**Step 1. 예시를 포맷할 템플릿 정의**

```ts
import { PromptTemplate } from "@langchain/core/prompts";

const examplePrompt = PromptTemplate.fromTemplate(
  "Question: {question}\n{answer}"
);
```

**Step 2. 예시 데이터 작성**

각 예시는 템플릿의 변수와 같은 키를 가진 객체다.

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

포맷 결과는 다음처럼 구성된다.

```
Question: Who lived longer, Muhammad Ali or Alan Turing?
Are follow up questions needed here: Yes.
...

Question: When was the founder of craigslist born?
Are follow up questions needed here: Yes.
...

Question: Who was the father of Mary Ball Washington?
```

`suffix`가 실제 질문이고, 예시들은 그 앞에 자동으로 붙는다. 모델은 이 패턴을 보고 비슷한 형식으로 답을 만든다.

### 언제 사용하면 효과적인가

- 출력 형식이 복잡하거나 특수한 경우
- 도메인 특화된 추론 방식을 강제해야 할 경우
- 제로샷으로 일관성이 떨어지는 경우

---

## 2. Partial 포맷 — 변수를 단계별로 주입

프롬프트가 여러 변수를 요구하는데, 그 변수를 체인의 서로 다른 단계에서 얻는 경우가 있다. 시스템 설정은 앱 초기화 때 정해지고 사용자 입력은 실행할 때 들어오는 식이다.

이럴 때 모든 변수를 한 번에 모으는 대신 `.partial()`로 이미 알고 있는 값을 먼저 바인딩할 수 있다.

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

체인의 여러 단계에 걸쳐 변수를 넘기는 복잡성이 줄고, 각 단계의 책임도 조금 더 분명해진다.

---

## 3. PipelinePromptTemplate — 프롬프트 모듈화

프롬프트가 길어지면 관리가 어려워진다. 시스템 지시사항, 예시, 실제 질문이 하나의 긴 문자열에 섞이면 작은 수정에도 전체를 다시 읽어야 한다.

`PipelinePromptTemplate`은 프롬프트를 재사용 가능한 서브 프롬프트로 나눠서 조합한다.

### 구성 방식

핵심 요소는 두 가지다.
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

`introductionPrompt`, `examplePrompt`, `startPrompt`를 각각 다른 파이프라인에서 재사용할 수 있다. 같은 `startPrompt`에 페르소나만 바꾸거나, 예시만 교체하는 식이다.

---

## 세 패턴 비교

| 패턴 | 핵심 클래스 | 언제 사용하나 |
|------|-----------|-------------|
| Few-shot | `FewShotPromptTemplate` | 출력 형식/추론 방식을 예시로 유도할 때 |
| Partial | `prompt.partial()` | 변수를 체인의 서로 다른 단계에서 주입할 때 |
| Pipeline | `PipelinePromptTemplate` | 긴 프롬프트를 역할별로 분리·재사용할 때 |

---

## 정리

세 패턴을 모두 써야 하는 건 아니다. 예시로 출력 형식을 잡아야 하면 Few-shot을 쓰고, 값이 단계별로 들어오면 Partial을 쓰면 된다. 프롬프트가 여러 조각으로 커졌을 때 Pipeline을 꺼내면 된다.

처음부터 거대한 프롬프트 시스템을 만들 필요도 없다. 문자열 하나로 시작한 뒤, 어디가 자주 바뀌는지 보면서 필요한 부분만 분리하는 편이 오히려 관리하기 쉽다.

## 참고

- [LangChain - How to use few shot examples](https://js.langchain.com/docs/how_to/few_shot_examples)
- [LangChain - How to partially format prompt templates](https://js.langchain.com/docs/how_to/prompts_partial/)
- [LangChain - How to compose prompts together](https://js.langchain.com/docs/how_to/prompts_composition/)
