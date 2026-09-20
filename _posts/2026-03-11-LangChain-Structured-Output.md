---
title: LangChain으로 구조화된 LLM 출력 제어하기 (TypeScript)
author: abruption
date: 2026-03-11 09:00:00 +0900
categories: [Programming, LangChain]
tags: [LangChain, TypeScript, LLM, Zod, OutputParser]
---

모델에게 JSON으로 답해달라고 시키는 건 쉽다. 실제로 그 응답을 DB에 넣거나 다음 API로 넘기려고 하면 이야기가 달라진다. 따옴표가 어긋나고, 필드 이름이 달라지고, 가끔은 JSON 대신 설명문이 통째로 돌아온다.

LangChain에서는 보통 두 가지 방법을 조합한다.

1. `.withStructuredOutput()` — 모델이 처음부터 스키마에 맞게 출력하도록 강제
2. `OutputFixingParser` — 출력이 스키마와 맞지 않을 경우 LLM이 자동으로 수정

---

## `.withStructuredOutput()` 메서드

이 메서드는 Zod 스키마나 JSON 스키마를 받아서, 모델이 그 구조에 맞는 출력을 내도록 필요한 설정과 출력 파서를 붙인다.

### 기본 사용법

```ts
import { ChatAnthropic } from "@langchain/anthropic"
import { z } from "zod"

const model = new ChatAnthropic({
  model: "claude-3-5-sonnet-20240620",
  temperature: 0
})

const joke = z.object({
  setup: z.string().describe("The setup of the joke"),
  punchline: z.string().describe("The punchline to the joke"),
  rating: z.number().optional().describe("How funny the joke is, from 1 to 10"),
})

const structuredLlm = model.withStructuredOutput(joke, { name: 'joke' })
const response = await structuredLlm.invoke("Tell me a joke about cats")

console.log(response)
// {
//   setup: "Why don't cats play poker in the wild?",
//   punchline: "Too many cheetahs!",
//   rating: 7
// }
```

### 포인트: 스키마 이름(`name`)을 반드시 전달하라

`withStructuredOutput(schema, { name: 'schemaName' })`처럼 이름을 함께 주면 모델이 이 스키마가 무엇을 나타내는지 알 수 있는 단서가 하나 더 생긴다. 스키마 이름 하나로 모든 문제가 해결되는 건 아니지만, 생략할 이유도 별로 없다.

### 포인트: Zod의 `.describe()`를 활용하라

각 필드에 `.describe()`를 붙이면 모델이 어떤 값을 넣어야 하는지 이해하기 쉬워진다. 설명이 없으면 필드 이름만 보고 추측해야 하므로, 이름이 짧거나 도메인 용어일수록 손해가 커진다.

```ts
// 설명 없음 (모델이 필드 이름만으로 추론)
const schema = z.object({
  s: z.string(),
  p: z.string(),
})

// 설명 있음 (권장)
const schema = z.object({
  setup: z.string().describe("The setup of the joke"),
  punchline: z.string().describe("The punchline to the joke"),
})
```

---

## `OutputFixingParser`로 오류 자동 복구

`.withStructuredOutput()`을 사용해도 모든 상황에서 출력이 완벽해지는 건 아니다. 특히 오래된 모델이나 파인튜닝된 소형 모델에서는 파싱 오류가 자주 생길 수 있다.

`OutputFixingParser`는 이런 상황을 위한 안전망이다. 기존 파서가 실패하면 잘못된 출력과 수정 지침을 다시 LLM에 보내서 고친 뒤 재파싱한다.

### 문제 상황

작은따옴표가 들어간 잘못된 JSON을 가정해보자.

```ts
import { z } from "zod";
import { StructuredOutputParser } from "@langchain/core/output_parsers";

const zodSchema = z.object({
  name: z.string().describe("name of an actor"),
  film_names: z
    .array(z.string())
    .describe("list of names of films they starred in"),
});

const parser = StructuredOutputParser.fromZodSchema(zodSchema);

const misformatted = "{'name': 'Tom Hanks', 'film_names': ['Forrest Gump']}";

await parser.parse(misformatted);
// Error: Failed to parse. Text: "{'name': 'Tom Hanks', 'film_names': ['Forrest Gump']}".
// Error: SyntaxError: Expected property name or '}' in JSON at position 1
```

작은따옴표(`'`)는 유효한 JSON이 아니어서 파싱이 실패한다.

### `OutputFixingParser` 적용

```ts
import { ChatAnthropic } from "@langchain/anthropic";
import { OutputFixingParser } from "langchain/output_parsers";

const model = new ChatAnthropic({
  model: "claude-3-sonnet-20240229",
  maxTokens: 512,
  temperature: 0.1,
});

const parserWithFix = OutputFixingParser.fromLLM(model, parser);

const result = await parserWithFix.parse(misformatted);
console.log(result);
// {
//   name: "Tom Hanks",
//   film_names: ["Forrest Gump", "Saving Private Ryan", "Cast Away", "Catch Me If You Can"]
// }
```

`OutputFixingParser.fromLLM(model, parser)`의 두 번째 인수에는 기존 파서를 넣는다. 파싱이 실패하면 내부적으로 다음 순서가 돌아간다.

1. 원본 출력과 기대 형식을 담은 수정 요청 프롬프트 생성
2. 전달된 `model`에 수정 요청
3. 수정된 출력을 원래 파서로 재파싱

### 언제 어떤 방법을 쓸까

| 상황 | 권장 방법 |
|------|----------|
| 최신 모델 (GPT-4, Claude 3.5+) | `.withStructuredOutput()` 단독 사용 |
| 소형/파인튜닝 모델 사용 | `OutputFixingParser` 추가 |
| 프로덕션 환경에서 안정성이 중요한 경우 | 두 방법 함께 사용 |
| 비용 최소화가 중요한 경우 | `.withStructuredOutput()` 단독 (수정 LLM 호출 비용 없음) |

---

## 그래서 어떻게 고를까

LangChain에서 구조화된 출력을 다룰 때 기억할 건 두 가지다.

- **처음부터 제대로**: `.withStructuredOutput()` + Zod 스키마의 `.describe()` 활용
- **실패에 대비**: `OutputFixingParser`로 자동 복구 레이어 추가

사용자 입력을 받아 DB에 저장하거나 다른 API로 넘기는 파이프라인이라면 출력 형식이 흔들리는 순간 뒤 단계도 같이 흔들린다. 다만 `OutputFixingParser`는 실패할 때 LLM을 한 번 더 호출한다. 비용과 지연을 감수할 수 있을 때 붙이는 게 맞고, 최신 모델에서 구조화 출력이 안정적이라면 `.withStructuredOutput()`만으로 끝내도 된다.

## 참고

- [LangChain - How to return structured data from a model](https://js.langchain.com/docs/how_to/structured_output/)
- [LangChain - How to fix OutputParser mistakes](https://js.langchain.com/docs/how_to/output_parser_fixing/)
