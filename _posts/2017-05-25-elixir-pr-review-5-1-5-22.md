---
layout: post
title: "Elixir PR 대충 읽기 (5.1-5.22)"
date: 2017-05-25 17:55:42 +0900
categories:
---

# Elixir 대충 읽기

## [Integer.parse/2 simpler and faster](https://github.com/elixir-lang/elixir/pull/6044)

직접 구현에서 얼랭 구현을 가져다 쓰는 것으로 변경. **쓸모없이** 빨라짐. 최소 10배 이상의 성능 개선.

## [Fix: ExUnit Setup_all fails with 0 exit status](https://github.com/elixir-lang/elixir/pull/6061)

버그 패치. `setup_all`에서 문제가 발생하면 테스트가 종료 코드 0으로 종료되는 문제가 있었음. ExUnit의 구조를 잘 몰라서 코멘트가 어렵긴한데, RunnerStats에서 테스트 실패를 잡아주는 `handle_cast/2`를 추가히는 걸로 해결함. 이것만 봐서는 왜 `setup_all` 블럭에서만 문제가 생기는지는 잘 모르겠다.

## [Warn when a :__struct__ key is used when building/updating structs](https://github.com/elixir-lang/elixir/pull/6113)

구조체에서 `__struct__`를 사용하면 이전에는 그냥 무시했지만, 이제 경고를 출력하도록 변경.

## [Deprecation of "strip" functions in String module](https://github.com/elixir-lang/elixir/pull/6128)

문자열용 `strip`, `just` 계열 함수를 전부 제거 예정으로 변경하고, `*_trailing`, `*_leading` 함수 사용을 추천하며 1.5에서 적용 예정. 명시적이진 않다고 생각하지만 관용적으로 쓰이는 표현이었는데 이걸 다 갈아버릴줄은... ;ㅅ;
