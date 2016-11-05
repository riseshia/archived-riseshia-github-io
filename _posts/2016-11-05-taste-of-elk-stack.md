---
layout: post
title: "Taste of ELK Stack"
date: 2016-11-05 11:13:41 +0900
categories:
---

## Prologue

회사에서 로그를 쌓기만 하다보니 뭔가 하나 붙여볼까, 해서 시작한 삽질에 대한
정리본입니다.

굳이 ELK 스택이 아니어도 괜찮았습니다만, 처음 이야기가 나온게 ELK라서 그대로
진행했습니다. 좀 시대에 늦었나, 같은 생각도 했지만, 기본적인 틀은 크게 차이가
나는 것 같지도 않고...

## ELK?

[ElasticSearch](https://www.elastic.co/products/elasticsearch),
[Logstash](https://www.elastic.co/products/logstash),
[Kibana](https://www.elastic.co/products/kibana) 를 묶어서 부르는 통칭입니다.
로그를 모아서 Logstash로 보내고, 필요한 조작을 수행한 뒤 이를 ElasticSearch로
수집합니다. 그리고 이를 Kibana로 예쁘게 시각화 해보자, 같은 흐름이 되겠습니다.
더 자세한 설명 같은 건 여기가 아니라도 설명한 곳이 많으니 생략.

## Example

요새는 모르겠지만, 이전에 ElasticSearch 설치하다가 삽푼 기억이 있어서 아무런
망설임 없이 Docker를 선택.

이미 누군가 잘 만들어 놓은 이미지가 있어서 그 중에 하나를 선택했습니다.
선택한 이미지는 [elk-docker](https://elk-docker.readthedocs.io)로, 문서 정리가
잘 되어있고, 이미지 하나로 편하게 쓸 수 있어서 선택했습니다.
실제 환경에서 스케일링을 고려하면 각각 따로 있는 이미지가 낫겠지만서도,
여기에서는 맛을 보는게 목적이기 때문에 그 부분에 대한 고려는 생략.

### Requirement

어떤 걸 써도 큰 문제는 없겠지만, 이하는 Rails의 애플리케이션 로그를 가져오는
경우를 상정하고 진행합니다. [여기](https://github.com/riseshia/elk_example)에
사용된 코드를 정리해두었으니 참고하세요.

### `docker-compose.yml`

일단 해당 이미지를 불러오기 위한 `docker-compose.yml`입니다. 딱히 유별날건 없고,
프로젝트 폴더를 `/app`에 마운트하고, `30-output.conf`를 Logstash 설정에
마운트합니다. 전자는 수월한 로그 수집을 위해서, 후자는 아래에서 설명합니다.

```yml
version: '2'
services:
  elk:
    image: sebp/elk
    ports:
      - "5601:5601"
      - "9200:9200"
      - "5044:5044"
      - "5000:5000"
    volumes:
    - ./30-output.conf:/etc/logstash/conf.d/30-output.conf
    - .:/app
```

### `Rails`

logstash에 손쉽게 로그를 먹이기 위해서 로그 출력 형식을 변환해둘 필요가
있으므로 다음을 설치합시다.

```
gem 'lograge'
gem 'logstash-event'
```

그리고 이를 사용할 수 있도록 설정(여기에서는 `development.rb`)을 추가하죠.

```ruby
config.lograge.enabled = true # lograge를 활성화
config.lograge.formatter = Lograge::Formatters::Json.new # JSON 형식으로 로그를 출력
# docker상에서 rails web_console를 사용하는 경우에 발생하는 경고를 처리
config.web_console.whitelisted_ips = '0.0.0.0/0'
```

### `Logstash`

마지막으로 로그를 어떻게 처리할지 지정하는 설정 파일입니다.
`/app/log/development.log`를 읽어와서 모종의 가공처리를 하고 ElasticSearch로
전송합니다.

```
input {
  file {
    path => "/app/log/development.log"
    start_position => beginning
    codec => multiline {
      pattern => "^(\s{2}|[\w_-]+\s\([0-9.]+\))"
      what => "previous"
    }
  }
}

filter {
  json {
    source => "message"
  }
}
 
output {
  elasticsearch { hosts => ["localhost"] }
  stdout { codec => rubydebug }
}
```

이는 에러메시지에 대한 처리입니다.
서버 에러가 발생하는 경우 lograge에서 JSON 형식으로 가공이 되지 않기 때문에
별도의 처리가 필요합니다. Logstash는 한 줄을 로그 하나라고 인식하기 때문에,
수십줄에 이르는 에러 메시지는 각각 별개로 처리되며, 심지어 여기에서는 json이라고
추측하고 있기 때문에 이 메시지들은 전부 정상적으로 처리되지 않습니다.

여기서 사용하는 것이 multiline이라는 코덱입니다. 특정 정규식에 매칭하는 로그가
들어오는 경우, 이전 로그의 뒤에 붙여줍니다. Rails의 에러 메시지의 경우 공백이
2개 들어있는 빈 줄이 하나, 그 뒤로 에러 메시지와 라이브러리 이름/버전으로
시작하는 스택 트레이스를 제공해주기 때문에 이를 이용한 정규식을 사용했습니다.

그리고 이 에러 메시지는 정상적으로 해석되지 않으므로 filter에서 메시지 전체를
source라는 속성에 넣어서 ElasticSearch까지 무사히 도달할 수 있도록 해줍니다.

### Done

설정은 끝입니다.

- 로컬에서 Rails 개발 서버를 실행하고
- `docker-compose up`를 실행하고
- `localhost:5601`으로 접속하면 Kibana를 보실 수 있습니다.
- 인덱스를 지정하고(로그를 하나 이상 전송해야 합니다), 이것저것 만져보면 됩니다.

## Conclusion

이상을 거쳐서 여러가지로 만져본 결과, 느낀 점입니다:

- ElasticSearch의 옵션을 손대는 것이 매우 귀찮다.
- Kibana는 꽤 강력한 도구지만 Aggregation에 대한 이해가 없다면 Dashboard는 커녕 차트 만드는 것조차 한세월 걸릴 것 같다.
- Logstash로 로그를 보내고, 가공하는게 생각보다 쉽지 않다(여러가지 방법이 있지만 잘 동작하는건 생각보다 적더라).
- Multi-Thread 상황에서 예상치 못한 형식의 로그에 대한 처리 방법을 고민해야할 필요가 있어보임. e.g. multiline codec

꽤 벽이 많긴 한데, 서버 환경이나 애플리케이션 튜닝에서 꽤 도움이 될 것처럼
보입니다. 예를 들어 에러를 발생시킨 특정 사용자의 로그를 추적해서 에러 상황을
재현한다든가, 특정 동작에 대한 성능 평가라든가.
개인적으로 매력을 느꼈던건 nginx/unicorn/app 로그 전체를 한번에 정리해서 볼 수
있겠다는 부분이었네요. 외부 서비스를 사용하게 되면 이런 부분에 대한 로그 확인도
안되고, 비교해볼 수도 없어서...

아무튼 전체적으로 이런 느낌이었네요. 기회가 되면 Fluentd도 좀 설정해봐야.

