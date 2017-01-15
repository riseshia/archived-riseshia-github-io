---
layout: post
title: "Travis에서 Postgres 최신판 사용하기"
date: 2017-01-14 16:54:02 +0900
categories:
---

Travis에서 최신 PostgreSQL을 사용하고 싶은 경우가 있습니다.
예를 들어서 JSONB 타입을 사용하는 경우라던가 말이죠.
물론 9.2 부터 사용가능합니다만, 이왕이면 실환경, 개발, 테스트 환경에서 동일한
버전의 데이터베이스를 쓰는 것이 바람직하기 때문에, 스터디에서 사용하고 있는
타겟 버전인 9.6을 사용하기로 했습니다. 그래서 이 글에서는 Travis에서
Postgres 9.6을 사용하는 방법에 대해서 알아봅니다.

## Status of Travis

당연하지만 Travis는 Postgres를 기본 옵션으로 지원하고
있습니다([PostgreSQL Setting](https://docs.travis-ci.com/user/database-setup/#PostgreSQL)
).

```yml
services:
  - postgresql
```

라는 옵션과 함께 사용할 수 있게 되며, 버전은 위에서 말했듯 9.1을 사용합니다.
그럼 다른 버전은 어떻게 사용할 수 있을까요? 직접 설치를 해야할까요?
문서를 잘 읽어보면 다른 버전도 사용할 수 있음을 알 수 있습니다.
다만 여기에는 제약사항이 존재합니다. 바로 사용하는 컨테이너의 종류,
sudo 옵션 사용 여부에 따라서 [사용 가능한 버전이 달라진다는 점](https://docs.travis-ci.com/user/database-setup/#Using-a-different-PostgreSQL-Version)입니다.

보시면 `trusty`를 사용하는 경우에 9.6 버전을 사용할 수 있다고 되어 있습니다.
후. 직접 깔 필요는 없겠네요. 하지만 여기서 의문점이 하나 떠오릅니다.

`trusty`라는 이름의 컨테이너가 뭐죠?

## `Trusty` Container

많은 분들이 이미 짐작하셨을지도 모르겠지만,
Ubuntu 14.04의 코드 네임인 그 `trusty`가 맞습니다.

혹시 모르셨던 분들도 있을까 싶어서 알려드리면, 지금까지 저희가 써왔던 컨테이너는
Ubuntu 12.04를 기반으로 하는 컨테이너입니다.
`trusty`는 [작년 11월 발표](https://blog.travis-ci.com/2016-11-08-trusty-container-public-beta/)된 새 컨테이너이며,
이름 그대로 Ubuntu 14.04를 기반으로 하는 컨테이너입니다. 현재는 베타로 제공되고
있으며 다음과 같은 옵션으로 사용할 수 있습니다.

```yml
dist: trusty
```

이 옵션을 추가하여 실행하게 되면 Travis에서 `This job ran on our container-based trusty beta. Please read our blog post about the public beta.`라는
안내 문구를 볼 수 있습니다. 설명에 따르면 주의할 점은 권한 문제 정도인데요.
`sudo` 옵션을 사용하고 계시는 경우, Travis에서 허가된 패키지가 아니라면 실행을
보장하지 않습니다. 그리고 이유는 아직 잘 모르겠는데 기존 컨테이너보다 수분 정도
느립니다. 초기만 그러는건지 앞으로 계속 그러는지는 좀 두고봐야할 듯.

자 그럼 이제 사용할 버전만 지정해주면 되겠네요.

## Conclusion

최종적으로 `.travis.yml`에 필요한 설정은 다음과 같습니다.

```yml
dist: trusty
services:
  - postgresql
addons:
    postgresql: "9.6"
```
