---
layout: post
title: "VIMTUTOR - VIM 도전의 시작"
date: 2016-06-17 05:51:33 +0900
categories:
---

최근들어 사용하는 에디터를 바꿔야할 필요성을 여기저기에서 느끼고 있어서 고민하던 와중-이건 나중에 기회가 되면 따로 이야기해볼 계획입니다-,
이전부터 관심이 있던 Vim을 좀 배워보기로 했습니다. 다만 적당한 초급자용 공부 도구를 찾지 못해서
해매고 있던 와중에 Vimtutor를 발견했습니다. 말 그대로 Vim을 가르쳐주는 툴인데, 실체는 Vim 문서입니다.
문서에서는 사용자에게 이런저런 동작의 설명과 예제를 주면서 기본적인 것들을 배울 수 있도록 도와줍니다.

이 포스팅에서는 Vimtutor를 통해서 배운 사용법 + 그 외에 궁금해서 찾아보았던 것들을 간단하게 요약합니다. 만약 Vim에 입문하고 싶으시다면,
직접 한번 실행해보시는 것을 권장합니다. 한글판도 있으니, 부담없이 도전해보세요.

## 실행하기

Vim을 설치하면 자동으로 따라옵니다. `vimtutor`를 실행해주세요.

## 동작들

### 이동

vi와는 다르게 기본 방향키도 사용할 수 있습니다.

```
h, j, k, l = 왼쪽, 아래, 위, 오른쪽
```

 * `w`: 한 단어 이동(다음 단어의 첫번째 글자)
 * `e`: 한 단어 이동(같은 단어의 마지막 글자)
 * `$`: 줄의 마지막으로 이동
 * `b`: 한 단어 뒤로 이동(같은 단어의 첫번째 글자)

### 텍스트 편집

 * `x`: 커서 위치를 삭제
 * `i`: 입력 모드로 변경
 * `d`: 삭제 명령(조합)
   * `dw`: 커서 위치부터 한 단어(뒤 공백 포함해서 삭제)
   * `de`: 커서 위치부터 한 단어(뒤 공백 포함X)
   * `d$`: 커서 위치부터 줄 끝까지
   * `dd`: 줄 전체
   * `[count]d[we$d]` or `d[count][we$d]`: count 만큼 반복
 * `c`: 복사 명령(조합): `d`와 동일하게 사용 가능
 * `u`: 마지막 명령 취소
 * `U`: 줄 전체를 수정
 * `p`: 커서 아래줄에 붙여넣기
 * `P`: 커서가 있는 줄에 붙여넣기
 * `r`: 커서 위치의 글자 변경하기
 * `R`: 커서 위치부터 변경하기(덮어씀)
 * `c`: 변환 명령(조합)
   * `cw`: `dw` + `i`
   * `ce`: `de` + `i`
   * `c$`: `d$` + `i`
   * `cc`: `dd` + `i`
   * `[count]c[we$c]` or `c[count][we$c]`: count 만큼 반복
 * `o`: 커서 아래에 줄을 만들고 편집 모드로 변경
 * `O`: 커서 위에 줄을 만들고 편집 모드로 변경
 * `a`: `l` + `i`
 * `A`: `$` + `i`

### 위치

 * `Ctrl + G`: 현재 줄번호 확인
 * `[number]Shift + G`: number번째 줄로 이동(number의 기본값 = 마지막 줄)
 * `:[number]`: number번째 줄로 이동

### 검색 및 치환

 * `/[keyword]`: keyword 순차 검색
 * `?[keyword]`: keyword 역순 검색
 * `n`: 다음 찾기
 * `N`: 이전 찾기
 * `%`: (), {}, [] 짝이 되는 괄호로 이동
 * `:s/[find]/[replace]/[flag]`: 치환
 * `:#,#s/[find]/[replace]/[flag]`: #,#(줄 번호) 사이에서만 치환
 * `:%s/[find]/[replace]/[flag]`: 파일 전체에서 치환

#### 치환 플래그

 * `g`: 현재 줄 전체에서 치환
 * `c`: 확인 후 치환

### 옵션

 * `:set ic`: 대소문자 구별 안함
 * `:set hls`: hlsearch. 찾은 글자를 강조
 * `:set is`: incsearch. 증분 검색
 * `:nohlsearch`: 강조 해제

### 기타

 * `:![command]`: 외부 명령 실행하기
 * `:w FILENAME`: 파일 저장하기
 * `:#,# w FILENAME`: #,#(줄 번호) 사이의 텍스트만을 저장
 * `:r FILENAME`: 파일 읽어오기
 * `:help [command]`: 도움말

## Comments

 * Vim 쓰는 사람들의 독특한 따닥, 따닥, 하는 스트로크를 이해할 수 있었다.
 * 영문 모드가 아니면 명령 키가 안먹는다. OTL
 * 이 정도 보고 나니 왠지 해볼만 할 것 같다. ~~같은 심리적인 장벽을 낮추는 효과가 있었다~~

## Some Curious things

 * 파일간에 네비게이션을 하는 설명이 전혀 없더라[..]
 * 여러줄을 Copy & Paste
 