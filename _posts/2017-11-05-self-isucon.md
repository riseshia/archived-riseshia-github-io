---
layout: post
title: "Self ISUCON"
date: 2017-11-05 19:39:52 +0900
categories:
---

## ISUCON?

[좋은 느낌으로 빠르게 만드는 콘테스트](http://isucon.net).

요는 주어진 어플리케이션 / 서버로 스코어를 최대한 많이 뽑으면 되는 대회입니다. 매번 병목 지점이 다르기 때문에 병목지점을 발견하고 그걸 잘 개선하는게 포인트. 3명 팀으로 참가하는 대회라서 역할 분담도 중요합니다.

이번달 말미에 사내 ISUCON도 있고 해서 가벼운 마음으로 과거문제를 풀어봤습니다. 이하는 타임라인.

## Timeline

### 10:28 start

일단 서버를 슥 둘러봅니다. 하나도 모르겠습니다 [-]
과거문 설명서에서 벤치마크를 돌리는 법을 찾아서 실행합니다.

```
> ./isucon6q-bench -target http://127.0.0.1
```

Score는 초기 상태인 0

### 10:36 sudo 명령을 쓸 수가 없어서 고통

기본으로 사용하는 isucon 사용자의 비밀번호가 미설정이라 sudo를 쓸 수 없다는 사실이 발각. 설정합니다.

```
> sudo passwd isucon
```

### 10:52 ruby관련 바이너리의 경로 설정

bundle/ruby/...이 전부 경로 설정이 안되어 있는걸 발견. systemctl 설정파일을 뒤져서 박혀있는 경로를 찾고, 이걸 전부 `~/bin`에 심볼릭 링크했습니다.

이제와서 생각해보면 그냥 환경변수 PATH에 `.local/ruby/bin`을 넣는게 나았을텐데...

### 11:12 mysql 비밀번호 발견

환경변수로 지정하는건 알고 있었는데 안보여서 도대체 어딘가 싶었는데, systemctl 설정에서 env.sh라는 환경 변수 설정용 스크립트를 호출하는 것을 발견.

### 11:24 nginx log format

nginx log를 보고 병목지점을 찾기 위해서 로그 포맷을 변경합니다. `alp`를 사용할 계획이므로 여기서 받을 수 있는 스타일로.

```
http {

  log_format ltsv "time:$time_local"
                  "\thost:$remote_addr"
                  "\tforwardedfor:$http_x_forwarded_for"
                  "\treq:$request"
                  "\tstatus:$status"
                  "\tmethod:$request_method"
                  "\turi:$request_uri"
                  "\tsize:$body_bytes_sent"
                  "\treferer:$http_referer"
                  "\tua:$http_user_agent"
                  "\treqtime:$request_time"
                  "\tcache:$upstream_http_x_cache"
                  "\truntime:$upstream_http_x_runtime"
                  "\tapptime:$upstream_response_time"
                  "\tvhost:$host";

  access_log  /var/log/nginx/access.log  ltsv;
}
```

### 11:25 alp install

```
sudo cat /var/log/nginx/access.log | alp -r | less
```

로 통계를 볼수 있는 것을 확인.

### 11:52 static serve

nginx 로그를 보니 일단 어셋이 제일 느리길래 nginx가 뿌려주도록 변경합니다.

```
location /css/ {
  root /home/isucon/webapp/public;
  sendfile           on;
  sendfile_max_chunk 1m;
  tcp_nopush on;
}
location /js/ {
  root /home/isucon/webapp/public;
  sendfile           on;
  sendfile_max_chunk 1m;
  tcp_nopush on;
}
location /img/ {
  root /home/isucon/webapp/public;
  sendfile           on;
  sendfile_max_chunk 1m;
  tcp_nopush on;
}
location /favicon.ico {
  root /home/isucon/webapp/public;
  sendfile           on;
  sendfile_max_chunk 1m;
  tcp_nopush on;
}
```

여전히 스코어는 0. Timeout의 감점이 너무 커서 눈에 안보이는 모양. 로그 상에서 어셋 처리 자체는 시간이 거의 안걸리게 되었으니 ok라고 치고 다음으로 넘어갑니다.

### 12:03 slow query 분석 준비

sql에서의 병목을 체크하기 위해서 slow query를 로그로 뽑아내기로 합니다.

```
[mysqld]
slow_query_log                = 1
slow_query_log_file           = /var/log/mysql/mysqld-slow.log
long_query_time               = 0.2
log-queries-not-using-indexes =
```

~하지만 이 로그 데이터가 사용되는 일은 없었ㄷ...~

### 12:12 unicorn 로그 설정

튜닝 중에 잘못 고친 코드로 인한 기동시의 에러를 잡아내기 위해서 이쪽에도 로그를 설정합니다.

```
stderr_path File.expand_path('../../../log/unicorn_stderr.log', __FILE__)
stdout_path File.expand_path('../../../log/unicorn_stdout.log', __FILE__)
```

### 12:27 isutar가 동작하지 않는 것을 확인

뭐지? 싶어서 봤더니, isutar 기동하는 것을 까먹고 있었던걸 발견. 지금까진 Timeout에 가려서 거기까지 요청이 가지도 못했던 모양.

### 12:46 unicorn process 숫자 증가

slow query를 봐도 0.3초 이상 걸리는게 없는데 끝없이 타임아웃이 뜨길래, 일단 늘려봤지만 여전히 줄지 않았습니다.

스코어는 여전히 0.

### 12:55 nginx worker process를 늘려봤다

위와 동일한 이유로 nginx 워커를 늘려봅니다.

스코어가 드디어 0에서 벗어납니다. 635. 여전히 타임아웃은 발생하는 중.

### 13:07 rack-lineprof를 넣었다

여전히 느린 이유를 이해할 수 없어서 rack-lineprof를 넣었습니다.

```
# gem 'rack-lineprof'

require 'rack-lineprof'
require 'profiler'
```

### 13:18 sql에 필요한 것만 select...

키워드 자동 링크를 생성하는 `htmlify`가 토나오게 느리길래 select를 최소화해봅니다.

스코어 6443. 드디어 타임아웃이 사라졌습니다.

### 13:33 키워드 매칭에 사용하는 정규식을 수정

키워드 매칭에 사용하는 정규식 생성에 시간이 많이 걸리길래 좀 더 빠르게 고쳐봅니다.

```
keywords = db.xquery(%| select keyword from entry order by character_length(keyword) desc |)
pattern = Regexp.union(keywords.map {|k| k[:keyword] })
kw2hash = {}
hashed_content = content.gsub(pattern) {|matched_keyword|
  "isuda_#{Digest::SHA1.hexdigest(matched_keyword)}".tap do |hash|
    kw2hash[matched_keyword] = hash
  end
}
```

스코어 5908. 딱히 빨라지지 않았습니다... rack-lineprof의 영향인가 싶었는데 그냥 애초에 빨라지지 않아졌던 모양.

캐시를 먹이기로 하고 되돌립니다.

### 14:12 키워드 regexp만드는 부분을 캐싱

스코어는 5936. 역시 눈에 띄는 속도 증가는 없었습니다. 되돌립니다.

### 14:31 html 캐싱

이번엔 아예 치환 작업까지 포함해서 캐싱.

스코어는 13261. 꽤 빨라졌습니다.

### 15:44 서버 대통합

서버 두 대가 HTTP 요청을 주고 받고 꽤 자주 쓰이는 부분이 있어서 서버를 통합하기로 결정.

스코어 15471.

### 16:05 unicorn_process 를 슬슬 8로

이제 리소스가 좀 더 남는 걸 확인했으므로 unicorn_process를 다시 끌어 올립니다.

스코어 15635. 딱히 올라가진 않습니다...

### 16:19 nginx/unicorn_process를 불림

다시 불려봅니다. 그러자 벤치마커가 죽습니다[..] 일단 파일 소켓을 동시 열림 상한을 끌어 올립니다.

```
ulimit -n 10000
```

스코어 20205.

ps: 이제와서 보면 이 시점의 벤치 결과는 캐싱이 되어있는 상태에서 동작한 모양...

### 16:30 star select 를 최소화

별박기에도 sql select 최소화를 걸어보기로 합니다.

스코어는 17113.

### 16:50 캐시 무효화 범위 줄이기

슬슬 벤치에 등장하는 파일 숫자가 5천을 넘어가기 시작했으므로 캐시 무효화에도 좀 더 신경을 쓰기로 합니다. 갱신이 필요한 키워드에만 무효화를 하도록 수정합니다.

스코어 기록해둔게 없었습니다..

### 17:32 stars도 캐시

마찬가지로 별박기 데이터에도 캐싱을 적용합니다. 목표는 select 쿼리를 아예 안던지는 것이었는데, 거기까진 진행을 못함...

그리고 여기가 최종 스코어. 18404. 캐싱된 상태라면 3.5만 언저리까지. 예선 통과 성적이 9만점 언저리니까 한참 머네요 [..]

## 감상

- 머리를 전혀 쓰지 않더라도 기계적으로 튜닝할 수 있는 범위는 의외로 있어서, 한번 경험해보면 해당 부분을 빠르게 기계적으로 최적화하고 시작할 수 있을 듯 했습니다.
  - 2회차를 하게 되면 2시간 정도는 절약할 수 있을 듯.
- 병목 지점을 찾는 게 의외로 어렵네요.
- 캐싱은 정의입니다[..]
- 혼자서 하다보니 서버에서 대강 작업했는데, 여러명이 했을때를 상정해봐야.
- 벤치 마킹에서 실패하는 건 점수를 엄청 깎아먹습니다. 뿐만 아니라 재기동 시험에서 실패해서 탈락하는 케이스가 발생할 수 있으니 이 부분에 대해서는 주의할 것.
