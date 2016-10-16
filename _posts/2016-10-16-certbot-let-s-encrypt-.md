---
layout: post
title: "Certbot 도입하기 ~ Let's Encrypt 갱신편 ~"
date: 2016-10-16 14:41:42 +0900
categories:
---

## Introduction

현재 개인적으로 운영하고 있는 웹서비스들은 전부
[Let's Encrypt](https://letsencrypt.org)를 이용하여 HTTPS를 적용 중입니다.
당시에 참고했던 글은 [Outsider](https://twitter.com/Outsideris)님의
[Let's Encrypt로 무료로 HTTPS 지원하기](https://blog.outsider.ne.kr/1178)이고,
그럭저럭 잘 쓰고 있었습니다.

다만 3개월 뒤에 발각된 문제는 **인증서 갱신을 할 수 없었다**는 점입니다[..]
원래 참고했던 글 내용도 있고해서 이번에도 Outsider님의
[Lets' Encrypt 인증서 갱신하기](https://blog.outsider.ne.kr/1198)를 참고하여
진행했는데, 이상하게 갱신이 안되더랬습니다.
덕분에 매 분기 인증서를 3~4개씩 아예 새로 만드는 방식으로 현실도피를 해왔는데,
슬슬 안되겠다 싶어 좀 진지하게 찾아보기 시작한게 몇일전.
당시에는 동작이 영 보장되지 않았던 [certbot](https://certbot.eff.org)이
이제 좀 안정적으로 동작한다는 정보를 입수하고,
이를 사용하는 구조로 작업흐름을 변경했습니다.
실제로 해야하는 작업도 적고 무척 편했는데, 그럼 가볍게 살펴보도록 하겠습니다.

## certbot?

간단하게 정리하면 환경에 맞춰 Let's Encrypt 인증서를 자동으로 발급/갱신해주는 봇입니다.
Let's Encrypt의 인증서 발급 방식을 간단하게 이야기하자면,
`인증서 요청 -> 도메인에 대한 소유권 확인 챌린지 -> 발급`과 같은 절차를 밟습니다.
certbot은 이러한 부분의 처리를 자동으로 수행해줍니다.

### Before start

이 글은 Ubuntu 14.04, Nginx 사용을 기준으로 작성되었습니다.

### Install

```bash
wget https://dl.eff.org/certbot-auto
chmod a+x certbot-auto
```

를 통해서 다운로드할 수 있습니다. 이는 Ubuntu 14.04, Nginx 기준이며,
다른 환경의 경우에는 [홈페이지](https://certbot.eff.org/)를 참조하세요.

```bash
./certbot-auto
```

그리고 방금 실행 권한을 추가한 스크립트를 실행하시면 필요한 의존성을 알아서
잘 설치해주게 됩니다(필요한 의존성 설치를 위해 root 권한을 요구할 겁니다).

### Optaining Certificate

홈페이지의 가이드를 보시면 하단에 대문짝만한
`Experimental Nginx Plugin Support`라는 문구를 보실 수 있습니다.
백업을 꼭 하시라고 신신당부하고 있죠.
그러니 우리는 눈물을 머금고 `webroot` 플러그인을 사용합시다.
이 플러그인을 통해서 실제 챌린지 파일을 위치시킬 경로를 지정해줄 수 있습니다.

```bash
$ ./path/to/certbot-auto certonly --webroot -w /var/www/challenge -d some.site.com
```

명령어를 간단하게 소개하자면,

- `certonly`: 인증서만을 얻어오겠음
- `--webroot`: `webroot` 플러그인을 쓰겠음
- `-w /var/www/challenge`: 챌린지 파일을 생성할 기준 폴더를 지정
- `-d some.site.com`: 인증서를 생성할 도메인을 지정

이를 실행하면 `some.site.com`에 대한 인증서를 생성할 수 있습니다.
하지만 잠시만 기다려주세요.

### Setup Nginx

명령을 실행하기 전에 외부에서 생성한 챌린지 파일에 접근할 수 있게 해줘야 합니다.
이를 위해서 서버 설정에 다음을 추가합시다.

```
location /.well-known {
  root /var/www/challenge/;
}
```

주의사항은 이렇습니다.

- HTTPS가 아닌, HTTP 포트 설정에 추가해줄 것.
- 여러 가상 호스트를 사용하고 있다면 각각을 별도로 추가해 줄 것(전역으로 location을 설정할 수 있을법도 한데 찾지는 못헸습니다;).


그럼 이제 명령어를 실행하세요. 서버가 켜져 있다면 정상적으로 발급됩니다.

### Setup Certificate

발급된 인증서는 `/etc/letsencrypt/live/sitename` 폴더에 생성됩니다.
여기에서 사용한 예제대로라면,

- /etc/letsencrypt/live/some.site.com/fullchain.pem
- /etc/letsencrypt/live/some.site.com/privkey.pem

처럼 인증서가 생성됩니다. 이를 서버 설정에 추가해주시면 됩니다.

ps: 다른 곳에서는 `/etc/certbot/..`에 생성되었다는 이야기도 있으니 참고하세요.

### Renew certificate with cron

갱신은 그냥 이전과 동일한 방식으로 renew 명령을 실행해주면 됩니다.
잘 동작하는지 확인하시려면 `--dry-run` 옵션을 사용하세요.

그럼 이제 알아서 자동으로 갱신해줄 수 있도록 cronjob에 등록해봅시다.

```bash
./certbot-auto renew --quiet --no-self-upgrade
```

등록해야하는 명령은 조금 다른 옵션을 가지고 있습니다.
본래 스크립트를 실행하면 자기 자신을 최신버전으로 업데이트하려고 시도하는데,
이 동작을 비활성화하기 위한 옵션과 로그를 무시하는 옵션을 넘겨주고 있습니다.

그럼 이제 crontab을 열고,

```crontab
# Begin Let's encrypt renew
0 19 1 1/3 * /bin/bash -l -c '/home/deployer/certbot-auto renew --quiet --no-self-upgrade'
# End Let's encrypt renew
```

같은 느낌으로 등록하시면 됩니다. 여기에서는 3개월에 한 번,
새벽 4시에 갱신을 시도하게끔 설정해뒀습니다.


### Conclusion

간단함은 정의입니다. 이왕 쓸 거라면 쉽게 쉽게 사용합시다.
Nginx 플러그인이 안정적이 되면 더 좋아질 거라 기대해봅시다.

### Trouble Shooting

필요 없으실거라 생각하지만 혹시 몰라 준비했습니다.

#### Server Error

챌린지를 통과하고 Let's Encrypt에 인증서를 요청하면 `Server Error`가 돌아오는 경우가 있습니다.
받을 수 있는 정보라곤 달랑 이것 뿐이라서 추측이 매우 어렵습니다만,
이럴 때는 그냥 `--force` 옵션을 넘겨주세요.

#### Root permission

지금까지 본 명령에서는 sudo를 사용하지 않았습니다만,
내부적으로 sudo로 root 권한을 요구합니다.
당장 dry-run만 해보더라도 다음과 같은 로그를 확인하실 수 있습니다.

```
...
Requesting root privileges to run certbot...
...
```

그런 이유로 권한 문제가 생기신 분들은 sudo를 비밀번호 없이 사용할 수 있도록 해주시거나,
혹은 자동 갱신을 포기하거나, 또는 다른 방법을 찾아주시면 되겠습니다.

