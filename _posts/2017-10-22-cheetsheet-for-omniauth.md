---
layout: post
title: "OmniAuth + Rails CheetSheet"
date: 2017-10-22 12:25:36 +0900
categories:
---

Rails에서 Devise에 의존하지 않고 OmniAuth만으로 OAuth2 인증을 처리하는 순서에
대해서 간단하게 정리합니다. 어쩌다 한번씩 하는 작업이라 자꾸 뭔가 하나씩
빼먹게 되서 체크리스트 대용으로 사용할 겸.

## 설명환경

- Ruby 2.4.2
- Rails 5.1.2

## 일단 깐다

```ruby
# Gemfile

# ex: gem 'omniauth-github'
gem 'omniauth-<provider>'
```

```bash
bundle install
```

## OmniAuth::Builder 설정

```ruby
# config/initializers/omniauth.rb

Rails.application.config.middleware.use OmniAuth::Builder do
  provider <provider>, ENV['AUTH_KEY'], ENV['AUTH_SECRET'], scope: 'some_scope'
end
```

기본적으로는 이렇고, 때때로 추가 옵션을 넘기는 경우가 있음. i.e. slack, github with ghe

## 모델 준비

```bash
bin/rails g model User provider:string uid:string
bin/rails g model SecureToken user:references token:string
```

- 필요한 필드를 추가(i.e. email)
- 제너레이터로는 null, limit 지정이 안되므로 필요하다면 migration 파일을 만질 것.
  - 개인적으로는 `null: false`는 필수. 길이는 대략 20~40쯤. 단 토큰은 50 이상을 지정.
- user 테이블에는 `provider`, `uid`에 복합 인덱스를 지정한다.
  `add_index 'users', %w[provider uid], name: 'user_uniq_provider_uid', unique: true, using: :btree`

### User 실장

```ruby
# app/models/user.rb

class User < ApplicationRecord
  has_one :secure_token, dependent: :destroy

  delegate :token, to: :secure_token, allow_nil: true

  def self.sign_in!(auth)
    find_or_initialize_by(provider: auth.provider, uid: auth.uid).tap do |user|
      if user.persisted?
        user.secure_token.update!(token: auth.credentials.token)
      else
        # user.assign_attributes(name: auth.info.user.name)
        user.build_secure_token(token: auth.credentials.token)
        transaction { user.save! }
      end
    end
  end
end
```

### SecureToken 실장

```ruby
# app/user/secure_token.rb

class SecureToken < ApplicationRecord
  belongs_to :user
end
```

자동 생성된 대로 방치.

## SessionController 준비

### Routing

```ruby
# config/routes.rb

Rails.application.routes.draw do
  # ...

  get 'sign_in', to: 'session#new', as: 'sign_in'
  delete 'sign_out', to: 'session#destroy', as: 'sign_out'

  get 'auth/:provider/callback', to: 'session#create'
  get 'auth/failure', to: 'session#failure'
end
```

OmniAuth에서 사용하는 라우트는 3개

- `auth/:provider`로 해당하는 OAuth2 제공자로 리다이렉션 -> OmniAuth에서 처리
- `auth/:provider/callback`가 OAuth2 redirect_uri
- `auth/failure`는 인증 도중에 문제가 생기면 이쪽으로 넘어감

### Controller

```ruby
# app/controllers/session_controller.rb

class SessionController < ApplicationController
  skip_before_action :require_sign_in, only: %i(new failure create)

  def new; end

  def failure; end

  def create
    auth = request.env['omniauth.auth']
    user = User.sign_in!(auth)
    session[:user_id] = user.id
    redirect_to root_path
  end

  def destroy
    session[:user_id] = nil
    redirect_to root_path
  end
end
```

### ApplicationController

```ruby
# app/controllers/application_controller.rb

class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception

  before_action :require_sign_in

  private

  def require_sign_in
    redirect_to sign_in_path unless session[:user_id]
  end

  def current_user
    User.find(session[:user_id])
  end
end
```

### View

```erb
# app/views/session/new.html.erb

<%= link_to 'Login with Your provider', '/auth/your_provider', data: { turbolinks: false } %>
```

```erb
# app/views/session/failure.html.erb

Auth Failure

<p>Message: <%= params[:message] %></p>
<p>Origin: <%= params[:origin] %></p>
<p>Strategy: <%= params[:strategy] %></p>
```

### Sign out

```erb
<%= link_to 'sign out', sign_out_path, method: :delete %>
```
