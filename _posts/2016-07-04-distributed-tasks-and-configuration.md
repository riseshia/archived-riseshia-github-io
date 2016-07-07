---
layout: post
title: "Elixir - OTP: Distributed tasks and configuration"
date: 2016-07-04 21:16:58 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 있습니다. 원문은 최하단의 링크를 참고하세요.

## Introduction

마지막 장에서는 `:kv` 애플리케이션으로 돌아가 버킷 이름을 기반으로 하는 노드 간에 분산된 요청을 할 수 있게끔 라우팅 레이어를 추가해볼 것입니다.

라우팅 레이어는 다음과 같은 형식으로 구성된 라우팅 테이블을 받을 것입니다.

    [{?a..?m, :"foo@computer-name"},
     {?n..?z, :"bar@computer-name"}]

라우터는 버킷 이름의 첫 번째 바이트를 받은 뒤, 테이블에서 거기에 적합한 노드를 찾아옵니다. 예를 들어, 버킷 이름이 "a"로 시작한다면(`?a`는 유니코드 코드 포인트로, "a"를 가리킵니다), 이때 받아오는 노드는 `foo@computer-name`일 것입니다.

만약 찾은 노드가 현재 요청을 처리하고 있는 노드라면 라우팅이 끝나는 대로, 이 노드는 요청된 연산을 실행할 것입니다. 만약 돌려받은 노드가 현재 요청을 처리하고 있는 노드가 아니라면 이 요청을 해당 노드에 넘겨주게 됩니다. 만약 어떤 노드도 발견하지 못한다면 에러를 발생시킬 것입니다.

왜 라우팅 테이블에서 발견한 노드에 직접 해당 요청을 처리하라고 말하는 대신, 해당 노드의 프로세스로 라우팅 요청을 보내는지 궁금할 수 있습니다. 라우팅 테이블이 모든 노드 간에 공유가 가능할 정도로 간단한 상황에서는 라우팅 요청을 보내는 것이 이후에 애플리케이션이 성장했을 때에 라우팅 테이블을 분리하기 쉽습니다. 어느 시점이 되면 `foo@computer-name`는 라우팅 요청만을 책임지게 되며, 자기가 알고 있는 범위 내의 다른 노드들에 대한 요청을 처리할 것입니다. 이런 식으로 `bar@computer-name`는 변경에 대해서 알 필요가 없습니다.

> Note: 우리는 이번 장 전체에 걸쳐서 같은 기기상에서 2개의 노드를 사용할 것입니다. 원한다면 같은 네트워크상에 있는 다른 기기들을 사용해도 좋습니다. 단 이 경우에는 몇몇 사전 작업이 필요합니다. 우선 모든 기기상의 `~/.erlang.cookie` 파일에 같은 값을 가지고 있어야 합니다. 두 번째로 [epmd](http://www.erlang.org/doc/man/epmd.html)가 사용하는 포트가 열려있는지 확인하세요(`epmd -d`를 통해 디버깅 정보를 볼 수 있습니다). 세 번째로 일반적인 분산 처리에 대해서 배우고 싶다면 [Learn You Some Erlang에 있는 훌륭한 Distribunomicon 장을 추천합니다](http://learnyousomeerlang.com/distribunomicon).

## 분산된 첫 코드

Elixir는 노드들을 연결하고, 서로 정보를 교환할 수 있는 기능이 있습니다. 사실 프로세스에 있는 개념인 메시지 교환을 분산 환경에서도 그대로 사용합니다만, 이는 Elixir의 프로세스가 *위치를 가리지 않기*때문입니다. 이 말은 메시지를 보낼 때, 수신 프로세스가 같은 노드인지, 다른 노드인지 관계없이 <abbr title="Virtual Machine">VM</abbr>이 잘 처리해줄 것이라는 의미입니다.

분산된 코드를 실행하기 위해서, 우선 이름을 사용해서 <abbr title="Virtual Machine">VM</abbr>을 실행해야 합니다. 이름은 짧아도(같은 네트워크 내에 있을 때) 좋고, 길어도(컴퓨터 전체 주소가 필요할 때) 좋습니다. 그럼 새 IEx 세션을 시작해보죠:

```bash
$ iex --sname foo
```

그러면 이전과는 조금 다르게 컴퓨터 이름과 노드 이름이 포함된 프롬프트를 볼 수 있을 것입니다:

    Interactive Elixir - press Ctrl+C to exit (type h() ENTER for help)
    iex(foo@jv)1>

여기에서 컴퓨터 이름은 `jv`이며, 그러므로 이 예제에서는 `foo@jv`를 볼 수 있지만, 각자의 세션에서는 다른 정보를 보여줄 것입니다. 여기에서는 `foo@computer-name`을 이름이라고 가정하고 진행할 것이며, 필요하다면 실제 본인의 것으로 변경한 뒤 실행하길 바랍니다.

이 셸에서 `Hello`라는 모듈을 정의해봅시다.

```iex
iex> defmodule Hello do
...>  def world, do: IO.puts "hello world"
...> end
```

만약 같은 네트워크에 Erlang과 Elixir가 설치된 다른 컴퓨터가 있다면, 거기에서 다른 셸을 실행하세요. 없다면 그냥 다른 터미널 창에서 새 IEx 세션을 시작하세요. 어느 쪽이든 이름은 `bar`라고 지정하도록 하죠.

```bash
$ iex --sname bar
```

새 IEx 세션에서는 `Hello.world/0`에 접근할 수 없는 것을 알 수 있습니다.

```iex
iex> Hello.world
** (UndefinedFunctionError) undefined function: Hello.world/0
    Hello.world()
```

하지만 `bar@computer-name`에서 `foo@computer-name` 상에 프로세스를 생성할 수 있습니다! 한번 해보죠(`@computer-name`은 실제 컴퓨터 이름을 사용하세요):

```iex
iex> Node.spawn_link :"foo@computer-name", fn -> Hello.world end
#PID<9014.59.0>
hello world
```

Elixir는 다른 노드에 프로세스를 생성하고 pid를 돌려줍니다. 코드는 `Hello.world/0` 함수가 존재하는 노드에서 실행되므로 이 함수를 호출할 수 있습니다. 그 결과인 "hello world"가 `foo`가 아닌 현재 노드 `bar`에 출력되었다는 점에 주의하세요. 다시 말하면, 출력될 메시지가 `foo`에서 `bar`로 전송되었습니다. 이는 프로세스가 생성된 다른 노드(`foo`)가 현재 노드(`bar`)가 속해있는 그룹의 리더이기 때문입니다. 우리는 [IO](http://elixir-lang.org/getting-started/io-and-the-file-system.html#processes-and-group-leaders)에서 그룹 리더에 대해 간략하게 이야기했었습니다.

평소대로 `Node.spawn_link/2`로 받은 pid에 메시지를 던지거나 받을 수 있습니다. 간단한 핑퐁 예제를 실행해보죠:

```iex
iex> pid = Node.spawn_link :"foo@computer-name", fn ->
...>   receive do
...>     {:ping, client} -> send client, :pong
...>   end
...> end
#PID<9014.59.0>
iex> send pid, {:ping, self}
{:ping, #PID<0.73.0>}
iex> flush
:pong
:ok
```

지금까지의 설명으로 `Node.spawn_link/2`를 사용해서 분산된 작업을 하고 싶을 때마다 원격 노드에 프로세스를 생성하면 된다는 결론을 얻을 수 있습니다. 하지만 지금까지 가이드를 따라오면서, 관리 트리의 외부에서 프로세스를 생성하는 것은 가능한 피하라고 배워왔습니다. 그러니 다른 방법을 찾아보죠.

지금 상황에서 `Node.spawn_link/2`보다 좋은 대안은 3가지입니다.

1. Erlang의 [:rpc](http://www.erlang.org/doc/man/rpc.html) 모듈을 사용하여 원격 노드에서 함수를 실행합니다. `bar@computer-name` 셸에서 `:rpc.call(:"foo@computer-name", Hello, :world, [])`를 실행하면 "hello world"를 출력할 것입니다.

2. [GenServer](http://elixir-lang.org/docs/stable/elixir/GenServer.html) API를 통해서 원격 노드로 요청을 전송하는 서버를 만들 수 있습니다. 예를 들어 `GemServer.call({name, node}, arg)`를 사용해 원격 서버에 요청을 보내거나, 아니면 원격 프로세스의 PID를 첫 번째 인수로 넘겨줄 수 있습니다.

3. [이전 장](http://elixir-lang.org/getting-started/mix-otp/task-and-gen-tcp.html)에서 배웠던 [Task](http://elixir-lang.org/docs/stable/elixir/Task.html)는 로컬/원격 모두에서 프로세스를 생성할 수 있으므로 이를 사용할 수도 있습니다.

각각의 선택지는 속성이 다릅니다. `:rpc`나 GenServer는 요청을 한 서버에서 직렬화하는 반면 Task는 슈퍼바이저에서 생성될 때 직렬화가 되고, 효율적으로 원격 노드에서 비동기로 실행됩니다.

여기에서는 라우팅 계층을 위해서 Task를 사용할 것입니다만, 궁금하다면 다른 방법에 대해서도 알아보세요!

## async/await

고립되어 동작되는 태스크에 대해서 알아보았습니다만, 지금까지는 반환 값에 대해서 고민하지 않았습니다. 하지만 때때로 태스크에서 어떤 값을 계산하고 나중에 그 결과를 읽어오는 것이 유용할 때가 있습니다. 태스크는 이런 경우를 위해 `async/await` 패턴을 제공합니다.

```elixir
task = Task.async(fn -> compute_something_expensive end)
res  = compute_something_else()
res + Task.await(task)
```

`async/await`는 동시에 값을 계산할 수 있는 무척 간단한 구조를 제공합니다. 그 뿐만이 아니라 `async/await`는 이전 장에서 공부했던 [`Task.Supervisor`](http://elixir-lang.org/docs/stable/elixir/Task.Supervisor.html)도 사용할 수 있습니다. 그저 `Task.Supervisor.start_child/2` 대신에 `Task.Supervisor.async/2`를 호출하고 `Task.await/2`를 사용하여 결과를 읽어오면 됩니다.

## 분산된 태스크

분산된 태스크는 관리되는 태스크들과 같습니다. 차이가 있다면 슈퍼바이저에서 태스크를 생성할 때 노드 이름을 넘겨준다는 점이 있습니다. `kv` 애플리케이션의 `lib/kv/supervisor.ex`를 열어서 태스크 슈퍼바이저를 관리 트리의 마지막에 추가해보죠.

```elixir
supervisor(Task.Supervisor, [[name: KV.RouterTasks]]),
```

이제 다시 이름을 가지는 노드를 2개 실행하되, `:kv` 애플리케이션에서 실행해주세요.

```bash
$ iex --sname foo -S mix
$ iex --sname bar -S mix
```

`bar@computer-name`에서는 슈퍼바이저를 통해 다른 노드에 곧장 태스크를 생성할 수 있습니다.

```iex
iex> task = Task.Supervisor.async {KV.RouterTasks, :"foo@computer-name"}, fn ->
...>   {:ok, node()}
...> end
%Task{pid: #PID<12467.88.0>, ref: #Reference<0.0.0.400>}
iex> Task.await(task)
{:ok, :"foo@computer-name"}
```

첫 번째 분산 태스크는 단순하게 현재 실행 중인 노드의 이름을 가져오는 코드입니다. 여기에서는 `Task.Supervisor.async/2`에 익명 함수를 넘겼습니다만, 분산된 환경에서는 모듈, 함수, 인수 등은 명확히 제공하는 것이 좋습니다.

```iex
iex> task = Task.Supervisor.async {KV.RouterTasks, :"foo@computer-name"}, Kernel, :node, []
%Task{pid: #PID<12467.88.0>, ref: #Reference<0.0.0.400>}
iex> Task.await(task)
:"foo@computer-name"
```

익명 함수를 사용하는 경우의 다른 점은 호출한 노드와 같은 버전을 사용할 것을 요구한다는 점입니다. 모듈, 함수와 인수를 명시적으로 넘겨주면 주어진 모듈에서 알맞은 함수를 찾아서 실행하기만 하면 되기 때문에 좀 더 안정적으로 동작시킬 수 있습니다.

이 부분을 이해하셨으면, 이제 마지막 코드를 작성해봅시다.

## 라우팅 계층

`lib/kv/router.ex` 파일을 만들고 다음을 추가하세요:

```elixir
defmodule KV.Router do
  @doc """
  Dispatch the given `mod`, `fun`, `args` request
  to the appropriate node based on the `bucket`.
  """
  def route(bucket, mod, fun, args) do
    # Get the first byte of the binary
    first = :binary.first(bucket)

    # Try to find an entry in the table or raise
    entry =
      Enum.find(table, fn {enum, _node} ->
        first in enum
      end) || no_entry_error(bucket)

    # If the entry node is the current node
    if elem(entry, 1) == node() do
      apply(mod, fun, args)
    else
      {KV.RouterTasks, elem(entry, 1)}
      |> Task.Supervisor.async(KV.Router, :route, [bucket, mod, fun, args])
      |> Task.await()
    end
  end

  defp no_entry_error(bucket) do
    raise "could not find entry for #{inspect bucket} in table #{inspect table}"
  end

  @doc """
  The routing table.
  """
  def table do
    # Replace computer-name with your local machine name.
    [{?a..?m, :"foo@computer-name"},
     {?n..?z, :"bar@computer-name"}]
  end
end
```

이 라우터가 정상적으로 동작하는지 확인하기 위한 테스트를 작성해보죠. `test/kv/router_test.exs` 파일을 생성하고 다음을 추가하세요:

```elixir
defmodule KV.RouterTest do
  use ExUnit.Case, async: true

  test "route requests across nodes" do
    assert KV.Router.route("hello", Kernel, :node, []) ==
           :"foo@computer-name"
    assert KV.Router.route("world", Kernel, :node, []) ==
           :"bar@computer-name"
  end

  test "raises on unknown entries" do
    assert_raise RuntimeError, ~r/could not find entry/, fn ->
      KV.Router.route(<<0>>, Kernel, :node, [])
    end
  end
end
```

첫 번째 테스트에서는 단순하게 버킷 이름인 "hello"와 "world"를 기반으로 `Kernel.node/0`을 호출하여 현재 노드의 이름을 돌려받고 있습니다. 라우팅 테이블대로라면 `foo@computer-name`와 `bar@computer-name`를 각각 응답으로 받을 수 있을 것입니다.

두 번째 테스트는 모르는 버킷 이름에 대해서 에러가 발생하는지를 확인합니다.

첫 번째 테스트를 실행하기 위해서는 2개의 노드가 실행 중이어야 합니다. `apps/kv`로 돌아가서 `bar`라는 이름의 노드를 테스트 환경에서 사용할 수 있도록 재시작합시다.

```bash
$ iex --sname bar -S mix
```

이번에는 다음과 같이 실행하세요:

```bash
$ elixir --sname foo -S mix test
```

테스트는 성공적으로 통과할 것입니다. 훌륭합니다!

## 필터와 태그 테스트하기

테스트가 통과했지만, 테스트 구조가 점점 복잡해지고 있습니다. 특히 이제 몇몇 테스트가 다른 노드와의 연결을 요구하기 때문에 `mix test`만을 실행하면 테스트 스위트가 실패한다는 점이 그렇습니다.

운 좋게도, ExUnit은 테스트를 태깅하는 기능을 지원하여, 이를 통해 특정 콜백을 실행하거나, 실행하지 않게끔 제외할 수 있습니다. 이미 저번 장에서 ExUnit에서 이미 정의된 `:capture_log`라는 태그를 사용한 바가 있습니다.

이번에는 `:distributed` 태그를 `test/kv/router_test.exs`에 추가해봅시다:

```elixir
@tag :distributed
test "route requests across nodes" do
```

`@tag :distributed`라고 적는 것은 `@tag distributed: true`라고 적는 것과 같은 의미입니다.

올바르게 태깅된 테스트가 있으면 이제 네트워크상에 노드가 살아 있는지 확인하고, 그렇지 않다면 모든 분산 테스트들을 실행하지 않을 수 있습니다. `kv` 애플리케이션의 `test/test_helper.exs`를 열고 다음을 추가하세요:

```elixir
exclude =
  if Node.alive?, do: [], else: [distributed: true]

ExUnit.start(exclude: exclude)
```

이제 `mix test`로 테스트를 실행해보세요:

```bash
$ mix test
Excluding tags: [distributed: true]

.......

Finished in 0.1 seconds (0.1s on load, 0.01s on tests)
7 tests, 0 failures
```

이번에는 모든 테스트가 통과했으며 ExUnit은 사용자에게 분산 테스트는 실행하지 않았음을 경고합니다. 만약 `$ elixir --sname foo -S mix test`를 사용해서 테스트를 실행했다면 `bar@computer-name` 노드가 살아있다는 전제로, 나머지 한 테스트가 성공적으로 통과할 것입니다.

`mix test` 명령은 태그를 동적으로 포함하거나 제외할 수도 있습니다. 예를 들어 `$ mix test --include distributed`라고 명령해서 `test/test_helper.exs`에 설정된 값과 관계없이 분산 테스트를 실행하게끔 만들 수도 있습니다. 반대로 `--exclude` 옵션을 통해서 특정 태그를 제외할 수도 있습니다. 마지막으로 `--only`를 사용해서 특정 태그가 있는 테스트만을 실행할 수도 있습니다:

```bash
$ elixir --sname foo -S mix test --only distributed
```

필터, 태그 그리고 기본 태그에 대한 더 자세한 설명은 [`ExUnit.Case` 모듈 문서](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.Case.html)를 참고하세요.

## 애플리케이션 환경과 설정

여기에서는 라우팅 테이블을 `KV.Router` 모듈에 직접 작성하고 있었습니다. 하지만 이 테이블을 동적으로 만들고 싶습니다. 이는 개발/테스트/프로덕션 환경을 설정할 수 있게 해줄 뿐만 아니라, 다른 노드들이 라우팅 테이블의 다른 엔트리에서 동작할 수 있게끔 해줍니다. <abbr title="Open Telecom Platform">OTP</abbr>에는 이에 딱 적당한 것이 있습니다. 애플리케이션 환경이라는 것이죠.

각 애플리케이션은 키를 사용해서 애플리케이션 설정 정보를 가지는 환경을 가집니다. 예를 들어 `:kv` 애플리케이션 환경에 라우팅 테이블을 저장하여, 이를 기본값으로 설정하고, 다른 애플리케이션에서 필요하다면 그 테이블을 변경할 수 있습니다.

`apps/kv/mix.exs`를 열고 `application/0` 함수를 다음과 같이 변경합니다:

```elixir
def application do
  [applications: [],
   env: [routing_table: []],
   mod: {KV, []}]
end
```

애플리케이션에 새 `:env` 키를 추가했습니다. 이는 `:routing_table`이라는 키와 빈 리스트를 가지는 애플리케이션 기본 환경을 반환합니다. 실제 라우팅 테이블은 테스팅/배포 환경에 따라 달라지기 때문에 애플리케이션 환경에서는 빈 테이블을 제공하는 것이 자연스럽습니다.

코드에서 애플리케이션 환경을 사용하려면 `KV.Router.table/0`을 다음으로 변경만 하세요:

```elixir
@doc """
The routing table.
"""
def table do
  Application.fetch_env!(:kv, :routing_table)
end
```

`Application.fetch_env!/2`를 사용하여 `:kv`의 환경에 있는 `:routing_table`을 가져왔습니다. 애플리케이션의 환경에 대한 더 자세한 정보나 조작하는 다른 함수들에 대해서는 [Application 모듈](http://elixir-lang.org/docs/stable/elixir/Application.html)을 참고하세요.

이제 라우팅 테이블에는 아무런 정보도 들어있지 않기 때문에, 분산 테스트는 분명 실패할 것입니다. 앱을 재실행하고 테스트도 재실행해서 다음 실패를 확인하세요:

```bash
$ iex --sname bar -S mix
$ elixir --sname foo -S mix test --only distributed
```

애플리케이션 환경에 대한 재미있는 점은, 현재 애플리케이션을 위한 설정뿐만 아니라, 모든 애플리케이션을 위한 설정도 가능하다는 점입니다. 그런 설정은 `config/config.exs`에서 처리합니다. 예를 들어서 IEx의 기본 프롬프트를 다른 값으로 변경하고 싶다고 하죠. `apps/kv/config/config.exs` 파일을 열고 다음을 추가하세요:

```elixir
config :iex, default_prompt: ">>>"
```

`iex -S mix`로 IEx를 시작하면 IEx의 프롬프트가 변경된 것을 확인할 수 있습니다.

마찬가지로 `apps/kv/config/config.exs`에서 `:routing_table`을 직접 설정할 수 있다는 의미가 됩니다:

```elixir
# Replace computer-name with your local machine nodes.
config :kv, :routing_table,
       [{?a..?m, :"foo@computer-name"},
        {?n..?z, :"bar@computer-name"}]
```

노드들을 재시작하고 분산 테스트를 다시 실행하세요. 그러면 이제 통과할 것입니다.

Elixir v1.2부터 모든 엄브렐라 애플리케이션은 서로의 설정을 공유하게 되었으므로, 엄브렐라 프로젝트의 최상위 `config/config.exs`에서는 각각 자식들의 설정들을 불러오게 됩니다:

```elixir
import_config "../apps/*/config/config.exs"
```

`mix run` 명령은 `--config` 옵션을 받으며, 이를 통해서 어떤 설정 파일을 사용할지 지정할 수 있습니다. 다른 노드를 시작할 때에 그 노드만을 위한 설정을 불러오기 위해서 사용되는 경우가 많습니다(예를 들어, 다른 라우팅 테이블이라든가).

전체적으로, 애플리케이션을 설정하기 위한 내장 기능을 사용했다는 점과 애플리케이션을 엄브렐라 애플리케이션으로 만들었다는 점은 배포를 위한 다양한 방법을 제공해줍니다.

* 엄브렐라 애플리케이션을 노드에 배포하여 TCP 서버와 키-값 저장소로서 동작하게끔 배포하기

* `:kv_server` 애플리케이션을 TCP 서버로서 다른 노드들에 대해 라우팅만을 하도록 배포하기

* 저장소 기능만을 사용하기 위해서 `:kv` 애플리케이션만 배포하기

미래에는 더 많은 애플리케이션을 추가하며, 계속해서 배포를 잘 나누어진 상태로 유지하여 특정 부분만을 특정 설정과 함께 배포할 수 있습니다.

또는 선택한 애플리케이션과 설정을 패키징해주는 [exrm](https://github.com/bitwalker/exrm) 같은 도구를 사용하여 다양한 릴리스를 만드는 것도 고려해볼 수 있습니다. 이를 사용하면 현재 Erlang과 Elixir 설치를 포함하여 패키지를 만들어 주기 때문에, 배포 환경에 런타임이 설치되어 있지 않더라도 배포를 할 수 있습니다.

마지막으로 이 장에서는 새로운 것들을 배웠으며 이들을 `:kv_server` 애플리케이션에서 사용했습니다. 다음 단계들은 연습문제 삼아서 남겨두도록 하겠습니다:

* `:kv_server` 애플리케이션이 4040번 포트를 사용하는 대신 애플리케이션 환경으로부터 사용할 포트 번호를 가져오기.

* `:kv_server` 애플리케이션을 변경하고 설정을 추가하여 현재 노드의 `KV.Registry`에서 직접 가져오는 대신 라우팅 기능을 사용하도록 만들어보기. `:kv_server` 테스트를 위해서 현재 노드를 가리키는 라우팅 테이블을 사용할 수도 있을 겁니다.

## 정리하기

이 장에서는 Elixir와 Erlang <abbr title="Virtual Machine">VM</abbr>의 분산 기능을 알아보기 위한 방법으로 간단한 라우터를 만들어 보았으며, 어떻게 라우팅 테이블을 설정하는지에 대해서 배웠습니다. 이 장은 Mix와 <abbr title="Open Telecom Platform">OTP</abbr> 가이드의 마지막 장입니다.

이 가이드를 통해서, 매우 간단한 분산 키-값 저장소를 만들며 GenServer, 슈퍼바이저, 태스크, 에이전트, 애플리케이션 등의 구조들을 배웠습니다. 그뿐만이 아니라 애플리케이션 전체를 확인하기 위한 테스트도 작성하며 ExUnit의 사용법도 알아보았으며, 더 많은 작업을 처리하기 위한 Mix 빌드 도구의 사용법도 알아보았습니다.

만약 분산된 키-값 저장소를 실제 환경에서 사용하고 싶다면, 반드시 Erlang <abbr title="Virtual Machine">VM</abbr> 상에서 동작하는 [Riak](http://basho.com/riak/)에 대해서 알아보기를 권장합니다. Riak에서는 버킷들을 복제하여 데이터 분실을 최소화하며, 라우터 대신에 [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing)을 사용하여 버킷과 노드를 사상하고 있습니다. 이 알고리즘은 인프라에 버킷을 저장하기 위한 새 노드를 추가할 때에 필요한 데이터 마이그레이션 작업을 줄여줍니다.

아직 배워야 할 많은 레슨이 있으며, 앞으로도 재미있길 바랍니다!

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Distributed tasks and configuration](http://elixir-lang.org/getting-started/mix-otp/distributed-tasks-and-configuration.html)
