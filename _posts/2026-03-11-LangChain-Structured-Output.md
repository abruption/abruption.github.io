---
title: LangChain으로 구조화된 LLM 출력 제어하기 (TypeScript)
author: abruption
date: 2026-03-11 09:00:00 +0900
categories: [Programming, LangChain]
tags: [LangChain, TypeScript, LLM, Zod, OutputParser]
---

LLM을 프로덕션에 도입할 때 가장 자주 맞닥뜨리는 문제 중 하나는 **출력 형식의 불안정성**입니다. 모델에게 JSON으로 응답해달라고 요청해도 따옴표가 어긋나거나, 필드 이름이 달라지거나, 아예 다른 형식으로 응답하는 경우가 생깁니다.

LangChain은 이 문제를 두 가지 방법으로 해결합니다.

1. `.withStructuredOutput()` — 모델이 처음부터 스키마에 맞게 출력하도록 강제
2. `OutputFixingParser` — 출력이 스키마와 맞지 않을 경우 LLM이 자동으로 수정

---

## `.withStructuredOutput()` 메서드

이 메서드는 Zod 스키마 또는 JSON 스키마를 전달받아, 모델이 해당 스키마와 일치하는 구조화된 출력을 반환하도록 필요한 파라미터와 출력 파서를 자동으로 설정합니다.

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

`withStructuredOutput(schema, { name: 'schemaName' })` 형태로 이름을 함께 전달하면, 모델에게 이 스키마가 무엇을 나타내는지 추가 컨텍스트를 제공할 수 있습니다. 공식 문서에서도 이름을 전달하면 성능이 향상된다고 명시하고 있습니다.

### 포인트: Zod의 `.describe()`를 활용하라

각 필드에 `.describe()`로 설명을 붙이면 모델이 해당 필드에 어떤 값을 넣어야 하는지 더 잘 이해합니다. 스키마만 정의하고 설명을 생략하면 필드 이름만으로 모델이 추론해야 하므로 정확도가 떨어질 수 있습니다.

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

`.withStructuredOutput()`을 사용하더라도, 모든 상황에서 모델이 완벽한 출력을 보장하지는 않습니다. 특히 오래된 모델이나 파인튜닝된 소형 모델을 사용할 경우 파싱 오류가 빈번하게 발생할 수 있습니다.

`OutputFixingParser`는 이런 상황을 대비한 안전망입니다. 기존 파서가 실패하면, 잘못된 출력과 수정 지침을 담아 다른 LLM을 호출하여 오류를 자동으로 복구합니다.

### 문제 상황

작은따옴표를 사용한 잘못된 JSON이 들어왔다고 가정합니다.

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

작은따옴표(`'`)는 유효한 JSON이 아니기 때문에 파싱이 실패합니다.

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

`OutputFixingParser.fromLLM(model, parser)`에서 두 번째 인수로 기존 파서를 전달합니다. 파싱이 실패하면 내부적으로 다음과 같은 흐름으로 동작합니다.

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

## 정리

LangChain에서 구조화된 출력을 다루는 핵심은 두 가지입니다.

- **처음부터 제대로**: `.withStructuredOutput()` + Zod 스키마의 `.describe()` 활용
- **실패에 대비**: `OutputFixingParser`로 자동 복구 레이어 추가

특히 사용자 입력을 기반으로 LLM이 응답을 생성하고, 그 결과를 DB에 저장하거나 다른 API로 전달하는 파이프라인에서는 출력 형식의 안정성이 전체 시스템의 신뢰성과 직결됩니다. 두 방법을 함께 적용하면 파싱 실패로 인한 예외를 효과적으로 줄일 수 있습니다.

## 참고

- [LangChain - How to return structured data from a model](https://js.langchain.com/docs/how_to/structured_output/)
- [LangChain - How to fix OutputParser mistakes](https://js.langchain.com/docs/how_to/output_parser_fixing/)
