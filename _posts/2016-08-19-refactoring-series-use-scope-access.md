---
layout: post
title: "Refactoring Series - use scope access"
date: 2016-08-19 12:27:15 +0900
categories:
---

가끔은 의식적으로 이유를 붙여가며 리팩토링해보는 것도 좋은 경험이 될 것 같아서 레포트 형식으로 작성하는 시리즈입니다. 소재는 여기저기서 제가 보던 코드를 가져오고 있으며, 물론 그대로 가져오진 않고 필요한 부분만 가져다가 새로 예제로 만듭니다. 필자가 게으르기 때문에 비정기로 연재될 예정입니다. 의견 및 오타 제보는 언제나 감사하게 받고 있습니다. 😆

## 배경

최근에 할 일 관리를 위해서 새 레일스 프로젝트를 진행하고 있습니다. 여기에서는 ~~당연하지만~~ 현재 로그인한 사용자만이 자기 자신의 할 일만을 CRUD할 수 있습니다. 이 명세를 평소대로 구현하고 보니 Rails Best Practice에서 잔소리를 하는 것을 발견했습니다.

## 문제점

코드는 대략 이랬습니다.

```ruby
class TasksController < ApplicationController
  before_action :set_task, only: :show
  before_action :permission_check, only: :show
  
  def show
  end
  
  private
  
  def set_task
    @task = Task.find(params[:id])
  end
  
  def permission_check
    raise User::NoPermissionError \
      unless @task.user_id == current_user.id
  end
end

class User
  # ...
  NoPermissionError = Class.new(StandardError)
end
```

필요한 액션에서 권한 체크라는 `before_action`을 추가하고, 해당하는 `Task`의 소유자가 현재 로그인한 유저가 아니라면 에러를 던집니다. 이 에러는 `ApplicationController`에서 잡아서 권한이 없다는 에러 메시지와 함께 사용자를 `root_path`로 리다이렉트 해줍니다. 지금 중요한 건 이 부분이 아니니 넘어가죠.

아무튼, Rails Best Practice 왈, 스코프를 사용하랍니다.

```
If you check the permission by comparing the owner of object with current_user,
it's verbose and ugly, you can use scope access to avoid this.
```

과연, 이제와서 생각해보면 좀 그렇습니다. 묘하게 길고, 이 작업 때문에 `before_action`을 두개나 걸어야 하고, 오만가지 생각이 머리 속을 스칩니다. 그래서 어떻게 스코프를 활용하라는 걸까요? 제시하는 코드는 이렇습니다.

```ruby
@post = Post.find(params[:id])
if @post.user != current_user
  flash[:warning] = 'Access denied'
  redirect_to posts_url
end
```

위와 같은 코드를,

```ruby
# raise RecordNotFound exception (404 error) if not found
@post = current_user.posts.find(params[:id])
```

이와 같이 개선할 수 있답니다. 실제 코드를 보니 깔끔하네요. 이와 같이 코드를 작성하게 되면 에러 처리에 대한 부분을 한 곳에서 모아 깔끔하게 정리할 수 있고, 코드는 읽기 쉬워진다는 장점이 생깁니다. 무엇보다 `Task`에서 가져온 할 일을 사용자의 것인지 확인하는게 아니라, 현재 사용자의 할 일을 가져오는 것으로 바뀐 덕분에, 초점이 할 일에서 사용자 중심으로 바뀐 부분이 마음에 듭니다. 그러면 이 규칙을 코드에 적용해보죠.

## 개선하기

### `before_action`을 하나로 뭉치기

이 리팩토링의 초점은 현재 라우팅에서 요구하는 `Task`가 현재 사용자의 것이 맞는지 아닌지 판단하는 로직을 컨트롤러에서 SQL로 옮기는 것입니다. 하지만 지금 코드에서는 `set_task`에서 일단 가져오고, `permission_check`에서 권한을 확인하고 있습니다. 이래서는 작업이 불편해지니 하나의 `before_action`으로 합쳐보죠.

```ruby
class TasksController < ApplicationController
  before_action :set_task, only: :show
  
  def show
  end
  
  private
  
  def set_task
    @task = Task.find(params[:id])
    raise User::NoPermissionError \
      unless @task.user_id == current_user.id
  end
end

class User
  # ...
  NoPermissionError = Class.new(StandardError)
end
```

하나로 뭉쳤습니다. 이번엔 스코프를 사용하도록 코드를 좀 고쳐보죠. 이 시점에서 우리가 원하는 에러는 `RecordNotFound`가 아니니까 `find`를 직접 사용할 수는 없습니다. 그러니 검색하지 못하더라도 에러를 던지지 않는 `find_by_id`를 사용해보죠.

```ruby
class TasksController < ApplicationController
  before_action :set_task, only: :show
  
  def show
  end
  
  private
  
  def set_task
    @task = current_user.tasks.find_by_id(params[:id])
    raise User::NoPermissionError if @task.nil?
  end
end

class User
  has_many :tasks
  # ...
  NoPermissionError = Class.new(StandardError)
end
```

얍. Rails Best Practice가 요구하는 리팩토링은 완료되었습니다. 그런데 막상 읽다보니 이런 생각이 듭니다. 

```
권한이 없는 사용자에게 자신의 것이 아닌 할 일은 없는 거랑 마찬가지 아닌가?
굳이 권한이 없다는 에러를 줘야하나?
```

왠지 `RecordNotFound`로 충분할 것 같다는 생각이 마구 들기 시작합니다. 기획팀에게 문의해보지 않으면...은 나잖아? 사양을 변경합시다. XD

```ruby
class TasksController < ApplicationController
  before_action :set_task, only: :show
  
  def show
  end
  
  private
  
  def set_task
    @task = current_user.tasks.find(params[:id])
  end
end

class User
  has_many :tasks
  # ...
end
```

이제 커스텀 에러가 아닌 `RecordNotFound`를 던지게 되었으니, `find`를 쓰게끔 고쳤습니다. `set_task`가 한 줄로 돌아오고, 권한 체크가 깔끔해져서 우리 모두~~제~~가 행복해졌습니다. 

## 결론

너무 당연히 습관적으로 짜고 있던 코드에서도 줄일 곳은 나오는 법이네요[..]
특히 권한 문제는 대부분의 웹 서비스에서 볼 수 있으니 더 주의깊게 살펴 보는 것도 좋겠습니다.

## 참고자료

관련 [스타일 가이드](http://rails-bestpractices.com/posts/2010/07/20/use-scope-access/)와 이 코드 전체가 포함되어 있는 [PR](https://github.com/riseshia/gannbaruzoi/pull/6/files)입니다. 커밋 전에 변경한 부분이라서 구체적인 변경 순서를 확인할 수는 없지만, 해당 부분은 포스팅으로 충분하리라 생각합니다.
