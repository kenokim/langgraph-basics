## LangGraph (TypeScript) 기본기 체크리스트

### 핵심 개념
- 상태 모델: Annotation으로 그래프 전역 상태의 키와 타입을 정의한다. 각 노드는 Partial 상태를 반환하고 그래프가 병합한다.
- 노드: 비동기 함수로 작성하며 입력 상태를 읽고 필요한 키만 갱신한다. 가능한 순수 함수로 유지한다.
- 엣지: 노드 간 실행 순서를 정의한다. 시작/종료를 위해 START/END 심볼을 사용한다.
- 실행: compile된 그래프를 invoke로 호출하면 최종 상태가 반환된다.

### 최소 예시
```ts
import { Annotation, StateGraph, START, END } from "@langchain/langgraph";

const State = Annotation.Root({
  input: Annotation<string>(),
  output: Annotation<string | undefined>(),
});

async function step(state: typeof State.State) {
  return { output: `echo: ${state.input}` } as Partial<typeof State.State>;
}

const builder = new StateGraph(State)
  .addNode("step", step)
  .addEdge(START, "step")
  .addEdge("step", END);

const graph = builder.compile();
const res = await graph.invoke({ input: "hello" });
```

### 상태 설계 원칙
- 상태 키는 작게, 명확하게 나눈다. 한 노드는 소수의 키만 책임진다.
- 병합 시 마지막 작성자가 승리한다. 누적이 필요한 값은 배열/맵에 push/merge 전략을 분리한다.
- zod 등으로 구조를 고정하고, LLM 결과는 구조화하여 저장한다.

### 노드 설계 원칙
- 외부 호출(LLM/HTTP)은 타임아웃과 재시도를 둔다.
- 입력 검증을 먼저 하고, 실패는 조기에 반환한다.
- 가능한 한 사이드이펙트를 줄이고, 로깅/메트릭만 남긴다.

### 흐름 제어 패턴
- 직렬 흐름: addEdge로 순차 실행을 표현한다.
- 분기: 라우팅 노드에서 다음 노드 이름을 결정해 조건부로 이어간다.
- 병렬: 여러 하위 호출이 필요하면 단일 노드 내부에서 Promise.all로 병렬 처리하고 결과를 하나의 Partial 상태로 합친다.

### LLM 통합 패턴
- 일반 호출: LLM 클라이언트를 노드 내부에서 호출하고 텍스트를 상태에 저장한다.
- 구조화 출력: zod 스키마를 정의하고 withStructuredOutput을 사용해 안전하게 파싱된 객체만 상태에 저장한다.
```ts
import { z } from "zod";
import { ChatGoogleGenerativeAI } from "@langchain/google-genai";

const SummarySchema = z.object({ title: z.string(), bullets: z.array(z.string()) });
const base = new ChatGoogleGenerativeAI({ model: "gemini-1.5-pro-002", temperature: 0 });
const summaryModel = base.withStructuredOutput(SummarySchema);

async function summarizeNode(state: typeof State.State) {
  const s = await summaryModel.invoke(state.input);
  return { output: s.title };
}
```

### 에러 처리와 복구
- 노드 경계에서 try/catch로 에러를 캡처하고 상태에 에러 정보를 기록한다.
- 재시도는 지수 백오프로 제한한다. 반복 실패 시 폴백 노드로 라우팅한다.
- 구조화 출력 파싱 실패는 입력 축소나 스키마 단순화로 회피한다.

### 테스트 전략
- 단위: 각 노드를 독립적으로 테스트하고, 모의 LLM을 주입한다.
- 통합: 대표 입력에 대해 그래프 invoke 결과의 상태 형태와 주요 필드를 검증한다.
- 회귀: 스냅샷보다는 스키마 유효성 검사를 사용한다.

### 운영 체크리스트
- 로깅: 노드 시작/종료, 외부 호출 지표, 오류 사유를 구조화 로깅으로 남긴다.
- 안정성: 타임아웃, 재시도, 회로차단기 같은 보호 장치를 둔다.
- 성능: 토큰/레이턴시 지표를 수집하고, 병렬화 가능한 작업을 식별해 통합한다.
- 변경관리: 상태 스키마 변경 시 마이그레이션 계획을 준비한다.


