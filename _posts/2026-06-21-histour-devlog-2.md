---
title: "태조 이성계 AI 시나리오 구현"
date: 2026-06-21 12:00:00 +0900
categories: [Project, Capstone]
tags: [histour, ai, openai, prompt-engineering, chatbot, story]
---

> Histour 개발기 (2)
>
> 📖 이전 글
>
> [스토리로 이해하는 역사 관광 플랫폼, HISTOUR](/posts/histour-devlog/)
>
> 이번 글에서는 태조 이성계 AI 시나리오를 구현하면서 고민했던 프롬프트 설계, 스토리 진행 방식, 선택지 생성 구조를 정리한다.

## MVP에서는 하나의 시나리오에 집중했다

Histour의 최종 목표는 외국인 관광객이 한국의 역사 관광지를 더 깊이 이해할 수 있도록 돕는 것이다.

관광지에 방문하더라도 역사적 배경이나 인물, 사건을 충분히 이해하지 못한 채 지나가는 경우가 많다.

그래서 Histour는 단순한 관광 정보 제공이 아니라, 역사 스토리와 AI를 활용해 관광지의 맥락을 자연스럽게 이해할 수 있도록 하는 것을 목표로 했다.

하지만 MVP 단계에서는 모든 관광지와 역사 인물을 구현하기보다, AI 기반 스토리 기능이 실제로 자연스럽게 동작하는지를 검증하는 데 집중했다.

그래서 경복궁과 태조 이성계를 중심으로 하나의 완성된 시나리오를 먼저 구현하였다.

---

## 역사 인물 챗봇이 아닌 역사 체험을 만들고 싶었다

처음부터 단순한 질의응답 챗봇을 만들고 싶었던 것은 아니었다.

일반적인 AI 챗봇은 사용자가 질문하면 AI가 답변하는 구조다.

```text
사용자 질문
↓
AI 답변
```

하지만 Histour에서 만들고 싶었던 것은 사용자가 역사적 상황 안에 들어가 인물과 대화하고, 선택을 통해 이야기를 진행하는 구조였다.

```text
사용자
↓
태조 이성계와 대화
↓
역사적 선택
↓
스토리 진행
↓
결말 변화
```

따라서 AI는 단순히 역사 정보를 설명하는 역할이 아니라, **역사 인물의 말투와 상황을 유지하면서 스토리를 진행시키는 역할**을 해야 했다.

---

## 역사 자료를 기반으로 스토리 단계를 나누었다

태조 이성계의 생애를 하나의 긴 대화로만 처리하면 AI가 현재 상황을 놓치기 쉽다고 생각했다.

예를 들어 아직 고려 말 시점인데 이미 조선의 왕처럼 말하거나, 왕위 계승 문제가 나오기 전인데 방원과 방석 이야기를 너무 빨리 꺼낼 수 있다.

그래서 태조 이성계의 생애와 조선 건국 흐름을 바탕으로 스토리를 단계별로 나누었다.

단계 구성에는 한국민족문화대백과사전 등 역사 자료를 참고했다.

```ts
const TAEJO_STEPS = [
  {
    title: "고려 말의 혼란",
    background:
      "고려는 오래된 나라였지만, 힘이 약해졌습니다. 백성들은 전쟁과 세금 때문에 힘들어했습니다. 이성계는 나라를 바꾸는 일이 정말 옳은지 고민했습니다.",
  },
  {
    title: "위화도 회군",
    background:
      "이성계는 북쪽으로 군대를 보내라는 명령을 받았습니다. 하지만 군사들은 지쳤고, 더 가면 많은 사람이 죽을 수 있었습니다. 이 선택은 훗날 새 나라를 여는 큰 시작이 됩니다.",
  },
  {
    title: "조선 건국",
    background:
      "이성계는 결국 새 나라 조선을 세웠습니다. 하지만 새 나라를 세우는 일에는 많은 사람의 도움과 희생이 있었습니다. 그의 아들 방원도 조선을 세우는 데 큰 역할을 했습니다.",
  },
  {
    title: "새 나라의 중심 정하기",
    background:
      "조선은 새로운 중심 도시가 필요했습니다. 한양은 나라를 다스리기 좋은 곳으로 여겨졌습니다. 하지만 나라를 안정시키려면 도시뿐 아니라 다음 왕 문제도 준비해야 했습니다.",
  },
  {
    title: "다음 왕 문제",
    background:
      "태조에게는 여러 아들이 있었습니다. 방원은 조선을 세우는 데 큰 역할을 한 아들이었습니다. 방석은 태조가 아끼던 어린 아들이었습니다. 태조는 나라를 안정시킬 사람과 마음이 가는 사람 사이에서 고민했습니다.",
  },
];
```

이렇게 단계를 나누면 AI에게 현재 진행 중인 역사 단계와 배경을 함께 전달할 수 있다.

```ts
const currentStep =
  TAEJO_STEPS[Math.min(progress ?? 0, TAEJO_STEPS.length - 1)];
```

실제 요청에서는 현재 단계 정보를 프롬프트에 포함했다.

```ts
content: `
현재 언어: ${language}
현재 진행률: ${progress ?? 0}/5
현재 역사 단계: ${currentStep.title}
현재 역사 배경: ${currentStep.background}

나라 중심 점수: ${nationScore ?? 0}
감정 중심 점수: ${emotionScore ?? 0}

이전 대화:
${JSON.stringify(history ?? [])}

사용자 입력:
${message}
`
```

이를 통해 AI가 항상 현재 역사 단계 안에서 답변하도록 제한할 수 있었다.

---

## 프롬프트로 태조 이성계의 페르소나를 설계했다

초기에는 단순히 "태조 이성계처럼 답변해줘" 정도의 프롬프트만 사용해도 충분할 것이라고 생각했다.

하지만 테스트를 진행하면서 여러 문제가 발생했다.

- 고려 장군 시점인데 이미 조선의 왕처럼 말하는 경우
- 현대적인 표현을 사용하는 경우
- 사용자의 질문에 답하기보다 역사 설명만 길게 하는 경우
- 현재 단계와 무관한 사건을 먼저 언급하는 경우

예를 들어 아직 위화도 회군 이전인데도 조선 건국 이후의 이야기를 언급하거나, 왕위 계승 문제가 등장하기 전인데 방원과 방석 이야기를 먼저 꺼내는 경우가 있었다.

그래서 단순히 인물 이름만 지정하는 것이 아니라,

- 현재 역사 단계
- 인물의 성격
- 말투
- 사용 가능한 역사 정보

를 모두 프롬프트에 포함하도록 수정했다.

```ts
const TAEJO_PROMPT = `
너는 조선의 첫 번째 왕, 태조 이성계다.

사용자는 미래에서 온 여행자이며,
너와 대화하며 조선 건국의 중요한 선택에 조언한다.

성격:
- 담대하고 결단력이 있다.
- 현실적인 판단을 중시한다.
- 군사와 백성의 희생을 가볍게 여기지 않는다.
- 나라의 미래를 중요하게 생각한다.
- 가족과 다음 왕 문제에서는 감정적으로 흔들린다.
`;
```

또한 이야기 시점이 꼬이지 않도록 현재 단계별 역할도 명시했다.

```ts
const TAEJO_PROMPT = `
이 이야기는 고려 말부터 조선 건국 초기까지 이어진다.

이성계는 처음에는 고려의 장군이고,
조선을 세운 뒤에는 조선의 왕, 태조가 된다.

현재 단계에 맞게 말하라.

- 고려 말 단계에서는 이성계는 고려의 장군이다.
- 조선 건국 이후 단계에서는 이성계는 조선의 왕이다.
- 고려 말에 나오는 "왕"은 고려 왕을 뜻한다.
- 조선 건국 이후에는 "왕"이 아니라 "나 태조", "나 이성계"라고 분명히 말하라.
`;
```

말투도 따로 제어했다.

```ts
const TAEJO_PROMPT = `
language가 "ko"인 경우:
- 한국어로 답한다.
- 태조 이성계의 말투로 답한다.
- "~하오", "~이오", "~아니겠소", "~그러하오" 등을 사용한다.
- 현대적인 표현은 피한다.
- 외국인 한국어 학습자가 이해할 수 있게 쉬운 단어를 사용한다.
- 한 문장은 짧게 쓴다.
- 답변은 3~5문장 정도로 짧게 한다.
`;
```

이렇게 작성하니 AI가 단순히 역사 정보를 설명하는 것이 아니라, 태조 이성계라는 인물처럼 답변하도록 유도할 수 있었다.

---

## 다국어 응답은 번역이 아니라 생성으로 처리했다

Histour는 외국인 관광객을 대상으로 하기 때문에 영어 응답도 필요했다.

처음에는 한국어로 답변을 생성한 뒤 번역하는 방식도 생각할 수 있었다.

하지만 역사 인물과 장소 이름은 번역 과정에서 일관성이 깨질 수 있다.

예를 들어 다음과 같은 고유명사는 서비스 안에서 일관되게 유지되어야 했다.

- 태조 이성계
- 경복궁
- 고려
- 조선

그래서 AI 응답은 한국어 답변을 번역하는 방식이 아니라, 처음부터 선택한 언어로 생성하도록 설계했다.

```ts
const TAEJO_PROMPT = `
If language is "en":

- Reply entirely in English.
- Speak as King Taejo of Joseon.
- Use a dignified royal tone.
- Use "King Taejo" instead of "Taejo".
- Use "Lee Seong-gye" instead of "Yi Seong-gye".
- Use "Gyeongbokgung Palace".
- Use "Joseon" and "Goryeo".
`;
```

요청을 보낼 때도 현재 언어 값을 함께 전달했다.

```ts
const { message, progress, history, nationScore, emotionScore, language = "ko" } =
  req.body;
```

프론트엔드에서는 사용자의 언어 설정을 읽어 AI 요청에 함께 넘겼다.

```tsx
const language =
  typeof window !== "undefined"
    ? window.localStorage.getItem("histour-language") ?? "ko"
    : "ko";

const result = await askTaejo(
  userMessage,
  progress,
  nextMessages,
  nationScore,
  emotionScore,
  language
);
```

이 방식의 장점은 번역 결과를 후처리하지 않아도 된다는 점이다.

고유명사를 프롬프트에서 직접 지정했기 때문에 영어 답변에서도 표현을 일관되게 유지할 수 있었다.

---

## 자유 채팅과 선택지를 함께 사용했다

Histour의 AI 체험은 자유 채팅과 선택형 스토리를 함께 사용한다.

처음부터 선택지만 제공하면 사용자가 AI와 대화한다는 느낌이 약해진다.

반대로 자유 채팅만 제공하면 스토리가 앞으로 진행되지 않을 수 있다.

그래서 사용자가 몇 번 자유롭게 대화한 뒤, 중요한 시점에서 선택지가 등장하도록 구현했다.

```tsx
const [messages, setMessages] = useState<ChatMessage[]>([
  {
    role: "assistant",
    text: getOpeningLine(character, language),
  },
]);

const [choices, setChoices] = useState<TaejoChoice[]>([]);
const [conversationCount, setConversationCount] = useState(0);
```

사용자가 메시지를 보내면 현재 대화 기록에 사용자 메시지를 추가하고, AI에게 이전 대화와 함께 전달했다.

```tsx
const nextMessages: ChatMessage[] = [
  ...messages,
  {
    role: "user",
    text: userMessage,
  },
];

const result = await askTaejo(
  userMessage,
  progress,
  nextMessages,
  nationScore,
  emotionScore,
  language
);
```

이전 대화 `history`를 함께 보내는 이유는 사용자마다 다른 흐름을 만들기 위해서다.

같은 역사 단계라도 사용자가 어떤 질문을 했는지에 따라 AI의 답변과 선택지가 달라질 수 있다.

---

## 사용자마다 다른 선택지가 나오도록 했다

![선택지 생성 화면](/assets/img/histour/story-choice.png){: width="75%" }

일반적인 선택형 스토리 게임은 미리 작성된 선택지를 보여주는 경우가 많다.

하지만 Histour에서는 사용자가 어떤 질문을 했는지에 따라 선택지가 달라질 수 있도록 만들고 싶었다.

그래서 선택지를 미리 저장하지 않고 AI가 생성하도록 설계했다.

AI에게 선택지를 생성하도록 요청할 때는 단순히 현재 질문만 전달하지 않았다.

```text
현재 역사 단계
이전 대화(history)
nationScore
emotionScore
사용자 입력
```

이 정보를 함께 전달하여 AI가 현재 상황을 이해한 상태에서 선택지를 생성하도록 했다.

예를 들어 백성들의 삶을 먼저 걱정하는 사용자는 백성을 위한 선택지가 생성될 수 있고, 군사적 판단을 중요하게 생각하는 사용자는 질서와 안정 중심의 선택지가 생성될 수 있다.

이를 통해 모든 사용자가 동일한 선택지를 보는 것이 아니라, 자신의 대화 흐름에 따라 조금씩 다른 경험을 할 수 있도록 했다.

선택지 생성 시에는 다음과 같은 규칙을 프롬프트에 포함했다.

```ts
const TAEJO_PROMPT = `
선택지 규칙
- 중요한 결정 순간에만 선택지를 만든다.
- 선택지를 만들 때만 정확히 2개를 만든다.
- 같은 선택지를 반복하지 않는다.
- 선택지는 현재 대화와 직접 관련되어야 한다.
- 선택지는 단순 찬성/반대가 아니라 실제 행동처럼 보여야 한다.
- 각 선택지는 nation 또는 emotion type을 가진다.
`;
```

또한 응답 형식을 고정하기 위해 JSON 구조를 명시했다.

```ts
const TAEJO_PROMPT = `
형식:
{
  "reply": "태조 이성계의 대답",
  "choices": []
}

선택지가 필요한 경우:
{
  "reply": "태조 이성계의 대답",
  "choices": [
    {
      "text": "첫 번째 선택지",
      "type": "nation"
    },
    {
      "text": "두 번째 선택지",
      "type": "emotion"
    }
  ]
}
`;
```

프론트엔드에서는 사용자가 어느 정도 대화를 진행한 뒤에만 선택지가 등장하도록 했다.

```tsx
const nextConversationCount = conversationCount + 1;
setConversationCount(nextConversationCount);

const shouldShowChoices =
  nextConversationCount >= 2 ||
  (nextConversationCount >= 1 && Math.random() < 0.5);

if (shouldShowChoices) {
  setChoices(result.choices ?? []);
} else {
  setChoices([]);
}
```

이렇게 하면 사용자가 첫 메시지를 보내자마자 선택지가 등장하지 않는다.

먼저 AI와 자유롭게 대화를 나누고, 이후 중요한 시점에 선택지가 등장하도록 하여 자유 채팅과 선택형 스토리의 균형을 맞추고자 했다.

---

## 선택은 점수가 되고, 점수는 결말이 된다

사용자의 선택은 단순히 다음 대화만 바꾸는 것이 아니라, 최종 결말을 결정하는 기준이 된다.

스토리 상태는 `StoryClient`에서 관리했다.

```tsx
const [progress, setProgress] = useState(0);
const [nationScore, setNationScore] = useState(0);
const [emotionScore, setEmotionScore] = useState(0);
```

선택지에는 `nation` 또는 `emotion` 타입이 있다.

사용자가 선택지를 고르면 해당 타입에 따라 점수가 증가한다.

```tsx
const nextProgress = progress + 1;

const nextNationScore =
  choice.type === "nation" ? nationScore + 1 : nationScore;

const nextEmotionScore =
  choice.type === "emotion" ? emotionScore + 1 : emotionScore;
```

이 구조는 사용자의 선택을 두 방향으로 나누기 위한 것이다.

- `nationScore`: 나라, 백성, 책임, 안정 중심의 선택
- `emotionScore`: 가족, 개인적 감정, 인간적 관계 중심의 선택

최종 단계에 도달하면 두 점수를 비교해 결말을 결정했다.

```tsx
const nextEnding =
  nextNationScore >= nextEmotionScore ? "great" : "lonely";
```

결말은 두 가지로 구성했다.

```text
🏛️ Great Founder
나라와 백성을 먼저 생각한 결말

👑 Lonely Father
가족과 인간적인 마음을 먼저 생각한 결말
```

복잡한 스토리 엔진을 구현하는 방법도 있었지만, MVP 단계에서는 구현 복잡도를 낮추면서도 선택의 결과를 체감할 수 있는 구조가 필요했다.

그래서 사용자의 선택을

- 나라와 책임 중심
- 가족과 감정 중심

이라는 두 개의 축으로 단순화했다.

사용자는 이를 인식하지 못한 채 선택하지만, 선택이 누적되면서 최종 결말에 영향을 주게 된다.

이를 통해 비교적 단순한 구조로도 "내 선택이 결과를 바꿨다"는 경험을 제공할 수 있었다.

---

## MVP 단계에서의 한계

이번 구현은 MVP였기 때문에 우선 하나의 시나리오가 처음부터 끝까지 자연스럽게 동작하는 것을 목표로 했다.

따라서 몇 가지 단순화된 설계를 사용했다.

첫째, 현재는 `nationScore`와 `emotionScore` 두 가지 기준만으로 결말을 결정한다.

```tsx
const nextEnding =
  nextNationScore >= nextEmotionScore ? "great" : "lonely";
```

구현이 단순하다는 장점은 있지만, 사용자의 성향을 충분히 반영하기에는 한계가 있다.

예를 들어 `nationScore`가 조금 더 높은 경우와 압도적으로 높은 경우가 같은 결말로 처리된다.

향후에는 점수 차이에 따라 중간 결말을 추가하거나, 선택 이유까지 함께 분석하는 방식으로 개선해보고 싶다.

둘째, 현재는 선택지가 생성되는 시점을 대화 횟수를 기준으로 결정한다.

```tsx
const shouldShowChoices =
  nextConversationCount >= 2 ||
  (nextConversationCount >= 1 && Math.random() < 0.5);
```

이는 자유 채팅과 선택형 스토리의 균형을 맞추기 위한 MVP 단계의 구현이었다.

향후에는 사용자의 질문 내용과 현재 역사 상황을 분석하여, 실제로 중요한 순간에만 선택지가 등장하도록 개선하고 싶다.

셋째, 현재는 태조 이성계 한 명만 지원한다.

하지만 현재 구조는 역사 단계, 프롬프트, 결말 로직을 분리해 두었기 때문에 향후 세종, 정조, 광해군, 단종 등 다른 역사 인물로 확장할 수 있도록 설계했다.

---

## 마무리

이번 구현에서 가장 중요했던 것은 단순히 OpenAI API를 연결하는 것이 아니었다.

AI가 역사 인물처럼 말하도록 프롬프트를 설계하고, 현재 역사 단계와 이전 대화를 함께 전달하며, 사용자의 선택을 점수로 누적해 결말로 연결하는 구조를 만드는 것이 핵심이었다.

특히 한국민족문화대백과사전 등의 역사 자료를 기반으로 스토리 단계를 구성하고, 현재 단계 정보를 프롬프트에 함께 전달하여 AI가 시점을 벗어나지 않도록 설계한 부분이 가장 많은 고민이 들어간 부분이었다.

또한 사용자의 질문과 이전 대화 내용을 바탕으로 선택지를 생성하도록 하여, 단순한 질의응답 챗봇이 아닌 역사 체험형 서비스에 가까운 경험을 제공하고자 했다.

이번 MVP를 통해 하나의 역사 인물 체험 흐름을 구현할 수 있었고, AI를 활용한 역사 콘텐츠의 가능성도 확인할 수 있었다.

앞으로는 세종, 정조, 광해군, 단종 등 다양한 역사 인물로 확장하고, 사용자의 선택에 따라 실제 역사와는 다른 가상 시나리오를 경험할 수 있는 구조도 실험해보고 싶다.

또한 AI 생성 영상을 활용해 결말을 시각적으로 보여주는 기능까지 연결한다면, 단순한 관광 정보 제공을 넘어 역사 체험 플랫폼으로 발전할 수 있을 것이라고 생각한다.

---
캡스톤디자인 프로젝트 Histour 개발 과정 정리.