---
layout: post
title: "Elixir - OTP: Task and gen_tcp"
date: 2016-06-29 08:38:06 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 많습니다. 원문은 최하단의 링크를 참고하세요.

## Introduction

이 장에서는 요청을 처리하기 위해서 [Erlang의 `:gen_tcp` 모듈](http://www.erlang.org/doc/man/gen_tcp.html)의 사용법을 알아볼 것입니다. 이 모듈은 Elixir의 `Task` 모듈에 대해 알아볼 훌륭한 기회를 제공합니다. 이 뒤로도 서버를 확장해나갈 것이므로, 유용하게 쓰일 것입니다.


## 에코 서버

우선 제대로 된 TCP 서버를 구현하기 앞서서 에코 서버로 시작해봅시다. 이것은 요청으로 받은 문자열을 그대로 돌려주는 간단한 서버입니다. 관리 트리로 관리할 수 있고, 한 번에 다수의 접속을 처리할 수 있게 될 때까지 조금씩 개선해나갈 것입니다.

TCP 서버는 크게 다음과 같은 흐름으로 동작합니다:

1. 포트를 열고, 소켓을 통해 그것을 유지합니다.
2. 클라이언트로부터의 요청을 기다리다가 받습니다.
3. 요청을 받아서 응답을 돌려줍니다.

이 순서대로 구현해봅시다. `apps/kv_server` 애플리케이션 폴더로 이동해서 `lib/kv_server.ex` 파일을 연 뒤에 다음의 함수를 추가하세요:

```elixir
require Logger

def accept(port) do
  # The options below mean:
  #
  # 1. `:binary` - receives data as binaries (instead of lists)
  # 2. `packet: :line` - receives data line by line
  # 3. `active: false` - blocks on `:gen_tcp.recv/2` until data is available
  # 4. `reuseaddr: true` - allows us to reuse the address if the listener crashes
  #
  {:ok, socket} = :gen_tcp.listen(port,
                    [:binary, packet: :line, active: false, reuseaddr: true])
  Logger.info "Accepting connections on port #{port}"
  loop_acceptor(socket)
end

defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end

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

`KVServer.accept(4040)`을 호출해서 서버를 동작시킬 것이며, 여기에서 4040은 포트 번호입니다. `accept/1`에서 처음으로 하는 작업은 소켓이 사용가능해질 때까지 포트를 듣고 있다가, `loop_acceptor/1`을 호출하는 것입니다. `loop_acceptor/1`은 단순히 클라이언트로부터의 접속을 받기 위한 루프입니다. 여기에서는 각각의 요청에 대해서 `serve/1`을 호출하게 됩니다.

`serve/1`은 소켓으로부터 받은 문자열을 읽고, 그대로 소켓으로 쓰는 또다른 루프입니다. `serve/1`은 [파이프 연산자 `|>`](http://elixir-lang.org/docs/stable/elixir/Kernel.html#%7C%3E/2)를 사용하여 작업의 흐름을 표현하고 있습니다. 이 파이프 연산자는 왼쪽을 평가한 결과를 오른쪽에 있는 함수의 첫번째 인자로 넘겨줍니다. 예제를 보시죠:

```elixir
socket |> read_line() |> write_line(socket)
```

는 다음과 동등합니다:

```elixir
write_line(read_line(socket), socket)
```

`read_line/1`은 내부에서 `:gen_tcp.recv/2`를 사용하여 소켓으로부터 데이터를 받아오며, `write_line/2`는 `:gen_tcp.send/2`를 사용하여 소켓에 정보를 쓰게 됩니다.

여기까지가 에코 서버를 구현하는 데 필요한 모든 것입니다. 한번 직접 만들어보세요!

`kv_server` 애플리케이션 폴더에서 `iex -S mix`를 사용해 IEx 세션을 시작해봅시다. IEx에서 다음을 실행하세요:

```iex
iex> KVServer.accept(4040)
```

서버가 실행되고, 콘솔이 사용할 수 없는 상태가 됨을 확인할 수 있을 것입니다. 그러니 [`텔넷` 클라이언트](https://en.wikipedia.org/wiki/Telnet)를 사용하여 서버에 접속해보도록 하죠. 대부분의 OS에는 사용가능한 클라이언트가 제공되며, 대부분 사용법이 비슷합니다:

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello
is it me
is it me
you are looking for?
you are looking for?
```

"hello"를 입력하고 엔터를 누르면, "hello"라는 메시지를 돌려받을 수 있을 겁니다. 훌륭합니다!

텔넷 클라이언트는 `ctrl + ]`나 `quit`라고 쓰고 `<Enter>`를 누르면 종료할 수 있는 경우가 많으며, 사용하는 클라이언트에 따라서는 다른 방법이 필요할 수도 있습니다.

텔넷 클라이언트를 종료하게 되면, IEx 세션에서 다음과 같은 에러 메시지를 확인할 수 있습니다:

    ** (MatchError) no match of right hand side value: {:error, :closed}
        (kv_server) lib/kv_server.ex:41: KVServer.read_line/1
        (kv_server) lib/kv_server.ex:33: KVServer.serve/1
        (kv_server) lib/kv_server.ex:27: KVServer.loop_acceptor/1

이는 클라이언트가 접속을 종료했음에도 불구하고 `:gen_tcp.recv/2`에서는 데이터를 기대하고 있기 때문입니다. 서버를 개선해 나가면서 이러한 경우를 처리할 수 있도록 해야 합니다.

지금은 그보다 더 중요한 문제점이 있습니다. TCP 처리기가 죽으면 무슨 일이 벌어지나요? 이 프로세스는 아무도 관리하고 있지 않기 때문에 서버가 죽으면 재시작이 되지 않으며, 더는 어떤 요청도 처리할 수가 없게 됩니다. 그러므로 우리는 이 서버를 관리 트리로 옮겨야 합니다.

## Task

지금까지 에이전트, 서버, 슈퍼바이저에 대해서 배웠습니다. 이것들은 모두 다수의 메시지나 상태를 다룹니다. 하지만 특정 작업을 실행만 하면 되는 경우에는 무엇을 사용해야 할까요?

[Task 모듈](http://elixir-lang.org/docs/stable/elixir/Task.html)은 바로 이런 기능을 제공합니다. 예를 들어서 여기에는 모듈, 함수, 인자를 받는 `start_link/3` 함수가 있으며, 이를 통해 주어진 함수를 관리 트리의 일부로서 실행할 수 있습니다.

직접 해보죠. `lib/kv_server.ex`를 열고 `start/2`에 있는 슈퍼바이저를 다음과 같이 변경하세요:

```elixir
def start(_type, _args) do
  import Supervisor.Spec

  children = [
    worker(Task, [KVServer, :accept, [4040]])
  ]

  opts = [strategy: :one_for_one, name: KVServer.Supervisor]
  Supervisor.start_link(children, opts)
end
```

이 변경을 통해서 `KVServer.accept(4040)`를 워커로서 실행했으면 좋겠다는 의도를 전달할 수 있습니다. 그리고 지금은 포트를 직접 입력하고 있습니다만, 이를 변경할 수 있도록 만드는 방법에 관해서도 이야기할 것입니다.

이제 서버는 관리 트리의 일부가 되었으므로, 애플리케이션을 실행함과 동시에 자동으로 시작될 것입니다. 터미널에 `mix run --no-halt`라고 입력하고, 다시 `텔넷` 클라이언트를 사용하여 정상적으로 동작하는지 확인해보도록 합시다:

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
say you
say you
say me
say me
```

잘 동작합니다! 클라이언트를 종료하여 서버가 죽게끔 하더라도 곧 새로 시작된 서버를 확인할 수 있을 겁니다. 하지만 이건 확장 가능한가요?

동시에 2개의 텔넷 클라이언트를 연결해보죠. 그러면 두 번째로 접속한 클라이언트에서는 응답이 돌아오지 않는다는 것을 확인할 수 있습니다:

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello?
HELLOOOOOO?
```

전혀 동작하는 것 같지 않습니다. 이는 접속을 받는 프로세스와 같은 프로세스에서 요청을 처리하고 있기 때문입니다. 한 클라이언트가 접속하면, 다른 클라이언트를 받을 수 없습니다.

## Task supervisor

서버가 여러 접속을 동시에 처리할 수 있게 만들려면 접속이 있을 때마다 별도의 요청 처리 프로세스를 생성하는 프로세스가 필요합니다. 예를 들어보자면 다음과 같이 말이죠:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end
```

`Task.start_link/3`와 유사하지만 모듈, 함수와 인자를 받는 대신 익명 함수를 넘겨받는 `Task.start_link/1`을 사용해봅시다:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  Task.start_link(fn -> serve(client) end)
  loop_acceptor(socket)
end
```

여기에서는 접속을 처리하는 프로세스에서 연결된 Task를 직접 시작하고 있습니다. 그런데, 우리는 비슷한 실수를 이전에도 한 번 저지른 적이 있습니다. 기억하시나요?

이것은 레지스트리에서 `KV.Bucket.start_link/0`을 직접 호출했던 것과 유사합니다. 하나의 버킷이라도 죽게 되면 레지스트리 전체를 망가뜨리는 문제가 있었습니다.

위의 코드에서도 같은 흐름이 가능합니다. `serve(client)` 작업을 접속 처리기에서 링크하게 되면, 요청을 처리하다가 문제가 발생하는 경우, 이는 접속 처리기로 영향을 주고 도미노처럼 다른 접속들에도 영향을 주어 죽게끔 합니다.

우리는 이러한 문제를 간단한 one for one 슈퍼바이저를 사용해서 해결했었습니다. 여기에서도 같은 전략을 사용할 생각입니다만, `Task`에서는 이러한 상황이 너무나 흔하므로 이미 여기에 대한 해결책을 가지고 있다는 점이 다릅니다. 임시 워커들을 위한 one for one 슈퍼바이저는 관리 트리에서 곧바로 사용할 수 있습니다!

`start/2`를 다시 변경하여 새 슈퍼바이저를 추가하도록 하죠:

```elixir
def start(_type, _args) do
  import Supervisor.Spec

  children = [
    supervisor(Task.Supervisor, [[name: KVServer.TaskSupervisor]]),
    worker(Task, [KVServer, :accept, [4040]])
  ]

  opts = [strategy: :one_for_one, name: KVServer.Supervisor]
  Supervisor.start_link(children, opts)
end
```

그저 [`Task.Supervisor`](http://elixir-lang.org/docs/stable/elixir/Task.Supervisor.html) 프로세스를 `KVServer.TaskSupervisor`라는 이름으로 실행했습니다. 접속 처리 작업은 이 슈퍼바이저에 의존하기 때문에, 이것이 가장 먼저 실행되어야 한다는 점을 잊지 마세요.

이제 `loop_acceptor/1`에서 `Task.Supervisor`를 사용하여 각 요청을 처리하도록 고치죠:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  {:ok, pid} = Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
  :ok = :gen_tcp.controlling_process(client, pid)
  loop_acceptor(socket)
end
```

`:ok = :gen_tcp.controlling_process(client, pid)`라는 코드가 추가 되었다는 점을 눈치채셨나요? 이 코드는 자식 프로세스가 `client` 소켓을 다룰 수 있게 만듭니다. 소켓은 처음에 승인된 프로세스에 묶이기 때문에, 만약 이렇게 제어권을 넘겨주지 않으면 처리기에서 클라이언트 소켓들을 전부 가지고 있게 됩니다. 그 결과 처리기에 문제가 생길 경우, 모든 소켓을 사용할 수 없게 됩니다.

`mix run --no-halt`로 새 서버를 실행한 뒤, 여러 개의 텔넷 클라이언트로 접속해봅시다. 이제 클라이언트를 종료하더라도 접속 처리기는 죽지 않는다는 것도 확인할 수 있을 겁니다. 좋습니다!

다음은 에코 서버의 전체 구현입니다:

```elixir
defmodule KVServer do
  use Application
  require Logger

  @doc false
  def start(_type, _args) do
    import Supervisor.Spec

    children = [
      supervisor(Task.Supervisor, [[name: KVServer.TaskSupervisor]]),
      worker(Task, [KVServer, :accept, [4040]])
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end

  @doc """
  Starts accepting connections on the given `port`.
  """
  def accept(port) do
    {:ok, socket} = :gen_tcp.listen(port,
                      [:binary, packet: :line, active: false, reuseaddr: true])
    Logger.info "Accepting connections on port #{port}"
    loop_acceptor(socket)
  end

  defp loop_acceptor(socket) do
    {:ok, client} = :gen_tcp.accept(socket)
    {:ok, pid} = Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
    :ok = :gen_tcp.controlling_process(client, pid)
    loop_acceptor(socket)
  end

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
end
```

슈퍼바이저 명세를 변경했기 때문에 이제 다시 질문할 시간이 왔습니다. 관리 전략은 여전히 적절한가요?

이 경우의 답은 네, 입니다. 만약 접속 처리기에서 문제가 발생하더라도, 이미 동작하고 있는 접속들을 함께 종료시킬 필요가 없습니다. 한편, 태스크 슈퍼바이저가 죽더라도 접속 처리기를 함께 종료해야 할 필요가 없습니다.

다음 장에서는 클라이언트의 요청을 분석하고 응답을 보낸 뒤, 서버를 종료하는 방법에 대해서 배워보도록 하죠.

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Task and gen_tcp](http://elixir-lang.org/getting-started/mix-otp/task-and-gen-tcp.html)
