---
layout: post
title: "Elixir - OTP: Introduction to Mix"
date: 2016-05-09 21:09:01 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 거의 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 많습니다. 원문은 최하단의 링크를 참고하세요.

# Introduction to Mix

이 가이드에서는 자신만의 관리 구조(supervision tree), 설정, 테스트 등을 포함하는 완전한 Elixir 애플리케이션을 어떻게 빌드하는지 배울 것입니다.

이 애플리케이션은 분산된 키-값 저장소처럼 동작합니다. 우리는 키-값 쌍을 버킷에 집어넣고 이것들을 여러 노드에 분산시킬겁니다. 나아가서 이 노드들에게 아래와 같이 접근할 수 있는 간단한 클라이언트도 만들어 볼 겁니다.

```
CREATE shopping
OK

PUT shopping milk 1
OK

PUT shopping eggs 3
OK

GET shopping milk
1
OK

DELETE shopping eggs
OK
```

이 키-값 저장소를 만들기 위해서 3가지 도구를 사용할 예정입니다.

* ***OTP*** _(Open Telecom Platform)_는 Erlang에 포함되어 있는 라이브러리의 집합입니다. Erlang 개발자들은 OTP를 사용해서 튼튼한 애플리케이션을 만듭니다. 이 챕터에서는 OTP가 Elixir에서 어떤 모습으로 사용하고 있는지를 살펴볼 겁니다.

* ***Mix***는 Elixir에서 기본으로 제공되며, 태스크의 생성, 컴파일, 애플리케이션의 테스트, 의존성 관리 등의 작업을 도와주는 빌드 도구입니다.

* ***ExUnit***은 Elixir에 포함되어 있는 테스트 유닛 기반 프레임워크입니다.

이 챕터에서는 Mix를 사용해서 첫번째 프로젝트를 만들고, 앞으로 함께할 <abbr title="Open Telecom Platform">OTP</abbr>, Mix, ExUnit에 대해서 살펴봅니다.

> Note: 이 가이드는 Elixir v1.2.0이나 그 이상을 요구합니다. Elixir의 버전은 `elixir -v` 명령을 사용하여 확인할 수 있으며, 필요하다면 [가이드의 첫번째 챕터](http://elixir-lang.org/getting-started/introduction.html)를 참고하여 버전을 업그레이드하세요.
>
> 만약 가이드에 대한 질문이나 바라는 점이 있다면 [메일링 리스트](https://groups.google.com/d/forum/elixir-lang-talk)나 [이슈 트래커](https://github.com/elixir-lang/elixir-lang.github.com/issues)를 통해서 알려주세요. 여러분들의 피드백은 가이드를 좀 더 명확하게, 그리고 최신으로 유지하기위해 무척 중요합니다!

## 첫번째 프로젝트

Elixir를 설치하면 실행 가능한 `elixir`, `elixirc`, `iex` 뿐만 아니라, `mix`라는 이름의 Elixir 스크립트도 생성됩니다.

그럼 `mix new`를 호출해서 첫번째 프로젝트를 만들어봅시다. 프로젝트의 이름을 인자(이번에는 `kv`를 사용할 겁니다)로 넘기고, 메인 모듈의 이름이 기본값인 `Kv`가 아닌, 모두 대문자(`KV`)가 되도록 만듭시다.

```bash
$ mix new kv --module KV
```

Mix는 `kv`라는 이름의 폴더와 몇개의 파일을 생성합니다.

    * creating README.md
    * creating .gitignore
    * creating mix.exs
    * creating config
    * creating config/config.exs
    * creating lib
    * creating lib/kv.ex
    * creating test
    * creating test/test_helper.exs
    * creating test/kv_test.exs

각각의 파일에 대해서 간단하게 살펴보죠.

> Note: Mix는 실행가능한 Elixir 코드입니다. 다시 말해서 `mix` 명령어를 실행하려면 Mix의 바이너리가 PATH에 포함되어 있어야 한다는 의미입니다. 만약 그렇지 않다면 `elixir`에 인자로 넘김으로서 실행할 수도 있습니다.
>
> ```bash
> $ bin/elixir bin/mix new kv --module KV
> ```
>
> 또는 -S 옵션을 사용해서 경로에 있는 어떤 스크립트라도 실행할 수도 있습니다.
>
> ```bash
> $ bin/elixir -S mix new kv --module KV
> ```
>
> -S 옵션을 사용하면, `elixir`는 현재 PATH 상의 모든 곳에서 스크립트를 찾아서 실행하려고 시도합니다.

## 프로젝트 컴파일하기

`mix.exs`라는 이름의 파일이 새 프로젝트 폴더(`kv`)에 생성되었으며, 이 파일은 프로젝트에 대한 설정을 관리하는 책임을 가집니다. 좀 더 자세히 살펴보죠(주석은 제외했습니다).

```elixir
defmodule KV.Mixfile do
  use Mix.Project

  def project do
    [app: :kv,
     version: "0.0.1",
     elixir: "~> 1.2",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     deps: deps]
  end

  def application do
    [applications: [:logger]]
  end

  defp deps do
    []
  end
end
```

이 `mix.exs` 파일은 두 개의 메소드를 정의합니다: `project`는 프로젝트의 이름, 버전 정보와 같은 설정을 돌려주며 `application`은 애플리케이션 파일을 생성합니다.

그리고 `deps`라는 비공개 메소드가 있습니다. 이는 `project`에서 호출되며, 여기에서는 프로젝트의 의존성을 정의하게 됩니다. `deps`를 반드시 여러 개의 분리된 함수로 정의할 필요는 없지만, 이를 통해서 프로젝트의 설정을 간결하게 유지할 수 있습니다.

Mix는 `lib/kv.ex` 파일에 간단한 모듈 정의를 생성합니다:

```elixir
defmodule KV do
end
```

이 구조는 프로젝트를 컴파일 하기에 충분합니다:

```bash
$ cd kv
$ mix compile
```

Will output:

    Compiled lib/kv.ex
    Generated kv app
    Consolidated List.Chars
    Consolidated Collectable
    Consolidated String.Chars
    Consolidated Enumerable
    Consolidated IEx.Info
    Consolidated Inspect

`lib/kv.ex` 파일이 컴파일 되며, 애플리케이션의 매니페스트 파일이 `kv.app`이라는 이름으로 생성됩니다. 이어서 [이전에 가이드에서 설명했던 것처럼 모든 프로토콜들이 통합됩니다](http://elixir-lang.org/getting-started/protocols.html#protocol-consolidation). `mix.exs` 파일에 정의된 옵션을 통해 컴파일된 모든 파일들은 `_build` 폴더에 저장됩니다.

일단 프로젝트가 컴파일 되면, 다음을 통해서 프로젝트 내부에 `iex` 세션을 열 수 있습니다.

```bash
$ iex -S mix
```

## 테스트 실행하기

Mix는 프로젝트 테스트에 필요한 구조도 함께 생성해줍니다. Mix 프로젝트는 일반적으로 테스트 파일들을 `lib` 폴더에 있는 파일들에 대해서 `<파일명>_test.exs`라는 네이밍 규칙을 사용하여 `test` 폴더에 위치시킵니다. 이런 이유로 우리는 이미 `lib/kv.ex`의 테스트를 담당하는 `test/kv_test.exs` 파일을 찾을 수 있습니다. 지금 시점에서는 많은 것을 해주지 않지만요.

```elixir
defmodule KVTest do
  use ExUnit.Case
  doctest KV

  test "the truth" do
    assert 1 + 1 == 2
  end
end
```

중요한 점을 몇가지 짚고 넘어갑시다:

1. 테스트 파일은 Elixir 스크립트 파일입니다. 실행하기 전에 별도의 컴파일을 할 필요가 없기 때문입니다.

2. 테스트 모듈의 이름을 `KVTest`라고 명명했으며, [`ExUnit.Case`](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.Case.html)를 사용하여 테스팅 API와 `test/2` 매크로를 사용하는 간단한 테스트를 정의합니다.

Mix는 `test/test_helper.exs`라는 이름의 파일도 생성하며, 이는 테스트 프레임워크의 설정을 담당하게 됩니다.

```elixir
ExUnit.start()
```

이 파일은 테스트를 실행하기 전에 매번 자동적으로 불러와집니다. 테스트는 `mix test`로 실행할 수 있습니다:

    Compiled lib/kv.ex
    Generated kv app
    [...]
    .

    Finished in 0.04 seconds (0.04s on load, 0.00s on tests)
    1 tests, 0 failures

    Randomized with seed 540224

`mix test`를 실행하면 Mix가 소스 파일을 컴파일하고 애플리케이션 파일을 다시 생성하는 것을 확인하세요. 이는 Mix가 여러 환경을 지원하기 때문이며, 여기에 대해서는 잠시 뒤에 다룰겁니다.

나아가서 ExUnit이 성공한 테스트에 대해서 점을 출력하고, 테스트의 순서도 자동으로 랜덤하게 실행하는 것을 확인할 수 있습니다. 실험삼아 실패하는 테스트를 작성하고 어떻게 되는지 살펴보죠.

`test/kv_test.exs`에 있는 선언문을 다음과 같이 변경하세요:

```elixir
assert 1 + 1 == 3
```

이제 `mix test`를 다시 실행하세요(이번에는 컴파일이 이루어지지 않는다는 점을 확인하세요):

    1) test the truth (KVTest)
       test/kv_test.exs:5
       Assertion with == failed
       code: 1 + 1 == 3
       lhs:  2
       rhs:  3
       stacktrace:
         test/kv_test.exs:6

    Finished in 0.05 seconds (0.05s on load, 0.00s on tests)
    1 tests, 1 failures

각각의 실페에 대해서 ExUnit은 이를 포함하는 테스트의 이름과 `==` 연산자의 죄변(lhs)과 우변(rhs)에 실제로 어떤 값이 넘어왔는지에 대한 자세한 설명을 제공합니다.

실패의 두번째 줄에서는 테스트 파일의 이름과 몇번째 줄에 테스트가 정의되어 있는지를 보여줍니다. 파일 이름과 행 정보를 포함하여 두번째 줄을 완전히 복사한 뒤 `mix test`의 뒤에 넘겨주면 Mix는 해당 테스트만을 실행합니다:

```bash
$ mix test test/kv_test.exs:5
```

이 방법은 프로젝트를 만들 때에 특정 테스트만을 쉽고 빠르게 실행할 수 있게 해주기 때문에 무척 유용합니다.

마지막으로 테스트의 실패에 대한 스택 트레이스 정보를 통해서 테스트에 대한 정보, 보통은 소스 파일의 어디에서 문제가 있었는지에 대한 정보를 얻을 수 있습니다.

## 환경

Mix는 환경이라는 개념을 지원합니다. 이는 개발자에게 특정 시나리오에 대한 여러 옵션들을 별도로 지정할 수 있게 해줍니다. 기본적으로 Mix는 다음의 환경을 알고 있습니다:

* `:dev` - Mix의 태스크(ex: `compile`)가 실행되는 기본 환경
* `:test` - `mix test`의 환경
* `:prod` - 실제 환경에서 프로젝트를 실행할때 사용하게 될 환경

이 환경들은 현재 프로젝트에서만 적용됩니다. 나중에 보겠지만, 프로젝트에 추가하는 모든 의존성들은 기본적으로 `:prod` 환경에서도 동작합니다.

`mix.exs` 파일에서 [`Mix.env` 함수](http://elixir-lang.org/docs/stable/mix/Mix.html#env/1)를 사용하면 현재 환경이 무엇인지 아톰으로서 반환되므로, 이를 사용하여 환경별로 값을 설정할 수 있습니다. 이미 `:build_embedded`와 `:start_permanent` 옵션에서 사용하고 있습니다.

```elixir
def project do
  [...,
   build_embedded: Mix.env == :prod,
   start_permanent: Mix.env == :prod,
   ...]
end
```

소스 코드를 컴파일 할 때에, Elixir는 `_build` 폴더에 코드들을 복사합니다. 하지만 많은 경우 불필요한 복사를 피하기 위해서, Elixir는 파일 시스템의 링크 기능을 사용해서 `_build`에서 실제 소스 파일로 연결합니다. 만약 `true`라면 `:build_embedded`는 이 동작을 수행하지 않고 애플리케이션의 실행에 필요한 모든 것들을 `_build`에 제공합니다.

비슷하게 `true`라면 `:start_permanent` 옵션이 애플리케이션을 영속적인 모드(permanent mode)로 실행하며, 이는 애플리케이션의 관리 트리가 종료되면 Erlang VM 역시 종료될 것이라는 의미입니다. 개발이나 테스트환경에서는 VM 인스턴스를 계속해서 켜두는 것이 디버깅에서는 유용하므로 평소에는 이 기능을 켜두지 않을 겁니다.

Mix는 `test` 명령을 제외한 모든 경우에 기본적으로 `:dev`를 사용합니다. 이 기본 환경은 `MIX_ENV` 환경 변수를 통해서 변경할 수 있습니다.

```bash
$ MIX_ENV=prod mix compile
```

윈도우에서는:

```batch
> set "MIX_ENV=prod" && mix compile
```

## 탐험하기

Mix에 대해서는 더 설명할 것이 많으므로, 프로젝트 만들면서 조금씩 더 살펴보도록 합시다. [개략적인 설명은 Mix 문서에서 확인할 수 있습니다](http://elixir-lang.org/docs/stable/mix/).

사용가능한 모든 명령 목록은 help를 호출해서 확인할 수 있다는 점을 기억해두세요:

```bash
$ mix help
```

`mix help TASK`를 통해서 특정 명령에 대한 상세한 설명을 확인할 수도 있습니다.

그럼 코드를 작성해보죠!

## Reference
 * [Elixir Homepage](http://elixir-lang.org)
 * [Introduction to Mix](http://elixir-lang.org/getting-started/mix-otp/introduction-to-mix.html)
