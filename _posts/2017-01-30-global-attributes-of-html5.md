---
layout: post
title: "Global Attributes of HTML5"
date: 2017-01-30 19:25:25 +0900
categories:
---

얼마전에 뜬금없이 `haruair`님에게 트윗을 하나 받았다.

> 글에서 맘대로 Shia님 인용해서 죄송합니다 ￼

뭐지? 싶어서 확인해본 결과 [이런 글](http://www.haruair.com/blog/3815)을 쓰셨는데,
깨진 사례로 내 포스트가 올라가 있었다.
사실 ~~트래픽이 벌려서~~ 저거 자체는 괜찮은데, 원인이 궁금했다.

한줄 요약하면 페이지에서 사용하는 폰트가 문제였다.
`lang`이라는 속성을 사용해서 한글이라고 알려주면 크롬에서 알아서 처리해주는 모양인데,
장기적으로는 다른 언어로도 글을 쓰고 싶다는 욕심이 있어서 해당 해결책은 스킵.
그리고 두번째 선택지(한글 사용한 폰트로 폴백하도록 지정)를 골랐다.

여기까지는 괜찮고 그 뒤에 이어서 이야기를 하다가 나온 말이 오늘의 주제다.

> haruair:`lang`을 여러개 지정할 수 있음 좋을텐데 ;ㅅ; 물론 엘리먼트별로 지정하는 방법도 있습니다 헤헤

> shia: global이라는데요...!?

> haruair: global 의미는 어떤 요소에서나 사용할 수 있다는 뜻입니다.

내 상식속의 global은 전역으로 적용된다는 의미였는데, 어디서든 사용할 수 있다니?

그래서 조사해봤다(링크는 `haruair`님이 주셨지만, 위 인용에서는 생략했다).

## `global` attribute

신뢰와 사랑, 용기로 가득한 mdn을 보자([링크](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes)).
문서에서는 다음과 같이 기술되어 있다.

> Global attributes are attributes common to all HTML elements; they can be used on all elements, though the attributes may have no effect on some elements.

여기서 말하는 `global`은 모든 요소(비표준 포함)에서 사용할 수 있다는 의미에서 `global`인 모양.
목록을 살펴보니 재미있는 것들이 있어서 좀 정리해보았다.

- `accesskey`: 키보드 단축키를 지정. 단 브라우저별로 `prefix`가 다르다.
- `class`:  당연하다고 하면 당연한 속성.
- `contenteditable`: 요소를 편집할 수 있게 함. 이게 전역 속성이었다니...!
- `data-*`: 지금까지 de-facto인줄 알았는데, 사실은 표준이었다. 당연하지만 dom에서 직접 접근할 수 있습니다[..]
- `title`: 요소를 설명할 수 있는 정보를 추가할 수 있게 함.
- `lang`: 이번에 문제가 되셨던 그 분
- `dir`: 문자열의 방향을 지정
- `aria-*`의 일부
- 수많은 이벤트 훅

## 결론

표준 문서 정도는 읽읍시다. 일단 저부터 반성하고 오겠습니다...
