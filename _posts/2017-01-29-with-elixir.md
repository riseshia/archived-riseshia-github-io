---
layout: post
title: "With Elixir"
date: 2017-01-29 11:43:14 +0900
categories:
---

Elixir에는 `with`라는 문법이 있는데요.
토이 프로젝트에서 사용해본 김에 이에 대해서 정리해봅니다.

## `with`??

### 간단한 소개

`with`는 `1.2.0`에서 추가된 새로운 문법입니다. 에, 작년 1월에 릴리스 되었으니 이제 딱 1년이 지났네요.
릴리스 노트에는 다음과 같이 설명되어있습니다.

> Addition of the `with` special form to match on multiple expressions:

```elixir
with {:ok, contents} <- File.read("my_file.ex"),
     {res, binding} <- Code.eval_string(contents),
     do: {:ok, res}
```

한번에 여러 표현식을 매칭하기 위한 특별한 방법이라고 언급되어 있습니다.

Jose가 [`with`를 소개한 글](https://gist.github.com/josevalim/8130b19eb62706e1ab37)을 한번 볼까요.

> with is for's younger brother.

잠시 사용예를 비교해보죠.

```elixir
for {:ok, x} <- [ok(1), error(2), ok(3)], do: x
#=> [1, 3]

with {:ok, x} <- ok(1),
     {:ok, y} <- ok(2),
     do: {:ok, x + y}
#=> {:ok, 3}
```

`for`는 컬렉션에서 각 요소를 꺼내서 매칭하는 경우에만 필요한 작업을 수행하며, with는 순서대로 작업을 처리하고 있습니다.
과연. 어느 쪽이든 주어진 요소들에 대해서 매칭을 수행한다는 점은 비슷하네요. 하지만 결과는 다릅니다.
전자는 각 요소들에 대해서 특정 작업을 실행한 결과의 리스트를, 후자는 모든 매칭 작업이 끝난 후에야 수행하는 표현식의 결과를 반환하고 있습니다.

### 상세한 소개

명세는 [API 문서](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#with/1)에 정의되어 있지만 링크 하나만 던지고 끝나면 재미 없으니
조금 더 정리해보죠.

`with`는 다음과 같은 형태를 가집니다.

```elixir
with matched_expr1 <- executed_expr1
        [, matched_expr2 <- executed_expr2…] do
  #=> 최종적으로 반환하고 싶은 값을 만드는 코드
else
  #=> 도중에 매칭이 실패했을 경우를 처리하는 코드
  unmatched_return -> expr3
  _ -> expr4
end
```

지금까지 설명에는 없었던 `else` 블럭이 등장했습니다.
이는 [작년 7월 즈음에 추가된 사양](https://github.com/elixir-lang/elixir/pull/4960)으로
이를 통해서 매칭에 실패한 상황을 좀 더 편하게 다룰 수 있게 되었습니다.
돌아와서 하나하나 설명해보도록 하죠.

- `with`의 뒤에 이어지는 매칭을 순차적으로 처리하며, 전부 매칭에 성공했다면 `do` 블록 내부의 코드를 실행합니다.
- 도중에 매칭이 실패했을 경우는 매칭에 실패한 값을 반환합니다.
- `else` 블럭이 존재한다면 실패한 값을 넘겨 받아 처리하게 되며, 이 블럭의 실행 결과를 `with` 구문의 반환값으로 사용합니다.
- `else` 블럭은 `->`를 사용하여 case처럼 처리하고 싶은 실패를 하나씩 지정하여 처리해야 합니다.

그리고 [테스트](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/test/elixir/kernel/with_test.exs)에 있는 코드를 참고해서 두가지 주의사항을 언급하겠습니다.

```elixir
test "else conditions with match error" do
  assert_raise WithClauseError, "no with clause matching: :error",  fn ->
    with({:ok, res} <- error(), do: res, else: ({:error, error} -> error))
  end
end
```

`else` 블럭은 모든 경우를 처리할 수 있다는 전제하에서 실행됩니다.
따라서 `else`를 사용하고, 그 내부에서 매칭에 실패한다면 `WithClauseError`가 발생합니다. 

```elixir
test "does not leak variables to else" do
  state = 1
  result = with 1 <- state, state = 2, :ok <- error(), do: state, else: (_ -> state)
  assert result == 1
  assert state == 1
end
```

`with` 내부에서 사용한 변수들은 외부에 영향을 주지 않습니다.

## Example

그럼 토이 프로젝트에서 있었던 상황을 잠시 살펴보겠습니다.

```elixir
defp validate_parent_id(changeset) do
  user_id = changeset.changes.user_id
  vpid = fn :parent_id, pid ->
    if p_task = Repo.get(Task, pid) do
      if p_task.user_id == user_id do
        []
      else
        [parent_id: "should be yours"]
      end
    else
      [parent_id: "is not found"]
    end
  end

  validate_change(changeset, :parent_id, vpid)
end
```

코드는 다음과 같은 사양으로 구현되었습니다.

- `Task` 모델은 부모 `Task`를 하나 가질 수 있다.
- `parent_id`가 있다면, 반드시 존재하는 `Task`를 가리켜야 한다.
- 부모 `Task`와 자식 `Task`는 동일한 사용자의 소유이어야 한다.

결과, 이를 검증하기 위해서 위와 같은 코드가 추가되었습니다.
`parent_id`로 `Task`를 가져올 수 있어야 하고, 그 소유자가 지금 검증하는 객체의 소유자와 같아야하며,
각각의 경우에 대해서 다른 결과 또는 에러를 돌려줘야 하는 상황입니다.
딱 봐도 `with`로 처리하면 좋을 듯한 코드네요.
그럼 `with`를 사용하도록 변경해보죠.

```elixir
defp validate_parent_id(changeset) do
  user_id = changeset.changes.user_id
  vpid = fn :parent_id, pid ->
    with p_task <- Repo.get(Task, pid),
         true <- p_task.user_id == user_id do
      []
    else
      nil -> [parent_id: "is not found"]
      false -> [parent_id: "should be yours"]
      _ -> raise "unhandled error"
    end
  end

  validate_change(changeset, :parent_id, vpid)
end
```

들여쓰는 횟수도 줄었고, 흐름도 명확해 보이네요.

## 마무리

어떠셨나요? 쓸만해 보이시나요?
[다른 링크](http://learningelixir.joekain.com/learning-elixir-with/)에서는 좀 더 명백하게 개선된 느낌을 주는 코드 예제가 있습니다만,
같은 예제를 재활용하는 것도 재미없고 해서 직접 경험한 상황에 대해서 기술했습니다.
이 글의 예제에서 뭐가 이득인지 잘 느껴지지 않으신다면 해당 예제를 읽어보시는 것도 좋겠네요.

저는 파이프 연산자를 긍정적으로 보고 있어서, 이와 함께 사용하기 어려운 `with`에 대해서 약간은 회의적인 입장이었습니다만
실제로 써보고 나서야 편리함을 느꼈습니다(그보다 지금까지 `with`를 한번도 안썼다는 점이 이상합니다만[..]).
지금와서 생각해보면 파이프 연산자를 쓰는 타이밍과 `with`를 사용하는 타이밍은 전혀 다른데, 왜 그리 피했나, 싶은 느낌이네요.
역시 뭐든 써봐야 한다는 교훈을 남긴 리팩토링이었습니다.

## 참고

- [Elixir Changelog v1.2.0](https://github.com/elixir-lang/elixir/blob/v1.2.0/CHANGELOG.md)
- [Introducing `with`](https://gist.github.com/josevalim/8130b19eb62706e1ab37)
- [Learning Elixir's with](http://learningelixir.joekain.com/learning-elixir-with/)
- [API documentation](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#with/1)
- [Implementation of `with`](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/src/elixir_with.erl)
- [with_test.exs](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/test/elixir/kernel/with_test.exs)
- [The else clause in with supports guards](https://github.com/elixir-lang/elixir/pull/4960)
