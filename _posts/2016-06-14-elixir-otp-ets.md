---
layout: post
title: "Elixir - OTP: ETS"
date: 2016-06-14 20:44:45 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 거의 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 많습니다. 원문은 최하단의 링크를 참고하세요.

## Introduction

지금대로라면 버킷을 검색해야 할 때마다, 레지스트리에 메시지를 보내야 합니다. 레지스트리는 여러 프로세스에서 동시에 접근할 수 있으므로, 점점 병목 지점이 될 것입니다!

이 장에서는 ETS(Erlang Term Storage)에 대해서 배우고 어떻게 캐싱에 활용하는지를 살펴보겠습니다.

> 주의하세요! ETS를 너무 빨리 캐시로 도입하지 마세요. 애플리케이션을 로깅하고 분석하여 어느 부분이 병목을 발생시키는지 확인하여 캐싱을 해야하는지, 무엇을 캐싱해야하는지를 알아야합니다. 이 장에서는 이미 그 필요성을 인정하고, 어떻게 ETS를 사용하는 지에 대한 예제입니다.

## 캐시로서의 ETS

ETS는 메모리 상의 테이블에 어떤 Elixir의 구조라도 저장할 수 있게 해줍니다. ETS 테이블은 [Erlang의 `:ets` 모듈](http://www.erlang.org/doc/man/ets.html)을 통해서 동작합니다:

```iex
iex> table = :ets.new(:buckets_registry, [:set, :protected])
8207
iex> :ets.insert(table, {"foo", self})
true
iex> :ets.lookup(table, "foo")
[{"foo", #PID<0.41.0>}]
```

ETS 테이블을 생성할 때에는 2개의 인수가 필요합니다: 테이블의 이름과 옵션의 리스트인데요. 사용가능한 옵션으로는 테이블의 형식과 접근 규칙 등이 있습니다. 여기에서는 `:set` 형식을 선택했으며, 이는 중복키를 허용하지 않겠다는 의미입니다. 그리고 테이블의 접근 권한을 `:protected`로 지정했으며, 이는 테이블을 생성한 프로세스만이 값을 추가할 수 있으며, 그 이외의 프로세스들은 읽기만 가능합니다. 마지막으로 이 옵션들은 기본값이므로 생략할 수도 있습니다.

ETS 테이블에 이름을 지어줄 수도 있으며, 이를 통해서 접근할 수 있습니다:

```iex
iex> :ets.new(:buckets_registry, [:named_table])
:buckets_registry
iex> :ets.insert(:buckets_registry, {"foo", self})
true
iex> :ets.lookup(:buckets_registry, "foo")
[{"foo", #PID<0.41.0>}]
```

`KV.Registry`가 ETS 테이블을 사용하도록 변경해봅시다. 레지스트리는 이름을 인수로 요구하므로, ETS 테이블에게 레지스트리와 동일한 이름을 지어주도록 합시다. ETS의 이름과 프로세스의 이름은 다른 장소에 저장되므로, 동일한 이름으로 충돌이 발생할 가능성은 없습니다.

`lib/kv/registry.ex` 파일을 열고 다음과 같이 변경하세요. 변경점에 대해서 주석으로 설명을 추가해둔 상태입니다:

```elixir
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry with the given `name`.
  """
  def start_link(name) do
    # 1. Pass the name to GenServer's init
    GenServer.start_link(__MODULE__, name, name: name)
  end

  @doc """
  Looks up the bucket pid for `name` stored in `server`.

  Returns `{:ok, pid}` if the bucket exists, `:error` otherwise.
  """
  def lookup(server, name) when is_atom(server) do
    # 2. Lookup is now done directly in ETS, without accessing the server
    case :ets.lookup(server, name) do
      [{^name, bucket}] -> {:ok, bucket}
      [] -> :error
    end
  end

  @doc """
  Ensures there is a bucket associated to the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  @doc """
  Stops the registry.
  """
  def stop(server) do
    GenServer.stop(server)
  end

  ## Server callbacks

  def init(table) do
    # 3. We have replaced the names map by the ETS table
    names = :ets.new(table, [:named_table, read_concurrency: true])
    refs  = %{}
    {:ok, {names, refs}}
  end

  # 4. The previous handle_call callback for lookup was removed

  def handle_cast({:create, name}, {names, refs}) do
    # 5. Read and write to the ETS table instead of the map
    case lookup(names, name) do
      {:ok, _pid} ->
        {:noreply, {names, refs}}
      :error ->
        {:ok, pid} = KV.Bucket.Supervisor.start_bucket
        ref = Process.monitor(pid)
        refs = Map.put(refs, ref, name)
        :ets.insert(names, {name, pid})
        {:noreply, {names, refs}}
    end
  end

  def handle_info({:DOWN, ref, :process, _pid, _reason}, {names, refs}) do
    # 6. Delete from the ETS table instead of the map
    {name, refs} = Map.pop(refs, ref)
    :ets.delete(names, name)
    {:noreply, {names, refs}}
  end

  def handle_info(_msg, state) do
    {:noreply, state}
  end
end
```

`KV.Registry.lookup/2`를 변경하기 전에는 서버로 요청을 보냈었습니다만, 지금은 모든 프로세스에서 공유하고 있는 ETS 테이블에 직접 읽어오고 있습니다. 이것이 바로 지금 구현하고 있는 캐시 구조의 뒤에 있는 아이디어입니다.

이 캐싱이 정상적으로 동작할 수 있도록, ETS 테이블에 `:protected`(기본값) 권한으로 접근할 수 있어야하며, 그 결과 `KV.Registry`만 쓰기 권한을 가지고 있지만, 모든 클라이언트들은 값을 읽어올 수 있습니다. 이외에도 테이블을 시작할 때 `read_concurrency: true`를 넘겨주면 일반적인 동시 읽기 시나리오에 대한 최적화를 자동으로 수행해줍니다.

방금 위에서 변경한 내용으로 인해 테스트가 실패할 것입니다. 이전에는 레지스트리 프로세스의 pid를 사용했었으나, 이제 레지스트리의 탐색 동작은 ETS의 테이블 이름을 요구하기 때문입니다. 하지만 ETS 테이블은 레지스트리 프로세스와 동일한 이름을 가지고 있으므로, 이를 고치는 것은 어렵지 않습니다. `test/kv/registry_test.exs`의 `setup` 함수를 다음과 같이 변경하세요:

```elixir
  setup context do
    {:ok, _} = KV.Registry.start_link(context.test)
    {:ok, registry: context.test}
  end
```

`setup`를 변경하더라도 몇몇 테스트들은 여전히 실패할 것입니다. 혹은, 몇몇 테스트가 일관적으로 동작하지 않는 것을 발견했을 수도 있습니다. 예를 들어 "spawns buckets" 테스트에서는:

```elixir
  test "spawns buckets", %{registry: registry} do
    assert KV.Registry.lookup(registry, "shopping") == :error

    KV.Registry.create(registry, "shopping")
    assert {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

    KV.Bucket.put(bucket, "milk", 1)
    assert KV.Bucket.get(bucket, "milk") == 1
  end
```

는 다음 부분에서 실패합니다:

```elixir
{:ok, bucket} = KV.Registry.lookup(registry, "shopping")
```

어떻게 좀 전에 생성한 버킷에서 실패가 발생하는 것일까요?

이 실패는 -의도적인- 두 가지 실수 때문입니다:

  1. 이 캐싱 레이어를 추가해서 너무 조급하게 최적화를 했습니다.
  2. (`call/2`를 썼어야 함에도 불구하고) `cast/2`를 사용했습니다.

## 경합 상태?

Elixir로 개발한다고 해서 경합 상태로부터 자유로워지는 것이 아닙니다. 하지만 Elixir의 기본적으로 아무것도 공유되지 않는다는 간단한 추상화는 경합 상태의 원인을 특정하기 쉽도록 만들어 줍니다.

지금 테스트에서 문제가 되고 있는 것은 명령을 실행하는 것과 그에 따른 변경사항을 ETS 테이블에서 확인할 때까지의 차이를 확인할때까지의 딜레이입니다. 이 상황에서 우리가 원하는 동작은 다음과 같습니다:

1. `KV.Registry.create(registry, "shopping")`를 호출합니다.
2. 레지스트리가 버킷을 만들고 캐시 테이블을 갱신합니다.
3. `KV.Registry.lookup(registry, "shopping")`를 통해서 테이블에 접근합니다.
4. 그러면 이 명령이 `{:ok, bucket}`를 반환합니다.

하지만 `KV.Registry.create/2`는 cast 함수이므로, 이 함수는 실제로 우리가 테이블에 값을 쓰기 전에 반환됩니다! 다시 말하자면:

1. `KV.Registry.create(registry, "shopping")`를 호출합니다.
2. `KV.Registry.lookup(ets, "shopping")`로 테이블에 접근합니다.
3. 이 명령은 `:error`를 반환합니다.
4. 레지스트리는 버킷을 만들고 캐시 테이블을 갱신합니다.

이 문제를 해결하려면 `KV.Registry.create/2`에서 `cast/2` 대신에 `call/2`를 사용하여 동기적으로 동작하게끔 만들어야 합니다. 이는 변경사항이 테이블에 번영된 뒤에 클라이언트에서 원하는 후속작업을 할 수 있게끔 보장할 것입니다. 그러면 다음과 같이 변경해봅시다:

```elixir
  def create(server, name) do
    GenServer.call(server, {:create, name})
  end

  def handle_call({:create, name}, _from, {names, refs}) do
    case lookup(names, name) do
      {:ok, pid} ->
        {:reply, pid, {names, refs}}
      :error ->
        {:ok, pid} = KV.Bucket.Supervisor.start_bucket
        ref = Process.monitor(pid)
        refs = Map.put(refs, ref, name)
        :ets.insert(names, {name, pid})
        {:reply, pid, {names, refs}}
    end
  end
```

단순하게 `handle_cast/2`를 `handle_call/3`로 변경한 뒤, 생성한 버킷의 pid를 함께 반환하도록 만들었습니다. 일반적으로 Elixir 개발자들은 메시지에 대한 대답을 받을 때까지 실행을 지연시킬 수 있는 `call/2`을 `cast/2`보다 선호합니다. `cast/2`는 이른 최적화가 필요없는 경우에 사용하세요.

그럼 이제 테스트를 다시 실행해봅시다. 이번에는 `--trace` 옵션을 함께 넘겨보죠:

```bash
$ mix test --trace
```

`--trace` 옵션은 모든 테스트를 동기적으로 실행해주며(이 경우 `async: true`는 무시됩니다) 각 테스트에 대해서 자세한 정보를 제공해주기 때문에, 테스트가 데드락 또는 경합 상태에 빠졌을 경우에 유용합니다. 이번에는 하나나 두개의 테스트가 실패할 것입니다:

```
  1) test removes buckets on exit (KV.RegistryTest)
     test/kv/registry_test.exs:19
     Assertion with == failed
     code: KV.Registry.lookup(registry, "shopping") == :error
     lhs:  {:ok, #PID<0.109.0>}
     rhs:  :error
     stacktrace:
       test/kv/registry_test.exs:23
```

실패 메시지에 따르면 우리는 버킷이 테이블에 존재하지 않을 것이라고 기대하였지만, 여전히 살아있습니다! 이 문제는 방금 해결한 문제와 정반대의 경우입니다: 좀 전에는 버킷을 생성하는 명령과 테이블을 갱신하는 사이의 지연이 문제였으며, 이번에는 버킷 프로세스가 죽고난 뒤, 이 정보들이 테이블에서 제거되는 사이에 발생하는 지연이 문제입니다.

불행하게도 이번에는 ETS 테이블의 정리를 담당하고 있는 동작인 `handle_info/2`를 동기적인 연산으로 변경해서는 해결할 수 없습니다. 대신 버킷이 죽었을 때 `:DOWN` 메시지를 동기적으로 처리할 수 있도록 보장할 수 있는 방법을 찾아야합니다.

간단한 방법으로는 레지스트리에 동기적인 요청을 보내는 것입니다: 왜냐하면 메시지들은 순서대로 실행되므로, 레지스트리가 `Agent.stop`이후에 전송된 메시지에 대한 응답을 돌려주었다면 `:DOWN` 메시지가 처리되었다는 것을 의미하기 때문입니다. 이를 위해 동기적인 메시지인 "bogus" 버킷 생성 요청을 한번 던져보죠:


```elixir
  test "removes buckets on exit", %{registry: registry} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(registry, "shopping")
    Agent.stop(bucket)

    # Do a call to ensure the registry processed the down message
    _ = KV.Registry.create(registry, "bogus")
    assert KV.Registry.lookup(registry, "shopping") == :error
  end

  test "removes bucket on crash", %{registry: registry} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

    # Kill the bucket and wait for the notification
    Process.exit(bucket, :shutdown)

    # Wait until the bucket is dead
    ref = Process.monitor(bucket)
    assert_receive {:DOWN, ^ref, _, _, _}

    # Do a call to ensure the registry processed the DOWN message
    _ = KV.Registry.create(registry, "bogus")
    assert KV.Registry.lookup(registry, "shopping") == :error
  end
```

이제 테스트는 언제나 통과할 겁니다.

이것으로 이번 장은 끝입니다. ETS를 캐시로 사용하여 어떤 프로세스에서든 접근하여 읽을 수 있지만, 생성한 프로세스만이 쓸 수 있게 해봤습니다. 그리고 데이터를 비동기적으로 읽어올 경우에는 좀 전에 소개했던 그런 경합 상태를 발생할 수 있다는 중요한 사실을 배웠습니다.

이제 덩치가 큰 코드에서 Mix가 어떻게 내/외부 의존성을 관리할 수 있도록 돕는지 알아봅시다.

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Elixir - OTP: ETS](http://elixir-lang.org/getting-started/mix-otp/ets.html)
