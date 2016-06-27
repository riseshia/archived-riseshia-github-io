---
layout: post
title: "Elixir - OTP: Dependencies and umbrella projects"
date: 2016-06-27 22:10:45 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 많습니다. 원문은 최하단의 링크를 참고하세요.

## Introduction

이 장에서는, Mix에서 어떻게 의존성을 관리하는지에 대해서 알아봅니다.

`kv` 애플리케이션이 완성되었으므로, 이제 첫 번째 장에서 정의했던 요청들을 처리할 서버를 구현해볼 시간입니다.

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

하지만 `kv` 애플리케이션에 코드를 추가하기 전에 `kv` 애플리케이션의 클라이언트가 될 TCP 서버를 별도의 애플리케이션으로 만들겠습니다. Elixir의 생태계와 전체 런타임은 여러 개의 애플리케이션을 실행할 수 있도록 겨냥되었으므로, 하나의 커다란 앱을 만드는 것보다 작은 애플리케이션으로 쪼개서 서로 협력하게 하는 것은 자연스럽습니다.

새 애플리케이션을 생성하기 전에 우선 Mix가 어떻게 의존성을 관리하는지에 대해서 이야기합시다. 보통 다루어야 하는 의존성은 크게 2가지가 있습니다: 내부와 외부 의존성입니다. Mix는 이 두 가지를 모두 처리할 수 있는 방법을 지원합니다.

## 외부 의존성

외부 의존성은 비즈니스 로직에 직접 연결되지 않는 것들입니다. 예를 들어서 만들었던 분산 KV 애플리케이션을 위해서 HTTP API가 필요하다면 외부 의존성으로서 [Plug](https://github.com/elixir-lang/plug)를 사용할 수 있을 겁니다.

외부 의존성을 설치하는 방법은 간단합니다. 가장 흔한 방법으로는 [Hex 패키지 매니저](https://hex.pm)를 사용하여 `mix.exs` 파일의 deps 함수에 의존성 목록을 작성하는 것입니다:

```elixir
def deps do
  [{:plug, "~> 1.0"}]
end
```

이 의존성은 Hex에 등록된 1.0.x 버전 대의 가장 최신인 Plug를 의미합니다. `~>`를 통해서 특정 버전 번호의 이후를 나타냅니다. 요구 버전을 기술하는 방법에 대해서 더 자세히 알고 싶다면 [Version 모듈에 대한 문서](http://elixir-lang.org/docs/stable/elixir/Version.html)를 확인하세요.

일반적으로 Hex에는 안정적인 릴리스가 등록됩니다. Mix는 git의 주소를 직접 입력할 수 있으므로 개발 상태에 있는 외부 의존성을 사용할 수도 있습니다:

```elixir
def deps do
  [{:plug, git: "git://github.com/elixir-lang/plug.git"}]
end
```

프로젝트에 의존성을 추가하면 Mix가 *반복되는 빌드*를 보장하기 위한 `mix.lock` 파일을 생성합니다. 이 잠금 파일은 프로젝트를 사용하는 사람들이 모두 같은 버전의 의존성을 사용할 수 있게끔, 버전 관리 시스템에 포함되어야 합니다.

Mix는 의존성을 다루기 위한 다양한 태스크를 제공하고 있으며 `mix help`를 통해서 확인할 수 있습니다: 

```bash
$ mix help
mix deps              # 의존성의 목록과 상태를 확인
mix deps.clean        # 주어진 의존성 파일들을 제거
mix deps.compile      # 의존성을 컴파일
mix deps.get          # 유효하지 않은 의존성을 갱신하기
mix deps.unlock       # 넘긴 의존성의 잠금 상태를 해제하기
mix deps.update       # 넘긴 의존성을 업데이트하기
```

가장 일반적인 태스크는 `mix deps.get`와 `mix deps.update`입니다. 의존성을 가져오면 이들은 자동으로 컴파일 됩니다. `mix help deps`를 입력하면 더 많은 정보를 보실 수 있으며, [Mix.Tasks.Deps 모듈에 대한 문서](http://elixir-lang.org/docs/stable/mix/Mix.Tasks.Deps.html)를 읽으셔도 좋습니다.

## 내부 의존성

내부 의존성은 프로젝트의 명세에 대한 것들입니다. 이들은 프로젝트/회사/조직의 외부에 두는 것은 상식적이지 않습니다. 대부분의 경우, 기술적, 경제적, 또는 사업상의 이유로 이것들을 비공개로 유지하길 원합니다.

만약 내부 의존성을 가지고 있다면, Mix는 이들을 다루기 위한 두 가지 방법을 제공합니다: Git 저장소와 엄브렐라 프로젝트입니다.

예를 들어 `kv` 프로젝트를 Git 저장소에 올렸다면 이를 사용하기 위해서는 그저 deps에 이 정보를 기술하기만 하면 됩니다:

```elixir
def deps do
  [{:kv, git: "https://github.com/YOUR_ACCOUNT/kv.git"}]
end
```

만약 저장소가 비공개인 경우라면 비공개 URL인 `git@github.com:YOUR_ACCOUNT/kv.git`을 필요로 할 수도 있습니다. 어떤 경우든 Mix는 개발자의 권한을 증명 가능한 한 문제 없이 가져올 것입니다.

내부 의존성으로 Git 의존성을 사용하는 것은 Elixir에서는 권장되지 않습니다. 런타임, 그리고 Elixir의 생태계가 이미 그러한 개념을 제공하고 있다는 점을 떠올리세요. 단 하나의 프로젝트라 하더라도, 코드들을 논리적으로 잘 구성된 애플리케이션들로 분리하길 바랍니다.

하지만 만약 애플리케이션을 각각 분리된 프로젝트로서 Git 저장소에 올린다면 프로젝트는 점점 코드를 작성하기보다 Git 저장소들을 관리하기 위한 시간이 많아지게 되어 유지 보수가 어려워질 것입니다.

이러한 이유로 Mix는 "엄브렐라 프로젝트"를 지원합니다. 엄브렐라 프로젝트는 각각 다른 코드 저장소에 있는 여러 개의 애플리케이션을 관리하는 하나의 프로젝트를 생성할 수 있게끔 해줍니다. 이것이 바로 다음 장에서 확인할 것들입니다.

새 Mix 프로젝트를 만들어보죠. 이름은 `kv_umbrella`라고 짓고, 이 새 프로젝트는 `kv` 애플리케이션과 `kv_server`라는 새 애플리케이션을 포함할 것입니다. 이 폴더 구조는 다음과 같이 구성될 겁니다:

    + kv_umbrella
      + apps
        + kv
        + kv_server

이러한 접근 방식의 흥미로운 점은 Mix는 이러한 구조의 프로젝트들을 위해 `apps`에 존재하는 모든 애플리케이션을 동시에 컴파일하거나 테스트할 수 있는 명령어 등의 편의성을 제공한다는 점입니다. 하지만 이 모든 것들이 `apps` 폴더에 함께 존재함에도 불구하고, 여전히 서로 독립되어 있으며, 원한다면 각각 고립된 환경으로 빌드, 테스트, 배포할 수 있습니다.

그럼 이제 시작해봅시다!

## 엄브렐라 프로젝트

`mix new`를 사용해서 새 프로젝트를 생성해 봅시다. 이름은 `kv_umbrella`라고 짓고, `--umbrella` 옵션을 넘길 필요가 있습니다. 이미 만들어둔 `kv` 프로젝트의 내부에 생성하지 마세요!

```bash
$ mix new kv_umbrella --umbrella
* creating .gitignore
* creating README.md
* creating mix.exs
* creating apps
* creating config
* creating config/config.exs
```

출력된 정보로부터, 꽤 적은 파일이 생성된 것을 알 수 있습니다. 생성된 `mix.exs`도 다릅니다. 한번 확인해보죠(주석은 생략했습니다):

```elixir
defmodule KvUmbrella.Mixfile do
  use Mix.Project

  def project do
    [apps_path: "apps",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     deps: deps]
  end

  defp deps do
    []
  end
end
```

이전에 생성했던 프로젝트와 이 프로젝트가 다른 점은 `apps_path: "apps"`가 프로젝트 정의에 포함되었다는 점입니다. 이는 바로 이 프로젝트가 마치 우산처럼 동작한다는 것을 의미합니다. 이러한 프로젝트들은 (자식들과는 공유하지 않는) 자신만의 의존성을 가짐에도 불구하고, 소스 코드나 테스트를 포함하지 않습니다. 이제 apps 폴더의 내부에 새 애플리케이션을 생성합시다.

apps 폴더로 들어가서 `kv_server`를 만들어봅시다. 이번에는 `--sup` 플래그를 넘겨서 직접 관리 트리에 정보를 추가하는 대신 Mix가 관리 트리를 자동으로 생성해주도록 합시다:

```bash
$ cd kv_umbrella/apps
$ mix new kv_server --module KVServer --sup
```

생성된 파일들은 처음에 `kv` 프로젝트를 생성했을 때와 거의 유사합니다. `mix.exs`를 열어보죠:

```elixir
defmodule KVServer.Mixfile do
  use Mix.Project

  def project do
    [app: :kv_server,
     version: "0.0.1",
     build_path: "../../_build",
     config_path: "../../config/config.exs",
     deps_path: "../../deps",
     lockfile: "../../mix.lock",
     elixir: "~> 1.2",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     deps: deps]
  end

  def application do
    [applications: [:logger],
     mod: {KVServer, []}]
  end

  defp deps do
    []
  end
end

```

첫번째로, `kv_umbrella/apps`에 프로젝트를 생성했기 때문에 Mix는 알아서 엄브렐라의 구조를 탐지하고 이 프로젝트의 정의에 다음의 4줄을 추가했습니다:

```elixir
build_path: "../../_build",
config_path: "../../config/config.exs",
deps_path: "../../deps",
lockfile: "../../mix.lock",
```

이 옵션들은 모든 의존성이 `kv_umbrella/deps`로부터 넘어오며, 같은 빌드, 설정, 잠금 파일들을 공유한다는 것을 의미합니다. 이는 의존성을 가져온 뒤 한번 컴파일하면, 엄브렐라 프로젝터 전체에서 사용할 수 있도록 해줍니다.

두 번째로 `mix.exs`의 `application` 함수가 다릅니다:

```elixir
def application do
  [applications: [:logger],
   mod: {KVServer, []}]
end
```

`--sup` 옵션을 넘겼기 때문에 Mix는 알아서 `KVServer`는 애플리케이션 콜백 모듈이라는 것을 명시하는 `mod: {KVServer, []}`를 추가해줍니다. `KVServer`는 애플리케이션 관리 트리에서 실행될 겁니다.

그리고 `lib/kv_server.ex`를 열어보죠:

```elixir
defmodule KVServer do
  use Application

  def start(_type, _args) do
    import Supervisor.Spec, warn: false

    children = [
      # worker(KVServer.Worker, [arg1, arg2, arg3])
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

여기에서는 애플리케이션 콜백 함수, `start/2`에 대해서 정의하고 있으며 `KVServer.Supervisor`를 정의하는 대신 `Supervisor` 모듈을 사용하여 슈퍼바이저를 한 줄로 간단하게 정의하고 있습니다! 이러한 슈퍼바이저들에 대해서는 [Supervisor 모듈 문서](http://elixir-lang.org/docs/stable/elixir/Supervisor.html)를 참고해주세요.

첫번째 엄브렐라의 자식을 만들어 보았습니다. `apps/kv_server`에서 테스트를 실행할 수 있습니다만, 이는 그다지 재미가 없습니다. 대신에 엄브렐라 프로젝트의 최상위 폴더로 가서 `mix test`를 실행해보죠:

```bash
$ mix test
```

잘 동작합니다!

`kv_server`가 `kv`에서 정의해두었던 기능을 사용하는 것이 최종 목표이므로, 이제 `kv`를 애플리케이션의 의존성으로 등록해야 합니다.

## 엄브렐라 의존성

Mix 간단하게 한 엄브렐라의 자식 애플리케이션이 다른 자식 애플리케이션에 의존할 수 있도록 해줍니다. `apps/kv_server/mix.exs`를 열고 `deps/0` 함수를 다음과 같이 변경하세요:

```elixir
defp deps do
  [{:kv, in_umbrella: true}]
end
```

이 변경은 `:kv`를 `:kv_server`의 의존성으로서 취급하도록 해줍니다. 이를 통해 `:kv`에 정의된 모듈들을 사용할 수 있습니다만, `:kv` 애플리케이션을 자동으로 실행해주지는 않습니다. 이를 위해서는 `:kv`를 `application/0`에 등록할 필요가 있습니다:

```elixir
def application do
  [applications: [:logger, :kv],
   mod: {KVServer, []}]
end
```

이제 Mix는 `:kv` 애플리케이션이 `:kv_server`가 시작되기 전에 실행되어야 한다는 점을 보장합니다.

마지막으로 지금까지 작성했던 `kv` 애플리케이션을 새 엄브렐라 프로젝트의 `apps` 폴더에 복사합니다. 마지막 폴더 구조는 이전에 언급했던 구조와 일치해야 합니다:

    + kv_umbrella
      + apps
        + kv
        + kv_server

이제 `apps/kv/mix.exs`를 변경하여 `apps/kv_server/mix.exs`에서처럼 엄브렐라의 구성 요소라는 것을 알 수 있도록 만들어야 합니다. `apps/kv/mix.exs`를 열고 `project` 함수에 다음을 추가하세요:

```elixir
build_path: "../../_build",
config_path: "../../config/config.exs",
deps_path: "../../deps",
lockfile: "../../mix.lock",
```

엄브렐라 프로젝트의 최상위 폴더에서 `mix test`를 실행하면 두 프로젝트의 테스트들을 한 번에 실행할 수 있습니다. 좋습니다!

엄브렐라 프로젝트는 다수의 애플리케이션을 쉽게 조직화하고 관리할 수 있도록 돕는다는 점을 기억하세요. `apps` 폴더에 들어있는 앱들은 여전히 각각 독립되어 있습니다. 그 애플리케이션 간의 의존성은 명시적으로 기술되어야 합니다. 이를 통해 함께 개발될 수 있으며, 필요하다면 독립적으로 컴파일, 테스트, 배포할 수 있도록 해줍니다.

## 정리하기

이 장에서는 Mix 의존성과 엄프렐라 프로젝트에 대해서 배웠습니다. `kv`와 `kv_server`가 이 프로젝트에서만 중요한 내부 의존성이라고 판단했기 때문에 엄브렐라 프로젝트를 만들기로 했었습니다.

앞으로는 애플리케이션들을 작성하고, 이들을 다른 프로젝트에서도 사용할 수 있는 간결한 구조로 추출할 수 있을 때가 있을 것입니다. 그런 경우에는 Git이나 Hex 의존성을 사용하면 됩니다.

여기에 의존성을 사용하면서 스스로에게 물어볼 수 있는 몇몇 질문들이 있습니다. 이 애플리케이션은 이 프로젝트의 바깥에서도 의미가 있습니까?:

* 만약 아니라면, 엄브렐라 프로젝트를 사용하세요.
* 만약 맞다면, 이 프로젝트를 회사/조직 바깥에서도 공유할 수 있습니까?
  * 만약 아니라면 비공개 Git 저장소를 사용하세요.
  * 만약 그렇다면, Git 저장소에 코드를 올리고, [Hex](https://hex.pm)를 통해서 릴리스하세요.

엄브렐라 프로젝트가 준비되었으므로, 이제 서버를 위한 코드를 작성할 시간입니다.

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Elixir - OTP: Dependencies and umbrella projects](http://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-apps.html)
