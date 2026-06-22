---
title: "스토리로 이해하는 역사 관광 플랫폼, HISTOUR"
date: 2026-06-20 12:00:00 +0900
categories: [Project, Capstone]
tags: [histour, nextjs, tourism, i18n, capstone]
---

# 스토리로 이해하는 역사 관광 플랫폼, HISTOUR

> Histour 개발기 (1)

## 프로젝트 소개

2026년 캡스톤 디자인 프로젝트로 **Histour**라는 웹 서비스를 개발했다.

Histour는 `History + Tour`의 의미를 담은 프로젝트로, 사용자가 한국 역사 관광지를 탐색하고 해당 장소와 연결된 역사 스토리를 체험할 수 있는 플랫폼이다.

이번 글에서는 Histour를 왜 만들게 되었는지, 외국인 관광객을 위해 어떤 방식으로 서비스를 설계했는지, 그리고 MVP 단계에서 어떤 기능을 중심으로 구현했는지 정리해보려고 한다.

![Histour 메인 화면](/assets/img/histour/home-ko.png)

---

## 문제 정의

한국에는 수많은 문화유산과 역사 관광지가 존재한다.

하지만 실제 관광 경험은 대부분 관광지를 방문하고 사진을 찍는 것에 머무르는 경우가 많다. 관광지 안내판이나 설명 자료를 통해 역사 정보를 확인할 수 있지만, 많은 관광객들은 이를 자세히 읽지 않거나 단편적인 정보만 접하게 된다.

특히 외국인 관광객은 언어와 문화적 배경의 차이로 인해 역사적 맥락을 이해하기 더욱 어렵다.

예를 들어 경복궁을 방문한다고 했을 때,
- 왜 경복궁이 조선의 첫 궁궐인지
- 왜 태조 이성계가 한양 천도를 결정했는지
- 조선은 어떤 과정을 거쳐 건국되었는지
를 자연스럽게 이해하기는 쉽지 않다.

그래서 우리 팀은 다음과 같은 질문에서 프로젝트를 시작했다.

> 관광지를 단순히 보는 것이 아니라, 역사 속 이야기로 이해하게 만들 수는 없을까?

---

## 외국인 관광객을 위한 서비스 설계

Histour는 외국인 관광객을 주요 사용자로 설정했다.

문화빅데이터 플랫폼의 관광지, 역사 인물, 추천 관광지, 외국인 이용 데이터를 활용해 서비스를 기획했다.

외국인 관광객은 대부분 공항을 통해 입국한다.

따라서 사용자의 실제 여행 시작점을 고려해 공항 기준으로 관광지를 탐색할 수 있도록 설계했다.

![공항 필터](/assets/img/histour/airport-filter.png)

현재 서비스에서는 다음과 같은 공항 기준 필터를 제공한다.

- 인천공항
- 김포공항
- 김해공항
- 제주공항

공항 선택 상태는 URL Query Parameter를 이용해 관리했다.

```tsx
const airport = params.get("airport") ?? "all";
const activeCategory = params.get("category") ?? "all";

const updateParams = (updates: Record<string, string | null>) => {
  const search = new URLSearchParams(params.toString());

  Object.entries(updates).forEach(([key, value]) => {
    if (!value || value === "all") {
      search.delete(key);
    } else {
      search.set(key, value);
    }
  });

  search.set("lang", lang);
  router.push(`/?${search.toString()}`);
};
```

이 방식을 사용하면 사용자가 선택한 필터 상태를 URL에 유지할 수 있다.

새로고침하거나 링크를 공유해도 동일한 조건의 관광지 목록을 다시 확인할 수 있다는 장점이 있다.

---

## 다국어 지원

Histour는 외국인 관광객을 주요 사용자로 설정했기 때문에 다국어 기능을 함께 구현했다.

![언어 설정 화면](/assets/img/histour/language-setting.png)

화면에 고정적으로 보이는 UI 문구는 한국어와 영어 데이터를 별도로 관리했다.

예를 들어 메인 화면의 제목, 설명, 버튼 문구는 다음과 같이 구성했다.

```tsx
const heroCopy = {
  ko: {
    eyebrow: "HISTOUR",
    title: "이야기 속으로 떠나는 역사 여행",
    description:
      "AI 역사 인물과 대화하고, 선택에 따라 달라지는 스토리를 따라 한국의 장소를 여행처럼 경험하세요.",
    browse: "장소 둘러보기",
    chat: "AI 인물과 대화 시작",
  },
  en: {
    eyebrow: "HISTOUR",
    title: "Start a Story-Led History Trip",
    description:
      "Talk with historical figures and explore Korean places through interactive story choices.",
    browse: "Browse Places",
    chat: "Start an AI Story",
  },
};

const t = lang === "en" ? heroCopy.en : heroCopy.ko;
```

이 방식은 정적인 UI 문구를 효율적으로 관리할 수 있고, 번역 API를 매번 호출하지 않아도 된다.

또한 문구가 매번 달라지지 않기 때문에 서비스 전체의 표현을 일관되게 유지할 수 있다.

![영문 메인 화면](/assets/img/histour/home-en.png)

반면 AI 역사 인물 응답은 사용자의 입력에 따라 매번 달라진다.

따라서 채팅 응답은 화면 UI처럼 직접 문구를 하나하나 넣는 방식이 아니라, 사용자가 선택한 언어 값을 AI 요청에 함께 전달해 해당 언어로 답변하도록 설계했다.

AI 응답 생성 방식은 다음 글에서 더 자세히 다룰 예정이다.

---

## 관광지 상세 페이지

관광지 상세 페이지에서는 단순히 장소 이름과 이미지만 보여주는 것이 아니라, 사용자가 관광지의 역사적 맥락을 이해할 수 있도록 관련 정보를 함께 제공했다.

Histour에서는 문화빅데이터 플랫폼에서 제공하는 관광지 데이터를 활용하였다.

관광지 이름과 위치 정보뿐 아니라 시대적 배경, 주요 키워드, 관련 관광지 정보 등을 함께 활용하여 사용자가 장소에 대한 맥락을 이해할 수 있도록 구성했다.

![경복궁 상세 페이지](/assets/img/histour/place-detail.png)

경복궁 상세 페이지에서는 다음과 같은 정보를 확인할 수 있다.

- 시대
- 위치
- 주요 키워드
- 대표 이야기
- 함께 둘러보기 좋은 장소
- 이 장소와 연결된 역사 인물

상세 페이지는 `placeId`를 기준으로 데이터를 조회하도록 구성했다.

```tsx
export function generateStaticParams() {
  const places = getAllPlaces();
  return places.map((place) => ({ placeId: place.id }));
}

export default async function PlacePage({
  params,
  searchParams,
}: {
  params: Promise<{ placeId: string }>;
  searchParams: Promise<{ lang?: string }>;
}) {
  const { placeId } = await params;
  const query = (await searchParams) ?? {};
  const lang = query.lang ?? "ko";

  const place = getPlace(placeId);

  if (!place) notFound();

  return (
    <section className="page-section place-detail-section">
      <PlaceHero place={place} lang={lang} />
      <ExperienceSection place={place} lang={lang} />
    </section>
  );
}
```

이 구조를 사용하면 `/places/gyeongbokgung`처럼 장소별 상세 페이지를 만들 수 있다.

또한 `lang` 값을 함께 받아 한국어와 영어 화면을 구분해서 보여줄 수 있도록 했다.

---

## MVP에서는 왜 경복궁과 태조 이성계를 선택했을까?

초기 기획 단계에서는 다양한 역사 인물을 후보로 고려했다.

예를 들어 정조처럼 인간적인 일화가 많고 대중적으로 친숙한 인물을 선택하거나, 임진왜란과 같은 전쟁 시기를 배경으로 한 체험형 콘텐츠를 만드는 방안도 고민했다.

하지만 MVP 단계에서는 단순히 유명한 인물보다는 **스토리 콘텐츠와 AI 체험에 적합한 인물**을 선택하는 것이 중요하다고 판단했다.

우리가 최종적으로 선택한 조합은 다음과 같다.

- 경복궁
- 태조 이성계

태조 이성계를 선택한 이유는 크게 세 가지였다.

### 1. 경복궁과의 연결성이 높았다

경복궁은 조선의 첫 궁궐이며, 태조 이성계는 조선을 건국한 인물이다.

관광지와 역사 인물의 관계가 명확했기 때문에 사용자가 스토리에 몰입하기 쉽다고 생각했다.

### 2. 하나의 인물 안에 여러 역사적 사건이 포함되어 있었다

태조 이성계의 이야기는 단순한 인물 소개에 그치지 않는다.

- 고려 말의 혼란
- 위화도 회군
- 조선 건국
- 한양 천도와 경복궁 건설
- 왕위 계승 문제
- 결말

이처럼 여러 사건을 자연스럽게 하나의 스토리로 연결할 수 있었다.

### 3. AI 역사 인물 체험에 적합했다

태조 이성계는 역사 기록이 비교적 많이 남아 있고, 인물의 성격과 가치관도 어느 정도 파악할 수 있는 인물이다.

따라서 프롬프트를 설계할 때 인물의 말투, 고민, 의사결정 방식을 구체적으로 설정하기 수월했다.

결과적으로 MVP에서는 많은 기능을 얕게 구현하기보다, **경복궁과 태조 이성계라는 하나의 시나리오를 완성도 있게 구현하는 것**을 우선 목표로 삼았다.

---

## 마무리

Histour의 첫 번째 목표는 관광지를 단순히 소개하는 것이 아니었다.

우리는 사용자가 관광지를 방문하기 전에 그 장소가 가진 역사적 맥락을 이해하고, 스토리 기반으로 관광지에 관심을 가질 수 있도록 만들고 싶었다.

이를 위해 공항 기반 관광지 추천, 다국어 지원, 관광지 상세 페이지, 역사 인물 연결 구조를 먼저 설계했다.

다음 글에서는 Histour의 핵심 기능인 **AI 역사 인물 체험과 선택형 스토리 시스템**을 어떻게 구현했는지 정리해보려고 한다.
