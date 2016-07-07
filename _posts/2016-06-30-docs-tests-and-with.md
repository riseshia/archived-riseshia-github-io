---
layout: post
title: "Elixir - OTP: Docs, tests and with"
date: 2016-06-30 19:26:38 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 많습니다. 원문은 최하단의 링크를 참고하세요.

## Introduction

이 장에서는 첫 번째 장에서 보인 명령들을 처리하기 위한 코드를 구현할 것입니다:

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

이 처리가 끝나면 필요한 명령을 `:kv` 애플리케이션에 전송하여 서버를 갱신할 것입니다.

## Doctests

언어 소개 페이지를 보면 Elixir는 언어 레벨에서 문서를 1급 시민으로 다룬다고 언급하고 있습니다. 우리는 이 개념에 대해서 이 가이드에서 `mix help`나 IEx에서 입력하는 `h Enum`과 같은 것들을 통해 여러 번 이야기해왔습니다.

이 절에서는 문서에서 바로 테스트를 작성할 수 있도록 해주는 Doctest를 사용하여 명령 처리 기능을 구현할 것입니다. 이를 통해 정확한 코드 예제를 포함하는 문서를 만들 수 있습니다.

그럼 이제 명령어 처리기를 위해 `lib/kv_server/command.ex`를 만들고 여기부터 시작해보죠:

```elixir
defmodule KVServer.Command do
  @doc ~S"""
  Parses the given `line` into a command.

  ## Examples

      iex> KVServer.Command.parse "CREATE shopping\r\n"
      {:ok, {:create, "shopping"}}

  """
  def parse(line) do
    :not_implemented
  end
end
```

Doctest는 문서 문자열 안에 포함된 4개의 공백과 그 뒤에 오는 `iex>`로 식별할 수 있습니다. 만약 명령어를 여러 줄로 작성해야한다면 IEx에서 볼 수 있는 것처럼 `...>`를 사용하세요. 그에 따른 기댓값은 `iex>`나 `...>`의 바로 다음 줄에 작성해야 합니다. 그리고 그 이후에는 개행이나 새로운 `iex>`가 와야 합니다.

그리고 문서를 작성하기 위해서 `@doc ~S"""`을 사용했다는 점을 눈치채셨나요? `~S`는 테스트 도중에 `\r\n`가 실제 개행이나 캐리지 리턴으로 변환되지 않게 합니다.

이 테스트를 실행하기 위해서는 `test/kv_server/command_test.exs` 파일을 만들고, `doctest KVServer.Command`를 실행해야 합니다.

```elixir
defmodule KVServer.CommandTest do
  use ExUnit.Case, async: true
  doctest KVServer.Command
end
```

테스트 스위트를 실행하면 테스트는 실패할 것입니다.

    1) test doc at KVServer.Command.parse/1 (1) (KVServer.CommandTest)
       test/kv_server/command_test.exs:3
       Doctest failed
       code: KVServer.Command.parse "CREATE shopping\r\n" === {:ok, {:create, "shopping"}}
       lhs:  :not_implemented
       stacktrace:
         lib/kv_server/command.ex:11: KVServer.Command (module)

좋습니다!

이제 테스트를 통과할 수 있게 만들면 되겠네요. 그럼 이제 `parse/1` 함수를 구현해보죠.

```elixir
def parse(line) do
  case String.split(line) do
    ["CREATE", bucket] -> {:ok, {:create, bucket}}
  end
end
```

이 구현은 넘겨받은 줄을 공백으로 나눈 뒤, 리스트에 있는 명령들과 매칭합니다. `String.split/1`을 사용한다는 것은 이 코드에서는 공백에 둔감하다는 의미입니다. 줄의 맨 처음, 맨 마지막에 나오는 공백들은 무시되며, 단어 사이에 있는 연속된 공백들도 그렇습니다. 그럼 다른 명령어들의 동작을 테스트하기 위해서 새 테스트를 추가로 작성해보죠!

```elixir
@doc ~S"""
Parses the given `line` into a command.

## Examples

    iex> KVServer.Command.parse "CREATE shopping\r\n"
    {:ok, {:create, "shopping"}}

    iex> KVServer.Command.parse "CREATE  shopping  \r\n"
    {:ok, {:create, "shopping"}}

    iex> KVServer.Command.parse "PUT shopping milk 1\r\n"
    {:ok, {:put, "shopping", "milk", "1"}}

    iex> KVServer.Command.parse "GET shopping milk\r\n"
    {:ok, {:get, "shopping", "milk"}}

    iex> KVServer.Command.parse "DELETE shopping eggs\r\n"
    {:ok, {:delete, "shopping", "eggs"}}

Unknown commands or commands with the wrong number of
arguments return an error:

    iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
    {:error, :unknown_command}

    iex> KVServer.Command.parse "GET shopping\r\n"
    {:error, :unknown_command}

"""
```

Doctest를 다 작성했으면 이제 이 테스트들이 통과할 수 있게 만들 시간입니다! 직접 한번 만들어보시고, 아래에 있는 코드와 비교해보세요:

```elixir
def parse(line) do
  case String.split(line) do
    ["CREATE", bucket] -> {:ok, {:create, bucket}}
    ["GET", bucket, key] -> {:ok, {:get, bucket, key}}
    ["PUT", bucket, key, value] -> {:ok, {:put, bucket, key, value}}
    ["DELETE", bucket, key] -> {:ok, {:delete, bucket, key}}
    _ -> {:error, :unknown_command}
  end
end
```

어떻게 명령어의 이름과 인수들의 개수들을 확인하기 위한 산더미 같은 `if/else` 없이 멋지게 명령어를 처리하는지 보세요.

마지막으로 각각의 Doctest는 다른 테스트로 취급되어서 현재 테스트 스위트에는 총 7개의 테스트가 있는 것처럼 보일 것입니다. 이는 ExUnit이 다음과 같은 경우에는 별도의 테스트인 것처럼 다루기 때문입니다:

```iex
iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
{:error, :unknown_command}

iex> KVServer.Command.parse "GET shopping\r\n"
{:error, :unknown_command}
```

다음과 같이 개행을 사용하지 않는다면 ExUnit은 하나의 테스트인 것처럼 컴파일합니다:

```iex
iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
{:error, :unknown_command}
iex> KVServer.Command.parse "GET shopping\r\n"
{:error, :unknown_command}
```

[`ExUnit.DocTest` 문서](http://elixir-lang.org/docs/stable/ex_unit/ExUnit.DocTest.html)에서 더 자세한 설명을 확인할 수 있습니다.

## with

이제 명령어를 분석할 수 있게 되었으므로, 이제 실제로 명령어를 실행하는 부분을 구현할 때가 왔습니다. 우선 이 함수의 최소한의 코드를 추가해봅시다.

```elixir
defmodule KVServer.Command do
  @doc """
  Runs the given command.
  """
  def run(command) do
    {:ok, "OK\r\n"}
  end
end
```

이 함수를 구현하기 전에, `parse/1`과 `run/1` 함수를 사용하기 전에 서버가 시작되도록 만들어 봅시다. 잊지 마세요, `read_line/1` 함수는 클라이언트가 소켓을 닫으면 같이 사망하기 때문에, 이를 해결할 기회를 줘야 합니다. `lib/kv_server.ex`를 열어서 서버의 명세를 변경하죠:

```elixir
defp serve(socket) do
  socket
  |> read_line()
  |> write_line(socket)

  serve(socket)
end

defp read_line(socket) do
  {:ok, data} = :gen_tcp.recv(socket, 0)
  data
end

defp write_line(line, socket) do
  :gen_tcp.send(socket, line)
end
```

이 코드를 다음과 같이 변경하세요.

```elixir
defp serve(socket) do
  msg =
    case read_line(socket) do
      {:ok, data} ->
        case KVServer.Command.parse(data) do
          {:ok, command} ->
            KVServer.Command.run(command)
          {:error, _} = err ->
            err
        end
      {:error, _} = err ->
        err
    end

  write_line(socket, msg)
  serve(socket)
end

defp read_line(socket) do
  :gen_tcp.recv(socket, 0)
end

defp write_line(socket, {:ok, text}) do
  :gen_tcp.send(socket, text)
end

defp write_line(socket, {:error, :unknown_command}) do
  # Known error. Write to the client.
  :gen_tcp.send(socket, "UNKNOWN COMMAND\r\n")
end

defp write_line(_socket, {:error, :closed}) do
  # The connection was closed, exit politely.
  exit(:shutdown)
end

defp write_line(socket, {:error, error}) do
  # Unknown error. Write to the client and exit.
  :gen_tcp.send(socket, "ERROR\r\n")
  exit(error)
end
```

서버를 시작하면 이제 명령을 전송할 수 있게 됩니다. 지금은 알고 있는 명령에 대해서 "OK"를 주거나 "UNKNOWN COMMAND"를 돌려줍니다.

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
CREATE shopping
OK
HELLO
UNKNOWN COMMAND
```

이는 구현 작업이 올바른 방향으로 가고 있다는 것을 의미합니다만, 그렇게 우아해 보이지 않습니다.

이전의 구현에서는 파이프라인을 사용해서 코드를 알기 쉽게 작성했었습니다. 하지만 다른 에러 코드들을 처리하기 위해서 `case`를 중첩해서 호출하고 있습니다.

고맙게도 Elixir v1.2는 위와 같은 코드를 간결하게 만들 수 있는 `with`라는 구조를 지원합니다. 이를 사용해서 `serve/1` 함수를 개선해봅시다.

```elixir
defp serve(socket) do
  msg =
    with {:ok, data} <- read_line(socket),
         {:ok, command} <- KVServer.Command.parse(data),
         do: KVServer.Command.run(command)

  write_line(socket, msg)
  serve(socket)
end
```

많이 나아졌습니다! `with`는 `for` 표현식과 많이 유사합니다. `with`는 `<-` 우측에 있는 값을 반환받고 좌측에 있는 패턴과 매칭을 시도합니다. 만약 값이 매칭된다면 `with`는 다음 표현식으로 넘어갑니다. 아무 것도 매칭하는 것이 없다면, 매칭에 실패한 값이 그대로 돌아옵니다.

말하자면, `case/2`에 넘겨주는 각 표현식을 `with`의 단계로 변환한 것입니다. 만약 `{:ok, x}`에 매칭되는 것이 없다면, `with`는 중단되고 매칭에 실패한 그 값을 반환하게 됩니다.

더 자세한 설명은 [`with`에 대한 문서](http://elixir-lang.org/docs/stable/elixir/Kernel.SpecialForms.html#with/1)를 참고하세요.

## 명령 실행하기

`KVServer.Command.run/1`을 구현하는 마지막 단계는 해석된 명령을 `:kv` 애플리케이션에서 실행하는 것입니다. 이 구현은 다음과 같습니다:

```elixir
@doc """
Runs the given command.
"""
def run(command)

def run({:create, bucket}) do
  KV.Registry.create(KV.Registry, bucket)
  {:ok, "OK\r\n"}
end

def run({:get, bucket, key}) do
  lookup bucket, fn pid ->
    value = KV.Bucket.get(pid, key)
    {:ok, "#{value}\r\nOK\r\n"}
  end
end

def run({:put, bucket, key, value}) do
  lookup bucket, fn pid ->
    KV.Bucket.put(pid, key, value)
    {:ok, "OK\r\n"}
  end
end

def run({:delete, bucket, key}) do
  lookup bucket, fn pid ->
    KV.Bucket.delete(pid, key)
    {:ok, "OK\r\n"}
  end
end

defp lookup(bucket, callback) do
  case KV.Registry.lookup(KV.Registry, bucket) do
    {:ok, pid} -> callback.(pid)
    :error -> {:error, :not_found}
  end
end
```

이 구현은 무척 직관적입니다. `:kv` 애플리케이션을 시작하며 등록한 `KV.Registry`를 가져옵니다. `:kv_server`는 `:kv` 애플리케이션에 의존하고 있으므로, 여기서 제공하는 서버/서비스를 사용하는 것은 문제가 없습니다.

`lookup/2`라는 이름의 비공개 함수를 생성해서 해당하는 버킷을 찾고 존재한다면 `pid`를 돌려주고 그렇지 않다면 `{:error, :not_found}`를 돌려주는 함수를 정의해 둔 것을 확인하세요.

그런데 만약 `{:error, :not_found}`를 돌려받았다면, 이를 `write_line/2`에 포함하여 다음과 같이 돌려줄 필요가 있습니다.

```elixir
defp write_line(socket, {:error, :not_found}) do
  :gen_tcp.send(socket, "NOT FOUND\r\n")
end
```

이제 서버 대부분의 기능이 완성되었습니다! 이제 테스트를 추가하면 됩니다. 이번에는 결정해야 할 몇몇 중요한 고려사항이 있으므로 테스트 작성을 가장 마지막에 남겨두었습니다.

`KVServer.Command.run/1`의 구현은 `:kv` 애플리케이션에 등록된 `KV.Registry`라는 이름의 서버로 직접 명령을 전송하고 있습니다. 이 말인즉슨, 이 서버가 전역으로 동작하고 있으며, 만약 동시에 2개의 테스트가 메시지를 전송하게 되면 서로 충돌이 발생할 것입니다(그리고 실패하겠죠). 우리는 이제 고립된 상태로 비동기로 동작하는 테스트를 작성할지, 아니면 전역 상태 위에서 동작하는 통합 테스트를 작성할지 결정해야 합니다.

지금까지 일반적으로 하나의 모듈만을 직접 테스트하는 유닛 테스트만을 작성해왔습니다. 하지만 `KVServer.Command.run/1`을 하나의 모듈로서 테스트할 수 있게 만들려면 내부에서 직접 `KV.Registry`로 메시지를 보내지 않고, 인자로서 넘겨받도록 구현을 변경해야 합니다. 다시 말해 `run`의 메소드 시그니처를 `def run(command, pid)`로 변경해야하고 `:create` 명령은 다음과 같이 구현될 겁니다.

```elixir
def run({:create, bucket}, pid) do
  KV.Registry.create(pid, bucket)
  {:ok, "OK\r\n"}
end
```

그리고 `KVServer.Command`의 테스트에서는 `apps/kv/test/kv/registry_test.exs`에서 그랬던 것처럼 `KV.Registry`의 인스턴스를 실행하고 `run/2`의 인자로 서버 정보를 넘겨주어야 합니다.

이것은 지금까지 우리가 테스트에서 취해왔던 접근법이며, 여기에는 몇 가지 장점이 있습니다:

1. 구현은 어떤 서버의 이름과도 묶여있지 않습니다.
2. 상태를 공유하지 않으므로 테스트를 비동기적으로 유지할 수 있습니다.

하지만 API는 모든 외부 인자들을 다루기 위해서 점점 커지게 될 것이므로 이는 그다지 바람직하지 않습니다.

대안으로는 TCP 서버로부터 버킷까지의 모든 스택을 동작시키는 전역 서버 이름에 의존하는 통합 테스트를 작성하는 것입니다. 통합 테스트의 단점은 유닛 테스트보다 꽤 느리다는 점이며, 그런 이유로 정말 필요한 순간에만 사용해야 합니다. 예를 들어 명령 분석 기능을 테스트하기 위해서 통합 테스트를 사용해서는 안 됩니다.

이제 통합 테스트를 작성해야 할 때가 왔습니다. 통합 테스트는 서버에 명령을 보내고, 원하는 응답을 받아서 확인하기 위해서 TCP 클라이언트를 사용합니다.

`test/kv_server_test.exs`에 다음의 통합 테스트를 작성해봅시다:

```elixir
defmodule KVServerTest do
  use ExUnit.Case

  setup do
    Application.stop(:kv)
    :ok = Application.start(:kv)
  end

  setup do
    opts = [:binary, packet: :line, active: false]
    {:ok, socket} = :gen_tcp.connect('localhost', 4040, opts)
    {:ok, socket: socket}
  end

  test "server interaction", %{socket: socket} do
    assert send_and_recv(socket, "UNKNOWN shopping\r\n") ==
           "UNKNOWN COMMAND\r\n"

    assert send_and_recv(socket, "GET shopping eggs\r\n") ==
           "NOT FOUND\r\n"

    assert send_and_recv(socket, "CREATE shopping\r\n") ==
           "OK\r\n"

    assert send_and_recv(socket, "PUT shopping eggs 3\r\n") ==
           "OK\r\n"

    # GET returns two lines
    assert send_and_recv(socket, "GET shopping eggs\r\n") == "3\r\n"
    assert send_and_recv(socket, "") == "OK\r\n"

    assert send_and_recv(socket, "DELETE shopping eggs\r\n") ==
           "OK\r\n"

    # GET returns two lines
    assert send_and_recv(socket, "GET shopping eggs\r\n") == "\r\n"
    assert send_and_recv(socket, "") == "OK\r\n"
  end

  defp send_and_recv(socket, command) do
    :ok = :gen_tcp.send(socket, command)
    {:ok, data} = :gen_tcp.recv(socket, 0, 1000)
    data
  end
end
```

통합테스트는 존재하지 않는 명령과 원하는 값이 발견되지 않는 상황을 포함한 모든 서버의 동작을 확인합니다. <abbr title="Erlang Term Storage">ETS</abbr> 테이블과 연결된 프로세스들, 그리고 소켓들을 전부 닫지 않는다는 점을 확인하세요. 테스트 프로세스가 종료되면 소켓은 자동으로 닫힙니다.

이제 테스트는 전역 데이터에 의존하기 때문에 이번에는 `use ExUnit.Case`에 `async: true`를 넘기지 않았습니다. 나아가 테스트가 언제나 깨끗한 상태에서 이루어진다는 것을 보장하기 위해서 각 테스트를 시작하기 전에 `:kv` 애플리케이션을 재시작합니다. 사실 `:kv`를 정지하면 터미널에서는 경고를 발생시킵니다:

```
18:12:10.698 [info] Application kv exited: :stopped
```

테스트 중에 이러한 에러 메시지를 출력하지 않기 위해 ExUnit은 `:capture_log`라고 불리는 정리 기능을 제공합니다. 각 테스트를 실행하기 전에 `@tag :capture_log`를 설정하거나 모든 테스트 케이스를 위해 `@moduletag :capture_log`를 설정하게 되면 ExUnit은 자동으로 테스트 중에 발생하는 모든 로그를 접수합니다. 테스트가 실패했다면 가지고 있던 로그를 ExUnit 보고서와 함께 출력합니다.

setup의 앞에 다음의 호출을 추가하도록 하죠:

```elixir
@moduletag :capture_log
```

테스트가 실패하는 경우, 다음과 같은 보고서를 볼 수 있을 것입니다.

```
  1) test server interaction (KVServerTest)
     test/kv_server_test.exs:17
     ** (RuntimeError) oops
     stacktrace:
       test/kv_server_test.exs:29

     The following output was logged:

     13:44:10.035 [info]  Application kv exited: :stopped
```

이 간단한 통합 테스트를 통해서 왜 통합 테스트가 느린지를 확인할 수 있습니다. 테스트가 비동기로 동작하지 않을 뿐만 아니라, `:kv` 애플리케이션을 재시작하는 등의 비싼 준비 비용을 요구하기 때문입니다.

마지막으로 애플리케이션에 대한 가장 좋은 테스트 전략을 밝혀내는 것은 당신과 당신의 팀에 달려 있다는 점을 기억하세요. 코드 품질, 신뢰도, 테스트 스위트의 실행 시간을 모두 고려해야 합니다. 예를 들어 통합 테스트에서만 서버를 실행하고, 미래의 릴리스에서 점점 서버의 크기가 커지게 된다면, 또는 버그가 쉽게 발생하는 애플리케이션 일부가 될지도 모른다면 이 테스트를 분리하여 통합테스트에 부담이 가지 않도록 좀 더 집중적으로 유닛 테스트를 작성하는 방법을 고려해보는 것도 중요합니다.

다음 장에서는 버킷 라우팅 알고리즘을 추가하여 분산 환경에서 동작할 수 있도록 만들어 보겠습니다. 더불어 애플리케이션 설정에 대해서도 알아보도록 하겠습니다.

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Docs, tests and with](http://elixir-lang.org/getting-started/mix_otp/docs-tests-and-pipelines.html)
