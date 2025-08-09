## LangGraph (TypeScript)에서 Structured Output 사용 가이드

### 개요
Structured output은 LLM 응답을 사전에 정의한 스키마에 맞춰 안정적으로 파싱하는 기법이다. TypeScript 환경에서는 보통 zod 스키마를 정의하고, LLM 클라이언트에 structured output을 활성화한 뒤 LangGraph 노드에서 호출해 상태에 반영한다.

무엇인지
- 모델 출력을 스키마(zod 등)로 강제해 예상 가능한 객체 형태로 받는 방식
- JSON 포맷 강제, 필드 타입 검증, 기본값 적용 등을 통해 후처리 비용과 오류를 줄임

언제 사용하는지
- 외부 API, DB 스키마, 분석 파이프라인 등과의 연동이 JSON 형태를 요구할 때
- 감성, 카테고리, 엔티티 등 분류·추출 작업 결과를 안정적으로 다룰 때
- 에이전트 간 상태 교환이나 툴 호출 파라미터를 자동 구성해야 할 때
- 평가 리포트, 감사 로그처럼 재현성과 일관성이 중요한 산출물이 필요할 때

### zod 간단 소개
- TypeScript 런타임 스키마/검증 라이브러리다. 선언한 스키마로 입력을 검증하고, 동일 정의에서 타입을 추론해 빌드타임과 런타임을 일치시킨다.
- 장점: 런타임 검증(parse/safeParse), 기본값(default), 변환(transform), 풍부한 에러, describe를 통한 LLM 지침 제공.
```ts
import { z } from "zod";

const UserSchema = z.object({
  id: z.string(),
  name: z.string().min(1),
  age: z.number().int().nonnegative().optional(),
});
type User = z.infer<typeof UserSchema>;

const user: User = UserSchema.parse({ id: "u_123", name: "Lee" });
```

### 설치
```bash
pnpm add @langchain/google-genai @langchain/langgraph zod
# 또는
npm i @langchain/google-genai @langchain/langgraph zod
```

환경 변수 설정
```bash
export GOOGLE_API_KEY=your_api_key
```

### 기본 스키마 정의와 LLM 구성
```ts
import { z } from "zod";
import { ChatGoogleGenerativeAI } from "@langchain/google-genai";

// 1) zod 스키마 정의
export const AnalysisSchema = z.object({
  title: z.string().describe("짧은 제목 요약"),
  sentiment: z.enum(["positive", "neutral", "negative"]).describe("감성 분류"),
  entities: z.array(z.string()).default([]).describe("본문에서 추출한 엔티티 목록"),
});
export type Analysis = z.infer<typeof AnalysisSchema>;

// 2) LLM 인스턴스 구성 (Gemini)
const baseModel = new ChatGoogleGenerativeAI({
  model: "gemini-2.5-flash",
  temperature: 0,
});

// 3) structured output 활성화
export const analysisModel = baseModel.withStructuredOutput(AnalysisSchema);

// 사용 예시
async function runOnce(userInput: string) {
  const result = await analysisModel.invoke(userInput);
  // result의 타입은 Analysis로 안전하게 보장됨
  return result;
}
```

### LangGraph 상태와 노드에 통합
```ts
import { Annotation, StateGraph, START, END } from "@langchain/langgraph";
import { analysisModel, Analysis } from "./model"; // 위 예제 분리 가정

// 1) 그래프 상태 정의 (Annotation 사용)
const GraphState = Annotation.Root({
  input: Annotation<string>(),
  analysis: Annotation<Analysis | undefined>(),
});

// 2) 노드 구현: structured output 모델 호출 후 상태 업데이트
async function extractNode(state: typeof GraphState.State) {
  const res = await analysisModel.invoke(state.input);
  // 반환 객체의 키만 그래프 상태에 머지됨
  return { analysis: res } as Partial<typeof GraphState.State>;
}

// 3) 그래프 구성 및 컴파일
const builder = new StateGraph(GraphState)
  .addNode("extract", extractNode)
  .addEdge(START, "extract")
  .addEdge("extract", END);

export const graph = builder.compile();

// 4) 실행
async function main() {
  const output = await graph.invoke({ input: "LangGraph와 structured output을 이용해 감성 및 엔티티를 추출해줘" });
  console.log(output.analysis);
}

main();
```

### 프롬프트 제어 팁
- 스키마의 필드에 describe를 활용해 의도를 명확히 전달한다.
- 모델 온도는 0에 가깝게 두면 구조화된 형식을 더 잘 준수한다.
- 입력이 장문일 경우 핵심 지침을 시스템 메시지나 스키마 설명에 담아 일관성을 높인다.

### 에러 처리와 유효성
- withStructuredOutput은 내부적으로 모델 출력을 스키마로 검증한다. 파싱 실패 시 예외가 발생할 수 있으므로 try/catch로 감싼다.
```ts
try {
  const result = await analysisModel.invoke(userInput);
  // 정상 처리
} catch (err) {
  // 모델 출력이 스키마를 위반하거나 통신 오류 등
  // 재시도 로직 또는 폴백 처리
}
```

### 복합 스키마 예시
```ts
const ProductSchema = z.object({
  name: z.string(),
  price: z.number().nonnegative(),
  tags: z.array(z.string()).default([]),
  specs: z.object({
    weightKg: z.number().optional(),
    color: z.string().optional(),
  }).optional(),
});

const productModel = baseModel.withStructuredOutput(ProductSchema);
const product = await productModel.invoke("다음 제품 설명을 구조화해줘: ...");
```

### 그래프에서 여러 노드가 구조화 출력에 기여하는 패턴
여러 노드가 각기 다른 스키마를 채우고, 마지막에 병합하는 방식이 흔하다.
```ts
const State2 = Annotation.Root({
  input: Annotation<string>(),
  analysis: Annotation<Analysis | undefined>(),
  product: Annotation<z.infer<typeof ProductSchema> | undefined>(),
});

async function extractAnalysis(state: typeof State2.State) {
  const res = await analysisModel.invoke(state.input);
  return { analysis: res };
}

async function extractProduct(state: typeof State2.State) {
  const res = await productModel.invoke(state.input);
  return { product: res };
}

const builder2 = new StateGraph(State2)
  .addNode("analysis", extractAnalysis)
  .addNode("product", extractProduct)
  .addEdge(START, "analysis")
  .addEdge("analysis", "product")
  .addEdge("product", END);

const graph2 = builder2.compile();
const result2 = await graph2.invoke({ input: "텍스트에서 감성과 제품 정보를 동시에 뽑아줘" });
console.log(result2.analysis, result2.product);
```

### 테스트 및 검증 전략
- 스키마는 좁게 정의하고 enum을 적극 활용해 허용값을 제한한다.
- 통합 테스트에서 모델 변동성에 대비해 입력 샘플을 고정하고 스냅샷보다는 스키마 충족 여부를 검증한다.
- 실패 시 재시도 정책을 두되, 동일 입력으로 2~3회 내 재시도로 제한한다.

### 모델 및 제공자 관련 메모
- Gemini 2.5 flash는 responseSchema 기반 구조화 출력을 지원한다. withStructuredOutput은 내부적으로 Gemini의 responseSchema를 활용해 스키마를 강제한다.
- 모델이 구조를 지키기 어렵다면 스키마를 단순화하거나 필수 필드를 줄이고 후처리로 보강한다.

### 자주 겪는 문제와 해결책
- 모델이 추가 텍스트를 섞어서 JSON 파싱이 실패하는 경우: withStructuredOutput을 사용하고, 프롬프트에서 JSON 외 텍스트 출력을 금지하도록 설명을 강화한다.
- enum 값이 어긋나는 경우: 스키마의 enum을 메시지에 노출해 반드시 이 값만 사용하도록 지시한다.
- 긴 문서 처리 시 토큰 초과: 입력을 전처리해 요약본을 먼저 만든 뒤 구조화 추출을 2단계로 수행한다.

### 참고 구조
- zod로 스키마를 정의하고, LLM 인스턴스에 withStructuredOutput 적용
- LangGraph 노드에서 invoke 호출 후 반환 키만 상태에 병합
- 필요한 경우 여러 노드에서 다른 스키마를 채워 최종 상태 완성


