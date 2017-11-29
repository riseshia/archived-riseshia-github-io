---
layout: post
title: "Monad? Monad!"
date: 2017-11-29 21:25:24 +0900
categories:
---

## Monad 괴담

자세한 설명은 [이 링크](https://e.xtendo.org/haskell/ko/monad_fear/slide)로 생략합니다.

전부 동의하는가? 라고 하면 동의하기 어려운 지점은 분명 존재하지만, 일반적으로 하스켈에 느끼는 좌절감을 가장 잘 이야기하고, 오해라는 걸 설명해주는 글이라서입니다. 무엇보다 읽다보면 재밌습니다.

이 글은 튜토리얼도 아니고, 그냥 필자가 이해한 모나드에 대해서 간단하게 정리합니다.
독자는 Parameterized Type와 컨텍스트를 알고 있다는 전제 하에서 설명합니다.

## Let's Monad!

### Parameterized Type는 귀찮아요

```haskell
a = Just 1
```

여기서 `a`에 1을 더하고 싶습니다.

```haskell
maybePlusOne :: Maybe Integer -> Maybe Integer
maybePlusOne (Just a) = Just (a + 1)
maybePlusOne Nothing = Nothing
```

얍! 모든 Maybe에서 이걸 다 짠다고 생각하니 끔찍하네요.
값이 존재하지 않을 수 있다라는 컨텍스트를 날로 얻기 위해서 너무 많은 타이핑 비용을 지불하는건 아닐까요?
뭔가 이걸 알아서 처리해주는 무언가가 있으면 좋을거 같습니다.
예를 들어, `(a -> b) -> Maybe a -> Maybe b` 이런 추상화된 함수가 있으면 행복할 것 같네요.

```haskell
callInMaybe :: (a -> b) -> Maybe a -> Maybe b
callInMaybe func (Just a) = Just (func a)
callInMaybe func Nothing = Nothing

callInMaybe (+ 1) a -- Just 2
```

이제 callInMaybe 가 있으면 어떤 함수든 다 처리할 수 있습니다! 근데 말이죠.

```haskell
(+ 1) <$> a -- Just 2
```

이미 있습니다. `<$>`가 바로 그 함수입니다. 그리고 이 함수를 제공하는 타입 클래스를 Functor라고 부릅니다. 정확한 시그니처는 다음과 같습니다.

`Functor f => (a -> b) -> f a -> f b`

간단하게 설명하면 감싸진 값을 꺼낸 뒤, 넘겨받은 함수를 이용해서 가공한 뒤, 다시 감싸고 있던 원래의 컨텍스트로 감싼 뒤 반환해줍니다.

### 인수가 많아도 귀찮아요

이번엔 Maybe로 감싸진 두 값의 보편적인 덧셈을 구현해보죠.

```haskell
maybePlus :: Maybe Integer -> Maybe Integer -> Maybe Integer
maybePlus (Just a) (Just b) = Just (a + b)
maybePlus _ _ = Nothing
```

점점 타이핑해야하는 양이 늘어나고 있습니다. 우울해집니다.

좀 전에 설명한 `<$>`를 사용하면 어떻게 될까요?

```haskell
:t (+) <$> a
-- (+) <$> a :: Maybe (Integer -> Integer)
```

컨텍스트의 내부에서 Partial apply을 해버렸네요! `<$>`만으로는 수습할 수가 없다는걸 깨달았습니다.
그렇다면, 컨텍스트 내부에서 partial apply를 해주는 무언가가 있으면 날로 먹을 수 있지 않을까요?

`Maybe (a -> b) -> Maybe a -> Maybe b`를 처리해주는 함수가 있으면 될거 같습니다.

```haskell
applyInMaybe :: Maybe (a -> b) -> Maybe a -> Maybe b
applyInMaybe (Just func) (Just a) = Just (func a)
applyInMaybe _ _ = Nothing

applyInMaybe ((+) <$> a) (Just 2) -- Just 3
```

험난하네요. 하지만 이제 이 함수가 있다면 컨텍스트 내에서 partial apply를 할 수 있습니다! 그리고 제가 뭘 말하려는지 아실겁니다.

```haskell
(+) <$> Just 1 <*> Just 2
```

역시 있습니다. `<*>`를 제공하는 타입 클래스를 Applicative라고 부르며, 정확한 시그니처는 다음과 같습니다.

`Applicative f => f (a -> b) -> f a -> f b`

야호!

### Finally Monad!

이번엔 Map에서 lookup한 값을 가지고 다시 lookup을 해봅시다.

```haskell
import qualified Data.Map as Map

m = Map.fromList [(1, 2), (2, 3)]

lookupFromM :: Integer -> Maybe Integer
lookupFromM a = Map.lookup a m

maybeLookupFromM :: Maybe Integer -> Maybe Integer
maybeLookupFromM (Just a) = lookupFromM a
maybeLookupFromM Nothing = Nothing

first = lookupFromM 1
second = maybeLookupFromM first -- Just 3
```

lookup을 두 번 했을 뿐인데 함수를 하나 선언해야 합니다. `dic[1][2][3]`같은 코드를 짤 생각을 하니 눈앞이 캄캄해지네요. 역시 개발자는 게을러야하니 범용 함수를 하나 준비해보죠. 어떤 시그니처가 필요할까요? 받은 Maybe를 벗기고, 그 값을 받아서 또다른 Maybe를 반환하는 함수를 받은 뒤, 그 연산값을 돌려주면 될 것 같네요.

`Maybe a -> (a -> Maybe b) -> Maybe b`

완벽합니다! 이제 짜봅시다.

```haskell
unwrapApply :: Maybe Integer -> (Integer -> Maybe Integer) -> Maybe Integer
unwrapApply (Just a) func = func a
unwrapApply Nothing _ = Nothing

unwrapApply (lookupFromM 1) lookupFromM -- Just 3
```

그리고 바로 이게 모나드입니다.

```haskell
lookupFromM 1 >>= lookupFromM -- Just 3
```

`>>=`의 시그니처는 `Monad m => m a -> (a -> m b) -> m b`입니다.
값을 받아서 컨텍스트로 감싼 값을 반환하는 어떤 함수를 컨텍스트에 사용할 수 있도록 해주죠.

## 정리하자면,

Functor, Applicative, Monad는 모두 Parameterized Type, 특히 컨텍스트를 표현하는 것들을 좀 더 손쉽게 다루기 위한 무언가입니다.

- Functor: 컨텍스트 내부의 값을 변환하고 싶을 때
- Applicative: 컨텍스트 내부에서 partial apply를 하고 싶을 때
- Monad: 컨텍스트로 감싼 값에 대해서 감싸지 않은 값을 받아, 컨텍스트로 감싼 값을 반환하는 함수를 사용하고 싶을 때
- pure: 이 글에서 설명하진 않았지만, 어떤 값을 컨텍스트로 감싸고 싶을 때에 활용할 수 있습니다.

그리고 이것들을 확장해서 사용하기 시작하면 리스트 모나드 같은 무시무시한 것들이 되죠. 물론 구현을 보면 의외로 이해하기 쉬운 물건입니다만...

## 결론

수학으로 돌아가서 원론적인 의미를 이해하는 것도 좋지만, 개발자의 피부에 와닿는 건 역시 실전적인 이유인 것 같습니다.

왜 모나드를 쓰죠? 없으면 불편하니까요!!

- ps1: 사실 Parameterized Type과 컨텍스트를 이해할 수 있으면 이 글은 필요 없는거 아닌가, 하는 생각이 다 쓰고 들었습니다...
- ps2: `Get Programming with Haskell` 라는 책에서 훨씬 알아듣기 쉽게 기초부터 잘 설명해줍니다. 흥미가 생기셨다면 일독을 권합니다.
