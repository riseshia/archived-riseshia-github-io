---
layout: post
title: "GraphQL Introduction"
date: 2016-12-10 11:39:22 +0900
categories:
---

이 글은 [GraphQL 튜토리얼](http://graphql.org/learn/queries/)을 보고 간단하게 정리한 글입니다.
문법의 기능적인 부분만 중점적으로 추려냈으니, 장단점에 대한 설명이 필요하시다면 원문을 참고하세요.

코드도 원문을 인용했습니다. 다만 몇몇 긴 부분은 추려서 줄였습니다.

## Queries and Mutation

### Fields

- GraphQL은 객체의 특정 필드에 대한 정보를 질의하는 것
- 그런 이유로 와일드카드는 존재하지 않음
- 요청과 결과가 비슷함
- 중첩된 객체를 가져올 수 있음

```javascript
// Query
{
  hero {
    name
    # Queries can have comments!
    friends {
      name
    }
  }
}

// Result
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friends": [
        {
          "name": "Luke Skywalker"
        },
        {
          "name": "Leia Organa"
        }
      ]
    }
  }
}
```

### Arguments

- 해당 타입에서 원하는 것만 가져올 수 있음.
- `<type>(<attr>: <value>)`

```js
// Query
{
  human(id: "1000") {
    name
    height(unit: FOOT)
  }
}

// Result
{
  "data": {
    "human": {
      "name": "Luke Skywalker",
      "height": 5.6430448
    }
  }
}
```

### Aliases

- 별칭도 지정할 수 있음.

```js
// Query
{
  empireHero: hero(episode: EMPIRE) {
    name
  }
  jediHero: hero(episode: JEDI) {
    name
  }
}

// Result
{
  "data": {
    "empireHero": {
      "name": "Luke Skywalker"
    },
    "jediHero": {
      "name": "R2-D2"
    }
  }
}
```

### Fragment

- 파셜과 같은 느낌으로 반복되는 부분을 별도로 정의할 수 있음.
- 문법을 보고 있으면 expand라는 느낌을 지울 수가 없다.

```js
// Query
{
  leftComparison: hero(episode: EMPIRE) {
    ...comparisonFields
  }
  rightComparison: hero(episode: JEDI) {
    ...comparisonFields
  }
}

fragment comparisonFields on Character {
  name
  appearsIn
  friends {
    name
  }
}

// Result
{
  "data": {
    "leftComparison": {
      "name": "Luke Skywalker",
      "appearsIn": [
        "NEWHOPE"
      ],
      "friends": [
        {
          "name": "Han Solo"
        }
      ]
    },
    "rightComparison": {
      "name": "R2-D2",
      "appearsIn": [
        "NEWHOPE"
      ],
      "friends": [
        {
          "name": "Luke Skywalker"
        }
      ]
    }
  }
}
```

### Variables

- 동적으로 변수를 주입할 수 있음
- `$variable`과 같은 방식으로 선언, 사용
- 변수 선언에는 타입을 지정해야함
- `!`을 사용해서 조건부인지 필수인지 지정할 수 있음
- 값을 주입할 때에는 `key: value`(주로 JSON) 방식으로 처리

```js
// Query
query HeroNameAndFriends($episode: Episode) {
  hero(episode: $episode) {
    name
    friends {
      name
    }
  }
}

// Variables
{
  "episode": "JEDI"
}

// Result
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friends": [
        {
          "name": "Luke Skywalker"
        }
      ]
    }
  }
}

### Directives

- 변수를 사용해 조건부로 특정 필드를 추가/제거할 수 있음
- Field, Fragment에서 사용할 수 있음
- 공식 스펙에서 정의하는 것은 2개.
  - `@include(if: Boolean)`
  - `@skip(if: Boolean)`

```js
// Query
query Hero($episode: Episode, $withFriends: Boolean!) {
  hero(episode: $episode) {
    name
    friends @include(if: $withFriends) {
      name
    }
  }
}

// Variables
{
  "episode": "JEDI",
  "withFriends": false
}
```

### Mutations

- not only fetch, but also post
- 일반 쿼리와 유사한 방식으로 사용할 수 있음
- 일반 쿼리는 변경 가능성이 없어서 병렬로 처리하지만, Mutation은 변경 가능성이 있으므로 순차실행

```js
// Query
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review) {
    stars
    commentary
  }
}

// Variables
{
  "ep": "JEDI",
  "review": {
    "stars": 5,
    "commentary": "This is a great movie!"
  }
}

// Results
{
  "data": {
    "createReview": {
      "stars": 5,
      "commentary": "This is a great movie!"
    }
  }
}
```

### Inline fragment

- 특정 타입인 경우에 값을 추가. -> 인터페이스 역할

```js
// Query
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    name
    ... on Droid {
      primaryFunction
    }
    ... on Human {
      height
    }
  }
}
```

### Meta field

- 해당 GraphQL 서비스의 정보를 반환하는 필드
- `__typename`
- `__schema`
- `__type`

```js
# Query
{
  __schema {
    types {
      name
    }
  }
}

// Result
{
  "data": {
    "__schema": {
      "types": [
        {
          "name": "Query"
        },
        // ...
        {
          "name": "__DirectiveLocation"
        }
      ]
    }
  }
}
```
