---
layout: post
title: "Elixir - OTP: Agent"
date: 2016-05-16 21:45:08 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 거의 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 많습니다. 원문은 최하단의 링크를 참고하세요.

## OTP: Agent

이 챕터에서는 `KV.Bucket`라는 이름의 모듈을 만들 겁니다. 이 모둘은 키-값 데이터들을 다른 프로세스들에서 읽고, 수정할 수 있도록 하는 책임을 집니다.

만약 튜토리얼 1부를 읽지 않았거나, 오래전에 읽었다면, [Processes](http://elixir-lang.org/getting-started/processes.html)를 읽어 보기를 권장합니다.

## 상태와 관한 문제

Elixir는 상태불변한 언어이며 어떤 것도 기본으로 공유되지 않습니다. 만약 공간을 만들고 다양한 장소에서 값을 추가/삭제할 수 있게끔 상태를 제공하고 싶다면, Elixir에서는 크게 두 가지 방법이 있습니다:

* 프로세스
* [ETS (Erlang Term Storage)](http://www.erlang.org/doc/man/ets.html)

이미 프로세스에 대해서는 이야기했지만, <abbr title="Erlang Term Storage">ETS</abbr>는 그렇지 않습니다. 여기에 대해서는 앞으로 배울 예정입니다. 프로세스에 대해서는 우리가 직접 사용하지 않고 Elixir에서 제공하는 추상화 계층과 <abbr title="Open Telecom Platform">OTP</abbr>를 사용할 겁니다:

* [Agent](http://elixir-lang.org/docs/stable/elixir/Agent.html) - 상태를 감싸줍니다.
* [GenServer](http://elixir-lang.org/docs/stable/elixir/GenServer.html) - "일반 서버들" (프로세스들)은 상태를 캡슐화하며, 동기, 비동기 호출, 코드 리로딩 등을 지원합니다.
* [GenEvent](http://elixir-lang.org/docs/stable/elixir/GenEvent.html) - "일반 이벤트"는 이벤트를 다수의 헨들러에게 보낼 때에 사용합니다.
* [Task](http://elixir-lang.org/docs/stable/elixir/Task.html) - 프로세스를 생성하고 결과를 나중에 받아올 수 있는 비동기적인 계산 유닛입니다.

이 가이드에서는 방금 설명한 대부분의 추상화에 대해서 살펴볼 것입니다. 이것들은 모두 <abbr title="Virtual Machine">VM</abbr>가 기본으로 제공하는 `send`, `receive`, `spawn` 그리고 `link`와 같은 기능들 위에서 동작하는 프로세스 상에서 동작한다는 점을 기억하세요.

## Agents

[Agents](http://elixir-lang.org/docs/stable/elixir/Agent.html)는 상태를 감싸기만 합니다. 만약 프로세스에게 원하는 것이 단순히 상태를 유지하는 것만이라면, 에이전트는 무척 잘 맞습니다. 그러면 `iex` 세션을 프로젝트에서 여는 것부터 시작합시다:

```bash
$ iex -S mix
```

그럼 이제 에이전트를 가지고 간단한 코드를 작성해보죠.

```iex
iex> {:ok, agent} = Agent.start_link fn -> [] end
{:ok, #PID<0.57.0>}
iex> Agent.update(agent, fn list -> ["eggs" | list] end)
:ok
iex> Agent.get(agent, fn list -> list end)
["eggs"]
iex> Agent.stop(agent)
:ok
```

에이전트를 빈 리스트와 함께 실행했습니다. 새 문자열을 리스트의 첫 자리에 넣는 것으로 에이전트의 상태를 변경합니다. [`Agent.update/3`](http://elixir-lang.org/docs/stable/elixir/Agent.html#update/3)의 두번째 변수는 에이전트의 현재 상태를 입력으로 받으며, 기대되는 새로운 상태를 반환받습니다. 마지막으로 리스트 전체를 받아옵니다. [`Agent.get/3`](http://elixir-lang.org/docs/stable/elixir/Agent.html#get/3)의 두번째 인자는 입력으로 상태를 받으며, 반환한 결과값을 `Agent.get/3` 자신의 반환값으로 돌려줍니다. 그리고 에이전트에 일을 다 시킨 뒤에는 `Agent.stop/1`릃 호출하여 프로세스를 종료시킬 수 있습니다.

그럼 이제 에이전트를 사용하여 `KV.Bucket`을 구현해봅시다. 하지만 구현을 시작하기 전에 일단 테스트를 작성하도록 하죠. `test/kv/bucket_test.exs`라는 파일을 만들어봅시다(`.exs` 확장자를 사용해야한다는 점을 기억하세요).

```elixir
defmodule KV.BucketTest do
  use ExUnit.Case, async: true

  test "stores values by key" do
    {:ok, bucket} = KV.Bucket.start_link
    assert KV.Bucket.get(bucket, "milk") == nil

    KV.Bucket.put(bucket, "milk", 3)
    assert KV.Bucket.get(bucket, "milk") == 3
  end
end
```

`KV.Bucket`의 테스트로 `get/2`와 `put/3`을 사용하고, 기대값을 적어봅시다. 여기에서는 명시적으로 에이전트를 정지시키지 않을 것입니다. 왜냐하면 이들은 테스트 프로세스에 연결되며, 테스트가 종료되면 자동적으로 종료되기 때문입니다. 프로세스에 이름을 정해주지 않는 이상 언제나 이렇게 동작합니다.

그리고 `ExUnit.Case`의 옵션으로 `async: true`를 넘겼다는 것을 확인하세요. 이 옵션은 이 테스트가 다른 테스트 케이스들과 병렬적으로 실행할 수 있게끔 해줍니다. 이는 머신 상에서 다수의 CPU 코어를 사용하여 테스트의 실행 속도를 끌어올릴수 있게끔 해줍니다. 하지만 `:async` 옵션은 테스트 케이스가 전역값에 의존하지 않거나, 변경하지 않는 경우에 한해서만 사용하세요. 만약 테스트가 파일 시스템에 무언가를 작성하거나, 프로세스를 등록 또는 데이터베이스에 접근하는 등의 작업을 수행한다면 다른 테스트 케이스들과의 경합상태에 빠지는 것을 방지하기 위해 이 옵션을 사용하지 마세요.

테스트가 동기이건, 비동기이건 상관없이 이 새로운 테스트들은 아직 실제 기능이 구현되지 않았기 때문에 반드시 실패합니다.

실패하는 테스드들을 통과시키기 위해서, 이제 `lib/kv/bucket.ex`라는 파일을 생성하고 다음의 내용을 작성해봅시다. 또는 직접 `KV.Bucket`의 내용물을 작성해봐도 좋습니다.

```elixir
defmodule KV.Bucket do
  @doc """
  Starts a new bucket.
  """
  def start_link do
    Agent.start_link(fn -> %{} end)
  end

  @doc """
  Gets a value from the `bucket` by `key`.
  """
  def get(bucket, key) do
    Agent.get(bucket, &Map.get(&1, key))
  end

  @doc """
  Puts the `value` for the given `key` in the `bucket`.
  """
  def put(bucket, key, value) do
    Agent.update(bucket, &Map.put(&1, key, value))
  end
end
```

키-값 쌍을 저장하기 위해서 맵을 사용하고 있습니다. 캡쳐 연산자 `&`는 [시작하기 전에](http://elixir-lang.org/getting-started/modules.html#function-capturing)라는 가이드에서 설명한 바가 있습니다.

그럼 이제 `KV.Bucket` 모듈이 정의되었으니 테스트를 통과할 수 있을겁니다. `mix test`를 실행해서 확인해보세요.

## ExUnit 콜백

나아가서 `KV.Bucket`에 새로운 기능을 추가하기 전에 ExUnit의 콜백에 대해서 짚고 넘어갑시다. 기대한대로 모든 `KV.Bucket`의 테스트는 실제 테스트가 시작되기 전에 저장할 공간이 생성되어 있어야하며, 테스트가 끝나면 종료되어야 합니다. 운이 좋게도, ExUnit은 콜백을 통해서 이러한 반복적인 작업을 줄일 수 있도록 해줍니다.

그러면 콜백을 사용해서 테스트 케이스를 재작성해봅시다:

```elixir
defmodule KV.BucketTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, bucket} = KV.Bucket.start_link
    {:ok, bucket: bucket}
  end

  test "stores values by key", %{bucket: bucket} do
    assert KV.Bucket.get(bucket, "milk") == nil

    KV.Bucket.put(bucket, "milk", 3)
    assert KV.Bucket.get(bucket, "milk") == 3
  end
end
```

우선 `setup/1` 매크로를 통해서 준비용 콜백을 정의했습니다. `setup/1` 콜백은 모든 테스트가 실행되기 전에 같은 프로세스의 내부에서 실행됩니다.

콜백으로부터 테스트 자체에게 `bucket`의 pid를 넘겨주기 위한 수단이 필요하다는 점을 깅거하세요. 이는 *테스트의 컨텍스트*를 통해서 처리할 수 있습니다. 우리가 `{:ok, bucket: bucket}`를 콜백으로부터 반환하게 되면 ExUnit은 튜플의 두번째 원소를 테스트의 컨텍스트에 추가합니다. 이 테스트 컨텍스트는 테스트 정의시에 매칭할 수 있는 맵이므로, 이를 통해서 컨텍스트 상에 있는 값들에 접근할 수 있게 됩니다:

```elixir
test "stores values by key", %{bucket: bucket} do
  # `bucket` is now the bucket from the setup block
end
```

테스트 케이스에 대한 더 자세한 설명은 [`ExUnit.Case` 모듈 문서](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.Case.html)를, 콜백에 대해서는 [`ExUnit.Callbacks` 문서](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.Callbacks.html)를 참고해주세요.

## 그 이외의 에이전트 동작들

에이전트의 상태를 변경하거나 값을 가져오는 것 뿐 만이 아니라, 에이전트는 한번에 값을 변경하고 가져올 수 있는 `Agent.get_and_update/2`를 지원합니다. 그럼 한번 `KV.Bucket.delete/2` 함수를 구현하여 저장 공간으로부터 키를 삭제하고, 현재 값을 번환해봅시다.

```elixir
@doc """
Deletes `key` from `bucket`.

Returns the current value of `key`, if `key` exists.
"""
def delete(bucket, key) do
  Agent.get_and_update(bucket, &Map.pop(&1, key))
end
```

이제 여기에 대한 테스트를 한번 작성해 봅시다! 더 필요한 정보가 있다면 [`Agent` 모둘에 대한 문서](http://elixir-lang.org/docs/stable/elixir/Agent.html)를 읽으며 더 배워봅시다.

## 에이전트에서의 클라이언트/서버

다음 챕터로 넘어가기 전에, 에이전트에 있어서의 클라이언트/서버가 무엇인지에 대해서 잠시 이야기해보죠. 좀 전에 우리가 구현한 `delete/2` 함수를 들여다봅시다.

```elixir
def delete(bucket, key) do
  Agent.get_and_update(bucket, fn dict->
    Map.pop(dict, key)
  end)
end
```

우리가 에이전트에게 넘긴 함수는 에이전트 프로세스의 내부에서 실행됩니다. 이 경우에는 이 경우에는 에이전트 프로세스가 우리의 메시지에 대해서 응답하는 유일한 프로세스이므로, 에이전트 프로세스는 서버의 역할을 하고 있다고 말할 수 있을겁니다. 그리고 이 함수 이외의 모든 것들은 클라이언트에서 일어납니다.

이 구별은 무척 중요합니다. 만약 처리되어야 하는 무척 비싼 비용의 행동이 있다고 한다면, 이 코드를 서버에서 처리하는 것이 좋을지, 클라이언트 쪽에서 처리하는 것이 좋은지에 대해서 고민해볼 필요가 있습니다. 예를 들어보죠:

```elixir
def delete(bucket, key) do
  :timer.sleep(1000) # puts client to sleep
  Agent.get_and_update(bucket, fn dict ->
    :timer.sleep(1000) # puts server to sleep
    Map.pop(dict, key)
  end)
end
```

만약 비용이 비싼 작업을 서버에서 실행하게 된다면, 그 작업이 종료될 때까지 다른 요청들은 하염없이 기다려야하며, 몇몇은 클라이언트에게 타임아웃이라는 결과를 돌려줄 수도 있을겁니다.

다음 챕터에서는 클라이언트와 서버의 구분을 좀 더 명확하게 만들어주는 GenServers에 대해서 알아봅니다.

## Reference
 * [Elixir Homepage](http://elixir-lang.org)
 * [Elixir - OTP: Agent](http://elixir-lang.org/getting-started/mix-otp/agent.html)
