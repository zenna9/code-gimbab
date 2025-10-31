---
title: "하루 만에 만들어본 회의록 자동화 서비스"
date: 2025-10-30
---

지난 블로그에서 [**LangChain**](/tech-blog/articles/langChain-frontend)에 대해 정리했었다. 어떻게 동작하는지, 왜 필요한지 알아만 보고 연휴 전 빡빡한 배포 일정으로 만들어보지 못했다. 반성하자^^

어떤 AI 서비스를 만들어볼까를 고민하다가, 요즘 회의를 할때 기록을 해두지 않으면 종종 놓치는 부분들이 생기고 있다. 회의 중에 집중하느라 놓치는 부분도 있고 악필이라 메모를 해도 이게 뭐였지 하는 부분이 가끔 있다. **이러한 부분을 발화한 부분을 자동으로 텍스트로 바꾸고, AI가 요약해주면 좋을 것 같다는 생각이 들었다.**

예전 항해를 했을때 멘토 시간에 꼭 서기가 필요했었는데, **자동 완성을 해주는 AI가 있으면 편하지 않을까**라는 생각도 들었었다.

## **어떻게 만들어야 할까?**

간단하게 생각하면 음성을 넣으면 자동으로 회의록이 나오도록 하는 것처럼 보이지만, 생각해보면 여러 단계가 필요하다. **음성을 텍스트로 바꾸고, 누가 말했는지 구분하고, AI로 요약해서 텍스트까지 뽑아내는 것까지.**

일단 하루만에 만들어 보는 것으로 목표를 했다. 완벽하게 만들려고 하면 끝이 없을 것 같았다. DB 같은 건 나중 문제고, **일단 실시간으로 녹음하고 회의록이 나오는 것만 만들어보기로 했다.** 그리고 완성된 회의록은 .md 형식으로 내보내서 노션에 복사 붙여넣기할 수 있도록 구성했다.

기술 스택은 React + TypeScript로 프론트엔드를 만들고, 백엔드는 Node.js + Express로 구성했다.

## **먼저 Speech-to-Text (STT) 모델 고르기**

회사에서는 [**Whisper**](https://openai.com/ko-KR/index/whisper/)를 사용하여 텍스트를 추출하고 있다. 정확도 면에서 괜찮은데, 회의록 서비스로 사용하기엔 몇가지 문제가 있다.

첫번째 문제는 화자 분리다. Whisper는 "누가 말했는지" 구분을 못한다. 회의록에서 제일 중요한 게 누가 뭐라고 했는지인데 말이다. 화자 분리를 하려면 별도의 라이브러리를 붙여야 하고, 두 개의 결과를 타임스탬프로 매칭하는 과정도 필요하다.

두번째 문제는 자동 종료이다. 회사에서 실제 서비스를 운영하다 보면 사용자들이 녹음 중지 버튼을 누르는 걸 자주 잊어버리는 경우가 존재한다. 만약 회의가 끝났는데도 계속 녹음되고 있거나, 퇴근 후에도 켜져있는 상황이라면, 자체 모델을 사용하고 있지 않는 한 많은 비용 지출이 일어나게 된다. 회사에서는 이부분을 단순 소리 임계값으로 해결하려 했지만, 큰 이슈가 존재했다. 외부의 큰 소음이 들어가면 중지가 되지 않는 이슈도 존재했다.

요즘 고도화를 위해 STT 모델을 많이 찾아보고 있는데, 요즘 눈여겨 봤던 [**Soniox**](https://soniox.com/)라는 STT를 이번 개발팀 회의때 STT 모델 마이그레이션을 제안했다. 화자 분리를 기본으로 내장하고 있고, 음성 기반 토큰 별로 끝을 알 수 있어, 자동 종료 기능도 소리 임계값이 아닌 화자의 음성으로만 판단하여 종료할 수 있게 된다.

## 프로젝트 구현

Soniox는 WebSocket 기반 실시간 스트리밍을 지원한다. 브라우저에서 직접 마이크로 녹음하고, 실시간으로 텍스트를 받을 수 있다. 가장 큰 특징은 각 토큰에 `is_final` 속성을 제공한다는 점이다. Final 토큰이 나오면 그 구간이 확정된 텍스트이고, 아니면 아직 변경될 수 있는 부분 텍스트이다.

<p align="center">
  <img src="/tech-blog/assets/images/soniox1.png" alt="react-virtualized vs window"  />
</p>

이 특징을 활용해서 Final 토큰이 3분간 없으면 자동으로 녹음을 중지하도록 구현했다. 화자가 말을 멈추면 Final 토큰이 나오지 않기 때문에, 3분간 조용하면 회의가 끝났다고 판단하는 것이다. 이렇게 하면 소음 임계값과 달리 외부 소음이 있어도 화자가 말하지 않으면 종료되므로 훨씬 정확하다. 하지만 필요할 때 내가 직접 컨트롤할 수 있도록 수동 중지 버튼도 함께 제공했다.

실시간으로 텍스트가 화면에 나타나도록 구현했다. Soniox는 Partial 토큰을 지속적으로 업데이트하기 때문에, 사용자가 말하는 동안 실시간으로 화면에 텍스트가 표시된다. 말을 멈추면 Final 토큰으로 확정되어서 파란색에서 검은색 텍스트로 바뀌는 식으로 시각적인 피드백도 제공할 수 있다.

Soniox의 `enableSpeakerDiarization: true` 옵션을 활성화하면 각 토큰에 speaker 필드가 포함된다. 콘솔을 보면 화자 정보가 실시간으로 출력되는 것을 확인할 수 있다. 예를 들어 "Speaker 1", "Speaker 2" 같은 식으로 화자가 구분되어 나온다. 이렇게 화자 분리가 자동으로 이루어지기 때문에 회의록에서 누가 무엇을 말했는지 추적하기 쉬워진다.

<p align="center">
  <img src="/tech-blog/assets/images/soniox2.png" alt="react-virtualized vs window"  />
</p>
<p align="center">
  <img src="/tech-blog/assets/images/soniox3.png" alt="react-virtualized vs window"  />
</p>

STT 결과를 받았으니 이제 AI로 회의록을 만들어야 한다. LangChain을 사용해서 OpenAI GPT-4o-mini로 회의록을 생성하도록 구현했다.

LangChain을 쓰지 않았다면 직접 OpenAI API를 호출해야 했을 것이다. fetch로 HTTP 요청을 보내고, 헤더에 API 키를 넣고, 응답을 파싱하는 번거로운 과정을 거쳐야 한다. 하지만 LangChain을 사용하면 ChatOpenAI 인스턴스를 만들고 invoke 메서드만 호출하면 된다. 코드가 훨씬 간결해지고, 프롬프트 관리도 쉽다. 나중에 체이닝을 추가하거나 다른 LLM 제공자로 전환할 때도 한 줄만 바꾸면 되기 때문에 유지보수가 훨씬 편하다.

실제 구현 코드를 보면 Soniox에서 받은 Final/Partial 토큰을 구분해주고 있다. Partial 토큰은 실시간으로 화면에 표시하고, Final 토큰이 나오면 확정된 텍스트로 누적한다. 이렇게 하면 사용자가 말하는 동안 계속 업데이트되다가, 말을 멈추는 순간 확정된다. Soniox의 `is_final` 속성을 활용하면 복잡한 로직 없이도 쉽게 구분할 수 있다.

```typescript
onPartialResult: (result) => {
  const tokens = result.tokens;
  const finalTokens = tokens.filter((t) => t.is_final);
  const partialTokens = tokens.filter((t) => !t.is_final);

  // Final 토큰은 누적, Partial 토큰은 실시간 표시
  if (finalTokens.length > 0) {
    accumulatedText.push(finalTokens.map((t) => t.text).join(""));
  }
  displayLiveTranscription(partialTokens);
};
```

회의록 형식을 구성할 때 초기 단계이니 간단하게 만들었다. 회의 날짜, 회의 개요, 주요 논의 사항만 포함하도록 설정했다. 나중에 필요하면 결정 사항, 액션 아이템 등을 추가할 수 있지만, 우선은 핵심 내용만 정리하도록 했다. 이렇게 간단한 구조로 시작하면 프롬프트도 단순해지고, LLM이 생성하는 회의록의 품질도 더 안정적이다.

```javascript
import { ConversationBufferMemory } from "langchain/memory";

const memory = new ConversationBufferMemory();
memory.saveContext({ input: "이전 대화 내용" }, { output: "회의록의 일부" });

// 이후 회의록 생성 시 메모리 활용
const messages = [
  new SystemMessage(systemPrompt),
  ...(await memory.chatHistory.getMessages()),
  new HumanMessage("새로운 대화 내용"),
];
```

이렇게 구현하면 "앞에서 논의한 프로젝트 A를 다시 정리해줘" 같은 요청도 메모리에 저장된 이전 대화를 활용해서 답변할 수 있다. LangChain이 없었다면 이런 메모리 관리를 직접 구현해야 했을 것이다.

## Memory 적용으로 얻은 실제 효과

ConversationBufferMemory를 적용하면서 체감한 가장 큰 변화는 토큰 절감이었다. 기존 방식은 회의가 진행될수록 전체 대화가 누적되어 토큰이 선형적으로 증가했다. 예를 들어 30분짜리 회의의 경우 입력 토큰만 7,000~8,000개를 넘기는 경우도 있었다.

하지만 ConversationSummaryMemory를 사용하면 LangChain이 이전 대화를 요약한 형태로 전달하기 때문에, 같은 길이의 회의에서도 약 60~70%의 토큰 절감 효과를 얻을 수 있었다. 이는 곧 OpenAI API 비용 절감과 요청 속도 개선으로 이어졌다.

| 항목              | Memory 미적용   | Memory 적용 후      |
| ----------------- | --------------- | ------------------- |
| 문맥 유지         | 배열 수동 관리  | LangChain 자동 요약 |
| 회의 재개 시 맥락 | 직접 전달 필요  | 자동으로 이어짐     |
| 평균 토큰 사용량  | 약 7,000 tokens | 약 2,500 tokens     |
| 평균 응답 시간    | 3.2초           | 1.4초               |

**실제 비용 계산**: 전체 회의록 생성 프로세스 비용을 계산해보면 다음과 같다. Soniox는 실시간 1시간당 약 $0.12 (audio 1M tokens당 $2.00), OpenAI GPT-4o-mini는 입력 1M 토큰당 $0.150, 출력 1M 토큰당 $0.600이다.

Memory를 적용하면 회의록 생성 비용만 64% 절감된다. (참고: [Soniox Pricing](https://soniox.com/pricing), [OpenAI Pricing](https://openai.com/ko-KR/api/pricing/))

요즘 STT, LLM 모델들이 정말 많이 나오고 있다. Whisper, [Soniox](https://soniox.com/pricing) (실시간 $2.00/1M audio tokens), AssemblyAI, Claude, GPT-4, Llama 등 끝도 없다. 이런 상황에서 개발자의 중요한 역량 중 하나는 **어떤 모델을 잘 선택하는 것**이다. 단순히 무료라고 선택하거나, 가장 정확하다고 선택하는 것이 아니라 프로젝트의 요구사항에 맞게 비용과 성능의 균형을 맞추는 것이다.

이번 프로젝트로 STT와 LLM 모델을 선택하는 기준을 명확히 정리할 수 있었다. 회사 동료들과 공유하기 좋은 자료가 될 것 같다.

## 배운 점

LangChain의 가치는 단순히 LLM을 호출하는 것을 넘어선다. 체이닝으로 여러 단계를 나눠서 처리할 수 있고, RAG처럼 검색해서 답변하는 흐름도 구현할 수 있다. 프롬프트 템플릿으로 재사용 가능한 프롬프트를 관리할 수도 있고, 메모리로 대화 히스토리를 관리할 수도 있다. 이번 프로젝트에서는 단순하게만 사용했지만, 나중에 고도화할 때 이런 기능들을 활용하면 훨씬 강력한 서비스를 만들 수 있을 것 같다.
