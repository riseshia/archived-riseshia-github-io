---
layout: post
title: "Digging newtype"
date: 2017-08-20 13:05:18 +0900
categories:
---

## Easy Explanation

- 간편하게 isomorphic을 확보하기 위한 수단.
- 특정 조건(하나의 레코드, 하나의 데이터 생성자를 가짐)하의 데이터 타입일 때 (최적화를 위해서) 쓰세요.

## Difficult Explanation

### Isomorphic

```haskell
data Any = Any { getAny :: Bool }
```

이 `Any`를 사용하면 다음과 같이 isomorphic이 성립할 것처럼 보입니다.

```haskell
:t Any . getAny -- Any -> Any
```

그런데 성립하지 않는 경우가 존재합니다.

```haskell
Any . getAny $ Any True  -- Any True
Any . getAny $ Any False -- Any False
Any . getAny $ Any ⊥     -- Any ⊥
Any . getAny $ ⊥         -- Any ⊥
```

맨 마지막 동작은 어떻게 봐도 isomorphic하지 않은 것처럼 보이는데, 이는 `data`로 선언된 데이터 생성자가 넘겨 받은 값을 리프팅하기 때문입니다. 그래서 이런 예외를 처리하기 위해서 언제나 다음과 같은 코드를 작성해야 합니다.

```haskell
case x of
  Any _ -> ()
```

`data`로 선언된 데이터 생성자를 사용하는 경우, 평가를 해봐야만 결과를 확정할 수 있다는 점을 알 수 있습니다. 이러한 경우에도 isomorphic을 확보하려면 리프팅이 발생하지 않게끔 할 필요가 있고, 이를 실현해주는 것이 `newtype` 키워드입니다.

약간의 추측을 담아서 추가 설명을 하자면, 데이터 타입의 isomorphic을 확보하기 위한 키워드이므로, 하나의 레코드를 가진 하나의 데이터 생성자를 가져야 한다는 제약조건이 존재하는 것도 이해할 수 있습니다. 다시 말해 이는 `newtype`의 제약조건이 아닌, isomorphic한 데이터 타입의 제약 조건이라고 추측할 수 있겠네요.

### Optimization

Isomorphic한 조건을 만족하는 경우, 특정 타입에 대한 데이터 생성자가 단 하나로 확정되기 때문에 다음과 같은 장점이 생깁니다.

- 패턴 매칭에서 데이터 생성자의 평가를 강요할 필요가 없다.
- 값을 감싸거나, 타입을 벗기는 작업을 수행할 필요가 없다.

이상으로부터 `newtype`은 컴파일 시간에 최적화가 가능함을 알 수 있으며, Haskell 컴파일러가 이를 실제로 수행하고 있다는 것을 확인해보죠.

```haskell
module Foo where

data Foo1 = Foo1 Int
newtype Foo2 = Foo2 Int

-- 생성자 패턴 매칭에서 인수는 평가될 필요가 없으므로 동작합니다.
x1 = case Foo1 undefined of
     Foo1 _ -> 1    -- 1

-- 위와 마찬가지로 잘 동작할 것처럼 보입니다.
x2 = case Foo2 undefined of
     Foo2 _ -> 1    -- 1

-- 생성자 패턴 매칭에 실패합니다.
y1 = case undefined of
     Foo1 _ -> 1    -- undefined

-- 패턴 매칭에 성공합니다[?]
y2 = case undefined of
     Foo2 _ -> 1    -- 1
```

다시 말해서, 컴파일 이후에는 아래의 두 코드가 동등하게 취급된다고 추측할 수 있습니다. 정확히는 `newtype`이 감싸고 있는 내부 데이터 타입과 동일하게 취급합니다.

```haskell
z1 = case Foo2 undefined of
     Foo2 _ -> 1    -- 1
z2 = case undefined of
     _ -> 1         -- 1
```

## Reference

- [Newtype - Haskell wiki](https://wiki.haskell.org/Newtype)
- [Lifting - Haskell wiki](https://wiki.haskell.org/Lifting)
- [Why is there "data" and "newtype" in Haskell?](https://stackoverflow.com/questions/2649305/why-is-there-data-and-newtype-in-haskell)
- [Why is newtype necessary?](https://www.reddit.com/r/haskell/comments/5m8wz6/why_is_newtype_necessary/)
