---
layout: post
title: "Elixir - OTP: GenServer"
date: 2016-05-20 00:01:28 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 거의 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 많습니다. 원문은 최하단의 링크를 참고하세요.

## OTP: GenServer

[이전 챕터](http://elixir-lang.org/getting-started/mix-otp/agent.html)에서는 키-값을 저장하기 위해서 에이전트를 사용했습니다. 첫번째 챕터에서 각각의 저장 공간을 다음과 같이 이름을 붙여서 사용하는 모습을 보였습니다:

```elixir
CREATE shopping
OK

PUT shopping milk 1
OK

GET shopping milk
1
OK
```

에이전트는 프로세스이기 때문에, 각각의 저장 공간은 프로세스 식별자(pid)를 가지고 있습니다만, 그것은 이름이 아닙니다. [이전 챕터](http://elixir-lang.org/getting-started/processes.html)에서 이름 등록에 대해서 배웠으며, 그러한 방법을 통해서 문제를 해결하고 싶을지도 모릅니다. 예를 들어, 다음과 같이 저장소를 만들어서 말이죠:

```iex
iex> Agent.start_link(fn -> %{} end, name: :shopping)
{:ok, #PID<0.43.0>}
iex> KV.Bucket.put(:shopping, "milk", 1)
:ok
iex> KV.Bucket.get(:shopping, "milk")
1
```

하지만 이것은 무척 끔찍한 발상입니다! 프로세스의 이름은 반드시 아톰이어야 하며, 이말은 (보통 외부의 클라이언트로부터 수신하는) 저장소의 이름을 아톰으로 변환할 필요가 있지만, **사용자의 입력을 아톰으로 변환해서는 안됩니다**. 왜냐하면 아톰은 가비지 컬렉션에 의해서 정리되지 않기 때문입니다. 아톰은 한번 생성되면 다시는 삭제되지 않습니다. 사용자의 입력으로부터 아톰을 생성하는 것은, 사용자가 충분한 양의 다른 이름을 사용해서 시스템 메모리를 다 쓰게끔 만들 수 있다는 의미입니다.

일반적으로는 메모리를 전부 소비하고 시스템이 다운되기 전에 Erlang <abbr title="Virtual Machine">VM</abbr>의 아톰 생성 제한에 걸리게 될 것입니다.

이렇게 이름 등록을 남용하는 대신, 저장 공간의 이름과 저장 공간의 프로세스에 대한 정보를 가지고 있는 맵을 통해 우리만의 *등록 프로세스*를 만들어보죠.

이름 저장소(registry)는 사전이 언제나 최신임을 보장할 필요가 있습니다. 예를 들어, 저장 공간 프로세스가 하하나라도 버그 때문에 동작을 멈추게 된다면, 이름 저장소는 잘못된 정보를 제공하지 않기 위해서 사전을 정리할 필요가 있습니다. 이를 Elixir에서는 이름 저장소가 각 저장소를 감시해야한다고 표현하죠.

우리는 [GenServer](http://elixir-lang.org/docs/stable/elixir/GenServer.html)를 사용하여 등록 프로세스를 생성하고 각 저장 공간 프로세스를 감시하도록 만들 겁니다. GenServer는 Elixir와 <abbr title="Open Telecom Platform">OTP</abbr>위에서 일반적인 서버를 구현하기 위한 추상화 계층입니다.

## 첫번째 GenServer

GenServer는 두 부분으로 구현됩니다: 클라이언트 API와 서버 콜백은 각각 하나의 모듈에서 구현하거나, 서로 다른 모듈에서 구현할 수 있습니다. 클라이언트와 서버는 서로 다른 프로세스에서 동작하며, 클라이언트가 함수를 통해 호출하면 서버와 메시지를 주고 받을 수 있습니다. 여기에서는 하나의 모듈을 사용해서 서버 콜백과 클라이언트 API를 함께 만들어보죠. `lib/kv/registry.ex`에 새 파일을 만들고 다음을 추가하세요:

```elixir
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry.
  """
  def start_link do
    GenServer.start_link(__MODULE__, :ok, [])
  end

  @doc """
  Looks up the bucket pid for `name` stored in `server`.

  Returns `{:ok, pid}` if the bucket exists, `:error` otherwise.
  """
  def lookup(server, name) do
    GenServer.call(server, {:lookup, name})
  end

  @doc """
  Ensures there is a bucket associated to the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  ## Server Callbacks

  def init(:ok) do
    {:ok, %{}}
  end

  def handle_call({:lookup, name}, _from, names) do
    {:reply, Map.fetch(names, name), names}
  end

  def handle_cast({:create, name}, names) do
    if Map.has_key?(names, name) do
      {:noreply, names}
    else
      {:ok, bucket} = KV.Bucket.start_link
      {:noreply, Map.put(names, name, bucket)}
    end
  end
end
```

첫번째 함수는 `start_link/0`이며, 이 함수는 새로운 GenServer에 3개의 인수를 넘깁니다:

1. 서버 콜백이 구현된 모듈입니다. 이 경우에는 현재 모듈이므로 `__MODULE__`를 사용합니다.

2. 초기화에 사용하는 인수이며, 이 경우에는 아톰 `:ok`를 사용합니다.

3. 옵션의 리스트입니다. 예를 들자면 서버의 이름 같은 걸 포함합니다. 여기에서는 빈 리스트를 넘깁니다.

GenServer에 보낼 수 있는 요청은 두 가지가 있습니다: 요청(call)과 캐스트(cast)로, 전자는 동기적으로 동작하며, 서버는 **반드시** 그 요청에 대한 응답을 돌려주어야 합니다. 캐스트는 비동기적이며, 서버는 응답을 돌려주지 않습니다.

그 다음에 있는 두 함수, `lookup/2`와 `create/2`는 서버에 요청을 보내는 책임을 가집니다. 그 요청들은 `handle_call/3`나 `handle_cast/2`의 첫번째 인자의 형태로 표현됩니다. 여기에서는 각각 `{:lookup, name}`과 `{:create, name}`로 표현하고 있습니다. 요청들은 하나의 인자에 하나보다 많은 정보를 제공하기 위해서 여기에서처럼 튜플의 형태로 사용되는 경우가 많습니다. 튜플의 첫번째 원소로서 어떤 동작인지를 기술하고, 나머지 원소들을 통해서 실제 필요한 정보들을 기술하는 것은 꽤 일반적입니다.

서버 쪽에서는 서버의 초기화, 종료, 요청을 처리하기 위한 다양한 콜백들을 구현할 수 있습니다. 이러한 콜백들을 반드시 구현해야하는 것은 아니며, 지금은 우리가 필요한 것들만 구현합시다.

첫번째는 `init/1` 콜백으로, 이는 `GenServer.start_link/3`가 받은 인자들을 넘겨 받고, `{:ok, state}`를 반환합니다. 이 때 상태는 새로운 맵이 됩니다. 이제 슬슬 `GenServer`의 API가 클라이언트와 서버를 어떻게 명확히 분리하는지 눈치챌 수 있을 것입니다. `start_link/3`는 클라이언트에서 실행되는 반면, `init/1`는 서버에서 동작합니다.

`call/2` 요청을 위해서 `request`를 받는 `handle_call/3`은 요청이 어디에서 왔는지(`_from`), 그리고 현재 서버의 상태(`names`)가 어떤지에 대한 정보를 넘겨 받습니다. `handle_call/3`는 `{:reply, reply, new_state}` 형태의 튜플을 반환하며, `reply`는 클라이언트로 보내질 정보, `new_state`는 서버의 새 상태입니다.

`cast/2` 요청을 위해서, `request`와 현재 서버의 상태(`names`)를 받는 `handle_cast/2`를 구현해야합니다. `handle_cast/2` 콜백은 `{:noreply, new_state}` 형태의 튜플을 반환합니다.

이외에도 `handle_call/3`와 `handle_cast/2` 콜백이 반환할 수 있는 다른 형태도 있습니다. 그리고 `terminate/2`나 `code_change/3`와 같은 콜백도 존재합니다. 이들에 대한 정보는 [GenServer 문서](http://elixir-lang.org/docs/stable/elixir/GenServer.html)를 통해서 확인해 보세요.

이제 우리의 GenServer가 기대하는 대로 동작하는지 보장하기 위한 테스트를 작성해봅시다.

## GenServer 테스트하기

Genserver를 테스트하는 것은 에이전트를 테스트하는 것과 크게 다르지 않습니다. 서버 프로세스를 setup 시점에서 생성하고, 테스트에서 사용할 겁니다. `test/kv/registry_test.exs` 파일을 만들고 다음을 추가합시다:

```elixir
defmodule KV.RegistryTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, registry} = KV.Registry.start_link
    {:ok, registry: registry}
  end

  test "spawns buckets", %{registry: registry} do
    assert KV.Registry.lookup(registry, "shopping") == :error

    KV.Registry.create(registry, "shopping")
    assert {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

    KV.Bucket.put(bucket, "milk", 1)
    assert KV.Bucket.get(bucket, "milk") == 1
  end
end
```

테스트는 반드시 성공해야 합니다!

테트스가 종료되는 시점에서 저장소가 `:shutdown`신호를 받기 때문에 명시적으로 이를 종료시킬 필요가 없습니다. 테스트에서는 이 해결책이 유효합니다만, 실제 애플리케이션에서는 `GenServer`를 종료시켜야할 필요가 있을지도 모릅니다. 예를 들어, `GenServer.stop/1` 같은 함수를 사용해서 말이죠:

```elixir
  ## Client API

  @doc """
  Stops the registry.
  """
  def stop(server) do
    GenServer.stop(server)
  end
```

## 모니터링의 필요성

이름 저장소는 이제 거의 완성되었습니다. 마지막으로 남은 문제는, 키-값 저장소들이 멈추거나 문제를 일으켰을때 입니다. 이러한 버그를 재현하는 테스트를 `KV.RegistryTest`에 추가해봅시다:

```elixir
  test "removes buckets on exit", %{registry: registry} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(registry, "shopping")
    Agent.stop(bucket)
    assert KV.Registry.lookup(registry, "shopping") == :error
  end
```

이 테스트는 마지막 부분에서 키-값 저장소가 종료되었음에도 불구하고 그 이름이 이름 저장소에 남아있기 때문에 실패합니다.

이 버그를 고치기 위해서, 이름 저장소가 모든 키-값 저장소를 감시할 필요가 있습니다. 일단 감시하기 시작하면, 이름 저장소는 키-값 저장소가 종료되는 시점에 알림을 받으며, 이를 통해서 이름 사전을 정리할 수 있습니다.

그럼 `iex -S mix`로 새 콘솔을 열어 모니터를 좀 확인해보죠:

```iex
iex> {:ok, pid} = KV.Bucket.start_link
{:ok, #PID<0.66.0>}
iex> Process.monitor(pid)
#Reference<0.0.0.551>
iex> Agent.stop(pid)
:ok
iex> flush
{:DOWN, #Reference<0.0.0.551>, :process, #PID<0.66.0>, :normal}
```

`Process.monitor(pid)`가 넘어오는 메시지를 모니터의 것이라고 확인하기 위한 유일한 레퍼런스를 반환한다는 점을 기억하세요. 에이전트를 멈추고 나서, `flush/0`로 모든 메시지를 내보내면 방금 모니터로부터 반환받은 바로 그 레퍼런스와 함께 `:DOWN` 메시지가 온 것을 확인할 수 있으며, 에이전트가 `:normal`이라는 이유로 종료되었음을 알 수 있습니다.

그럼 이제 다시 서버의 콜백을 고쳐서 이 문제를 해결해 봅시다. 우선, GenServer의 상태를 다음의 두 사전으로 나눌 것입니다: 하나는 `name -> pid`이며, 나머지 하나는 `ref -> name`입니다. 그리고 `handle_cast/2`를 통해 생성된 키-값 저장소를 감시하게끔 만들고, `handle_info/2` 콟맥을 구현해서 관련된 메시지를 처리할 수 있도록 합시다. 완전한 서버 콜백 구현은 다음과 같습니다:

```elixir
  ## Server callbacks

  def init(:ok) do
    names = %{}
    refs  = %{}
    {:ok, {names, refs}}
  end

  def handle_call({:lookup, name}, _from, {names, _} = state) do
    {:reply, Map.fetch(names, name), state}
  end

  def handle_cast({:create, name}, {names, refs}) do
    if Map.has_key?(names, name) do
      {:noreply, {names, refs}}
    else
      {:ok, pid} = KV.Bucket.start_link
      ref = Process.monitor(pid)
      refs = Map.put(refs, ref, name)
      names = Map.put(names, name, pid)
      {:noreply, {names, refs}}
    end
  end

  def handle_info({:DOWN, ref, :process, _pid, _reason}, {names, refs}) do
    {name, refs} = Map.pop(refs, ref)
    names = Map.delete(names, name)
    {:noreply, {names, refs}}
  end

  def handle_info(_msg, state) do
    {:noreply, state}
  end
```

우리는 클라이언트 API를 전혀 손대지 않고 서버의 구현만을 변경하여 문제를 해결했습니다. 이는 서버와 클라이언트를 명시적으로 분리하여 얻을 수 있는 장점 중 하나입니다.

마지막으로 다른 콜백들과 다르게, 우리는 `handle_info/2`에서 모르는 메시지를 처리하기 위해서 "catch-all" 절을 사용했습니다. 왜 이렇게 되는지를 이해하려면 다음 장으로 넘어갑시다.

## `call`, `cast` 아니면 `info`?

지금까지 우리는 3개의 콜백을 사용했습니다: `handle_call/3`, `handle_cast/2`, `handle_info/2`. 각각에 대해서 언제 사용해야하는지 알아봅시다:

1. `handle_call/3`은 동기적인 요청에 대해서 사용되어야 합니다. 이것은 서버의 응답을 기다려서 처리하는 경우, 일반적인 선택지입니다.

2. `handle_cast/2`는 응답에 대해서 고려하지 않는 경우, 다시 말해 비동기적인 요청에 대해서 사용해야 합니다. 캐스트는 서버가 응답을 받았는지에 대한 보장을 하지 않으며, 이러한 이유로 자주 사용해서는 안됩니다. 예를 들어 이 챕터에서 구현했던 `create/2`은 본래 `call/2`를 사용해야합니다. 여기에서는 `cast/2`가 교육적인 목적으로 사용되었습니다.

3. `handle_info/2`는 `send/2`를 포함한 `GenServer.call/2`나 `GenServer.cast/2`가 아닌 곳으로부터 오는 모든 메시지에 대해서 사용됩니다. 모니터링에서 보내지는 `:DOWN` 메시지가 이에 대한 완벽한 예시입니다.

`send/2`를 포함한 모든 메시지는 `handle_info/2`에서 처리되며, 때때로 서버에 기대되지 않은 메시지가 도착하는 경우도 있습니다. 그러므로 만약 모든 메시지를 처리하는 부분을 만들지 않으면 처리할 수 없는 메시지들은 우리의 이름 저장소를 망가뜨릴 것입니다.

`handle_call/3`나 `handle_cast/2`에서는 이러한 문제를 고민할 필요가 없습니다. 왜냐하면 해당 함수들은 `GenServer`의 API를 통해서만 처리되기 때문에, 처리할 수 없는 메시지는 대부분 개발자의 실수인 경우가 많습니다.

## 모니터링? 링크?

우리는 [Process 챕터](http://elixir-lang.org/getting-started/processes.html)에서 링크에 대해서 배웠습니다. 이제 이름 저장소가 완성된 지금, 한가지 의문이 생겼을지도 모릅니다: 언제 모니터링을 쓰고, 언제 링크를 사용해야할까요?

링크는 양방향입니다. 만약 두 프로세스를 링크한다면 하나가 고장났을때, 다른 한쪽이 함께 멈춥니다(트랩을 사용하는 경우는 제외합시다). 반면 모니터는 단방향입니다: 모니터링 프로세스만이 감시하고 있는 프로세스에 대한 정보를 받을 수 있습니다. 간단하게 정리하자면, 연결된 프로세스를 함께 죽이고 싶다면 링크를, 그렇지 않고 문제를 확인하기만 원한다면 모니터를 사용하세요.

이제 `handle_cast/2` 구현으로 돌아가보면, 링크와 모니터링을 함께 사용하고 있는 것을 볼 수 있습니다:

```elixir
{:ok, pid} = KV.Bucket.start_link
ref = Process.monitor(pid)
```

우리는 키-값 저장소가 문제가 생겼을때 이름 저장소까지 동작을 멈추기를 바라지 않으므로, 이것은 좋지 않은 방법입니다. 일반적으로 새 프로세스를 직접 생성하는 대신, 이 작업을 슈퍼바이저에게 위임합니다. 이를 다음 챕터에서 알아볼 것이며, 슈퍼바이저는 링크에 의존하고 있으며, 이는 왜 Elixir와 <abbr title="Open Telecom Platform">OTP</abbr>에서 링크 기반 API(`spawn_link`, `start_link`, etc)가 널리 퍼져있는지 설명해줄 것입니다.

## Reference
 * [Elixir Homepage](http://elixir-lang.org)
 * [Elixir - OTP: GenServer](http://elixir-lang.org/getting-started/mix-otp/genserver.html)
