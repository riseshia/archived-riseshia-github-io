---
layout: post
title: "Elixir PR 대충 읽기 (5.1-5.25)"
date: 2017-05-25 17:55:42 +0900
categories:
---

## [Integer.parse/2 simpler and faster](https://github.com/elixir-lang/elixir/pull/6044)

직접 구현에서 얼랭 구현을 가져다 쓰는 것으로 변경. **쓸모없이** 빨라짐. 최소 10배 이상의 성능 개선.


## [Fix: ExUnit Setup_all fails with 0 exit status](https://github.com/elixir-lang/elixir/pull/6061)

버그 패치. `setup_all`에서 문제가 발생하면 테스트가 종료 코드 0으로 종료되는 문제가 있었음. ExUnit의 구조를 잘 몰라서 코멘트가 어렵긴한데, RunnerStats에서 테스트 실패를 잡아주는 `handle_cast/2`를 추가히는 걸로 해결함. 이것만 봐서는 왜 `setup_all` 블럭에서만 문제가 생기는지는 잘 모르겠다.


## [Warn when a :__struct__ key is used when building/updating structs](https://github.com/elixir-lang/elixir/pull/6113)

구조체에서 `__struct__`를 사용하면 이전에는 그냥 무시했지만, 이제 경고를 출력하도록 변경.


## [Deprecation of "strip" functions in String module](https://github.com/elixir-lang/elixir/pull/6128)

문자열용 `strip`, `just` 계열 함수를 전부 제거 예정으로 변경하고, `*_trailing`, `*_leading` 함수 사용을 추천하며 1.5에서 적용 예정. 명시적이진 않다고 생각하지만 관용적으로 쓰이는 표현이었는데 이걸 다 갈아버릴줄은... ;ㅅ;


## [Support erlang modules for use as structs](https://github.com/elixir-lang/elixir/pull/6146)

Erlang 모듈을 구조체 취급할 수 있도록 변경합니다. Erlang 레벨의 레코드를 끌어 올리는데 이것저것 실험해본 결과, 이게 제일 좋은거 같더라, 같은 PR. PR에 첨부된 예제를 살펴보면,

```iex
iex> :foo.fetch(%:foo{bar: %:bar{value: 1}})
{:ok, 1}
```

```elixir
# lib/foo.ex
defmodule :foo do
  @type t() :: %__MODULE__{bar: :bar.t(integer())}
  defstruct [:bar]
  def fetch(%:foo{bar: %:bar{value: value}}) do
    {:ok, value}
  end
end
```

```erlang
% src/bar.erl
-module(bar).

-type t(T) :: #{ '__struct__' := ?MODULE, value := T }.
-export_type([t/1]).

-export(['__struct__'/0]).
-export(['__struct__'/1]).

'__struct__'() ->
    #{ '__struct__' => ?MODULE, value => nil }.

'__struct__'(L) when is_list(L) ->
    '__struct__'(maps:from_list(L));
'__struct__'(M) when is_map(M) ->
    maps:fold(fun maps:update/3, '__struct__'(), M).
```

이해한대로 설명하자면 모듈을 노출시키기 위해서, `__struct__/0`, `__struct__/1`을 구현해서 구조체 취급받을 수 있게 만들고 있습니다. 단 이건 Erlang의 구현이 필요한 부분이라, 실제 PR과는 관계가 없으며, 실제 내용물은 elrang 모듈명을 구조체의 이름으로 받을 수 있도록 해줍니다. 다시 말해서 `%:erlang_module{}`를 파싱할 수 있도록 만든 패치가 되겠네요.


## [Inspect Opts Printable String Limit](https://github.com/elixir-lang/elixir/pull/5822)

`inspect/1`에 `printable_limit` 옵션이 추가되었습니다. 이 함수로 출력할 문자열/바이너리의 길이 제한을 줄 수 있게 되며, 기본값은 1024입니다.


## [Add Exception.blame/3 and Exception.blame_mfa/3](https://github.com/elixir-lang/elixir/pull/6127)

Elixir v1.5 & OTP 20에서 지원될 예정이며, 해당 함수는 매칭이 실패했을 때 가능한 매칭 목록을 나열해줍니다. 예를 들어,

```elixir
Access.fetch(:foo, :bar)
```

에서는,

```
** (FunctionClauseError) no function clause matching in Access.fetch/2
    (elixir) lib/access.ex:239: Access.fetch(:foo, :bar)
```

이런 에러 메시지에서

```
** (FunctionClauseError) no function clause matching in Access.fetch/2

The following arguments were given to Access.fetch/2:

    # 1
    :foo

    # 2
    :bar

Attempted function clauses (showing 5 out of 5):

    def fetch(%struct{} = container, key)
    ...
```

이와 같이 좀 더 자세한 결과를 볼 수 있게 됩니다. 이 기능이 지원되면 앞으로는 `Pheonix`같은 오타로 삽을 푸는 일이 줄어들거 같아서 매우 좋습니다. XD

## 기타

- Windows 환경 CI 및 테스트 설정이 착착 진행중입니다.
