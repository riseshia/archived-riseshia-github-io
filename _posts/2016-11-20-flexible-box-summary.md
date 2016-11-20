---
layout: post
title: "Flexible Box Summary"
date: 2016-11-20 17:46:09 +0900
categories:
---

[CodeSchool](https://www.codeschool.com/)에서 3일간 전 코스를 무료로 공개하는 이벤트가
진행중(작성 시점)입니다. 이 글은 그 중에서 `Cracking the Case With Flexbox`를
수강하며 정리한 노트입니다.

## Flexible Box?

- Flexible Box Layout Module Level 1
- Flexbox는 컨텐츠를 정렬하거나 공간을 나누기 위해서 사용하는 CSS 속성 집합
- IE 10이상은 부분 지원, 그 이외의 브라우저는 대부분 지원한다고 봐도 무방.

## Enable UI Patterns in Flexbox

- Equal heights
- Vertically centered content
- Media objects
- Sticky footers
- Column layouts

## What's special?

- 기존의 `display` 속성들은 자기 자신만 변경하고 상속되지 않음
- Flex는 자식 요소들의 레이아웃을 변경함(손자 요소에 대한 영향 X)

## Explanation

### `display`

- `flex`
  - 사용 가능한 너비를 모두 사용.
  - 자식 요소들을 수평으로 배치하되, 너비를 알아서 조절.
- `inline-flex`
  - 컨텐츠의 너비만큼만 사용.

### `flex-wrap`

기본값은 `nowrap`

- `wrap`: 너비가 충분하지 않을 경우 각 요소들이 수평으로 배치되지 않도록 함.
- `wrap-reverse`: `wrap`과 같지만 라인 순서가 반대.

### `flex-direction`

- `row`: 기본값. 요소들이 수평으로 배치됨.
- `row-reverse`: 요소들이 수평으로 역순 배치됨.
- `column`: 요소들이 수직으로 배치됨.
- `column-reverse`: 요소들이 수직으로 역순 배치됨.

### `justify-content`

기본값은 `flex-start`.

- 요소들은 메인 축을 기준으로 정렬되며, 이 속성을 통해서 공간을 나눔.
- 축을 기준으로 출발점, 중간점, 끝점을 사용.
- values
  - `flex-start`, `flex-end`: 각각 시작점, 끝점을 기준으로 배치.
  - `center`: 중간에 배치.
  - `space-between`: 축 전체를 사용하되, 각 요소들 사이에 공백을 넣어서 배치.
  - `space-around`: 축 전체를 사용하되, 각 요소들의 좌우에 공백을 넣어서 배치.

### `order`

- 요소들의 표시 순서를 지정.
- 기본값은 0.
- 음의 정수에서 양의 정수 순서로 정렬

### `align-items`

- 요소들의 정렬 축에 직교하는 방향(cross axis)의 정렬 방향을 결정
- values
  - `stretch`: 기본값. 가능한 모든 공간을 활용해서 확장
  - `flex-start`, `flex-end`: 직교축의 시작점, 끝점 기준으로 정렬
  - `center`: 직교축의 중간점 기준으로 정렬
  - `baseline`: 직교축의 baseline 기준으로 정렬.

### `flex-grow`

- 정렬 축 기준으로 얼마나 공간을 차지해야하는지에 대한 지분을 지정.
- 기본값은 0
- 내용물의 폭(또는 높이)에 따라 실제 계산되는 지분이 달라질 수 있음.
- 화면 사이즈에 따른 크기 변경은 반응형 디자인 시에 주의해야 할 부분.

### `flex-shrink`

- 기본값은 1.
- 0이면 크기 변경에 관계없이 리사이징되지 않는다.

### `flex-basis`

- 요소의 초기 크기를 결정(정렬 축 방향의 길이).
- 기본값은 `auto`
- 단위: `%`, `px`, `em`, `rem`, ...
- 절대값을 사용하면 요소의 너비는 기본적으로 고정된다.
- 다른 속성값이 추가될 가능성이 있음

### `align-self`

- `align-items`를 덮어쓰기 위해서 사용.
- 기본값은 `stretch`
- 받는 값은 `align-items`와 동일.

### `align-content`

- `flex-wrap: wrap`을 정렬하기 위해서 사용.
- 기본값은 `stretch`
- 받는 값은 `justify-content`와 동일.

### `flex`

- Shorthand
- `flex-grow`, `flex-shrink`, `flex-basis`
- 구 IE에서는 미지원.
- 두번째 값이 `%`라면, `flex-shrink`가 1이라고 추측
- `none`: `flex-shrink`를 0으로 설정

### `flex-flow`

- Shorthand
- `flex-direction`, `flex-wrap`
- 둘 중 하나만 입력해도 올바른 속성이 설정된다[사용 가능한 속성값이 전혀 다름]

## Conclusion

- 네이밍이 심히 혼란스러움.
- 그리드를 잡기 위한 div 중첩을 해소할 수 있을 것으로 기대...?
- ~~새 생명을 얻었어요!~~
