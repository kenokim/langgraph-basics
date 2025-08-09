## LangGraph (TypeScript)에서 State 이해하기

### 개요
LangGraph의 state는 그래프 실행 중 노드들이 읽고 쓰는 공유 데이터 컨테이너다. TypeScript에서는 Annotation을 사용해 상태의 키와 타입을 선언하고, 각 노드는 Partial 상태를 반환해 그래프가 병합한다. 상태는 그래프의 단일 진실 원천으로, 노드 간 결합도를 낮추고 흐름 제어를 단순화한다.

### 상태 정의
```ts
import { Annotation } from "@langchain/langgraph";
import { BaseMessage } from "@langchain/core/messages";

// messages 누적 리듀서와 기본값
function messagesStateReducer(
  previous: BaseMessage[] | undefined,
  update: BaseMessage | BaseMessage[] | undefined
): BaseMessage[] {
  const prev = previous ?? [];
  if (!update) return prev;
  const next = Array.isArray(update) ? update : [update];
  return [...prev, ...next];
}

// 전역 상태 스키마 정의
export const GraphState = Annotation.Root({
  input: Annotation<string>(),
  summary: Annotation<string | undefined>(),
  analysis: Annotation<{ sentiment: "positive" | "neutral" | "negative"; reasons: string[] } | undefined>(),
  error: Annotation<string | undefined>(),
  messages: Annotation<BaseMessage[]>({
    reducer: messagesStateReducer,
    default: () => [],
  }),
});

// GraphState.State는 타입 안전한 상태 형태를 제공한다
type State = typeof GraphState.State;
```

### 병합 규칙(머지 동작)
- 노드는 Partial<State>를 반환하며, 반환된 키만 상태에 반영된다.
- 기본적으로 동일 키에 대해 마지막 작성자가 값을 덮어쓴다.
- 자동 누적은 없다. 배열 누적, 맵 병합 등은 노드 내부에서 명시적으로 구현해야 한다.
- 충돌을 줄이려면 각 노드는 자신이 책임지는 키 집합만 갱신한다.

### 직렬 흐름 예제
```ts
import { StateGraph, START, END } from "@langchain/langgraph";
import { GraphState } from "./state";

async function summarize(state: State) {
  const text = state.input;
  const s = text.length > 50 ? `${text.slice(0, 47)}...` : text;
  return { summary: s } as Partial<State>;
}

async function analyze(state: State) {
  const s = state.summary ?? state.input;
  const sentiment = /좋|만족|추천/.test(s) ? "positive" : /불만|나쁨/.test(s) ? "negative" : "neutral";
  return { analysis: { sentiment, reasons: ["간단 규칙 기반 추정"] } } as Partial<State>;
}

const builder = new StateGraph(GraphState)
  .addNode("summarize", summarize)
  .addNode("analyze", analyze)
  .addEdge(START, "summarize")
  .addEdge("summarize", "analyze")
  .addEdge("analyze", END);

export const graph = builder.compile();

// 실행
const out = await graph.invoke({ input: "이 제품은 가성비가 좋고 디자인이 마음에 듭니다." });
// out.summary, out.analysis 사용
```

### 분기와 라우팅 예제
```ts
import { StateGraph, START, END } from "@langchain/langgraph";
import { GraphState } from "./state";

async function route(state: State) {
  // 다음 노드 이름을 반환하여 조건부 분기
  const next = state.input.length > 100 ? "summarize" : "analyze";
  return { __route__: next } as unknown as Partial<State>;
}

async function summarize(state: State) { /* ... */ return { summary: state.input.slice(0, 50) } }
async function analyze(state: State) { /* ... */ return { analysis: { sentiment: "neutral", reasons: [] } } }

const builder = new StateGraph(GraphState)
  .addNode("route", route)
  .addNode("summarize", summarize)
  .addNode("analyze", analyze)
  .addEdge(START, "route")
  .addConditionalEdges("route", (state: State) => (state.input.length > 100 ? "summarize" : "analyze"))
  .addEdge("summarize", END)
  .addEdge("analyze", END);

const graph = builder.compile();
```

### 메시지(messages) 상태와 대화 누적
- 메시지는 대화형 LLM 호출의 핵심 상태다. `@langchain/core/messages`의 타입을 사용해 안전하게 누적한다.
- 리듀서와 기본값을 상태 정의에 넣으면, 노드는 메시지 객체만 반환해도 자동으로 누적된다.
```ts
import { Annotation, StateGraph, START, END } from "@langchain/langgraph";
import { BaseMessage, HumanMessage, AIMessage, SystemMessage } from "@langchain/core/messages";
import { ChatGoogleGenerativeAI } from "@langchain/google-genai";

function messagesStateReducer(
  previous: BaseMessage[] | undefined,
  update: BaseMessage | BaseMessage[] | undefined
): BaseMessage[] {
  const prev = previous ?? [];
  if (!update) return prev;
  const next = Array.isArray(update) ? update : [update];
  return [...prev, ...next];
}

const MsgState = Annotation.Root({
  input: Annotation<string>(),
  messages: Annotation<BaseMessage[]>({
    reducer: messagesStateReducer,
    default: () => [],
  }),
});
type Msg = typeof MsgState.State;

const model = new ChatGoogleGenerativeAI({ model: "gemini-2.5-flash", temperature: 0 });

async function systemNode(_: Msg) {
  return { messages: new SystemMessage("너는 간결하고 사실 중심으로 답변한다.") } as Partial<Msg>;
}

async function userNode(state: Msg) {
  return { messages: new HumanMessage(state.input) } as Partial<Msg>;
}

async function callLlmNode(state: Msg) {
  const ai = await model.invoke(state.messages);
  return { messages: ai } as Partial<Msg>;
}

const b = new StateGraph(MsgState)
  .addNode("system", systemNode)
  .addNode("user", userNode)
  .addNode("llm", callLlmNode)
  .addEdge(START, "system")
  .addEdge("system", "user")
  .addEdge("user", "llm")
  .addEdge("llm", END);

const chatGraph = b.compile();
const outChat = await chatGraph.invoke({ input: "제품 A의 배터리 수명은?" });
// outChat.messages: [SystemMessage, HumanMessage, AIMessage]
```

메시지 병합 팁
- 리듀서를 사용하면 노드에서는 메시지 객체만 반환한다. 리듀서를 쓰지 않는 경우에는 직접 배열 누적을 구현한다.
- 시스템 지침은 최초 1회만 추가하거나, 중간 요약 노드로 압축해 토큰을 절약한다.
- 멀티턴 입력은 외부에서 `messages`에 HumanMessage를 추가한 뒤 그래프를 재호출한다.

구조화 출력과 메시지 함께 쓰기
```ts
import { z } from "zod";
const ExtractSchema = z.object({ entities: z.array(z.string()).max(5) });
const structured = model.withStructuredOutput(ExtractSchema);

async function extractNode(state: Msg) {
  const result = await structured.invoke(state.messages);
  // 필요 시 결과 요약을 AIMessage로 기록 (리듀서가 자동 누적)
  return { messages: new AIMessage(`추출 엔티티: ${result.entities.join(", ")}`) } as Partial<Msg>;
}
```

리듀서 주의점
- 리듀서는 순수 함수여야 하며 외부 상태에 의존하지 않도록 한다.
- 대용량 배열 누적 시 별도 프루닝 전략을 병행해 메모리/토큰을 관리한다.
- 교체가 필요한 경우 전용 노드에서 별도 키에 저장하거나, 리듀서 로직을 교체 모드로 확장한다.

### 상태 핸들링 베스트 프랙티스
- 키 명세는 작게, 역할 단위로 분리한다. 한 노드는 자신의 책임 키만 수정한다.
- 누적이 필요하면 기존 값을 읽어 새로운 값을 만들어 반환한다. 암묵적 병합에 의존하지 않는다.
- 오류를 상태의 전용 필드(error 등)로 남기고, 후속 노드가 이를 감지해 폴백 분기를 수행하게 한다.
- 외부 리소스 핸들(파일 경로, ID 등)만 상태에 저장하고, 대용량 데이터는 외부 스토리지를 사용한다.
- 구조화된 데이터는 zod로 검증된 형태로 저장해 다운스트림 안정성을 높인다.

### LLM과 함께 쓰기(선택)
structured output을 사용해 LLM 결과를 안전하게 상태에 저장할 수 있다.
```ts
import { z } from "zod";
import { ChatGoogleGenerativeAI } from "@langchain/google-genai";
import { Annotation, StateGraph, START, END } from "@langchain/langgraph";

const SummarySchema = z.object({ title: z.string(), bullets: z.array(z.string()) });
const llm = new ChatGoogleGenerativeAI({ model: "gemini-2.5-flash", temperature: 0 })
  .withStructuredOutput(SummarySchema);

const LlmState = Annotation.Root({
  input: Annotation<string>(),
  summary: Annotation<z.infer<typeof SummarySchema> | undefined>(),
});

async function summarizeNode(state: typeof LlmState.State) {
  const s = await llm.invoke(state.input);
  return { summary: s };
}

const builder = new StateGraph(LlmState)
  .addNode("summarize", summarizeNode)
  .addEdge(START, "summarize")
  .addEdge("summarize", END);

const graph = builder.compile();
```

### 테스트와 검증
- 각 노드를 독립적으로 테스트하고, 상태 입력/출력의 타입을 엄격히 검증한다.
- 통합 테스트에서는 대표 입력에 대해 최종 상태의 필수 키가 채워지는지 확인한다.
- 상태 스키마 변경 시 호환성(기존 키 유지, 마이그레이션 전략)을 고려한다.


