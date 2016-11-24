---
layout: post
title: "Secret with Travis"
date: 2016-11-25 00:01:33 +0900
categories:
---

Travis에서 배포나 푸시, 웹훅 등을 설정하고 싶을 때가 있습니다.
특히 공개 저장소인 경우에는 민감한 정보를 어떻게 저장할 지에 대한 고민이 생깁니다.
이러한 경우를 위해서 몇가지 방법이 Travis에서 제공되고 있습니다. 여기에 대해서 알아보죠.

## Secure Environment Variable

### Setting on Repository

저장소 설정을 통해서 암호화된 환경 변수를 설정할 수 있습니다.

Travis 상의 저장소에 접속 -> Setting -> Environment Variable

이 메뉴에서 키와 값을 입력하고, `Display value in build log` 옵션을 `OFF`로
설정(기본값)하면 됩니다.
실수로 `ON`으로 설정하면 로그에 그대로 값이 출력되는 것을 볼 수 있습니다.

[Reference](https://docs.travis-ci.com/user/environment-variables/#Defining-Variables-in-Repository-Settings)

### Defining encrypted variables in `.travis.yml`

또는 `.travis.yml` 파일에 직접 값을 저장할 수도 있습니다.
물론 암호화해서 말이죠.
`travis` 젬을 설치하고, 로그인합시다.

```ruby
travis encrypt KEY=value --add env.global
```

이를 실행하면 `.travis.yml`의 `env.global`에 `KEY` 값을 가져오기 위해서
사용되는 값이 추가됩니다.

```patch
diff --git a/.travis.yml b/.travis.yml
index 1a416a4..da92070 100644
--- a/.travis.yml
+++ b/.travis.yml
@@ -6,6 +6,7 @@ env:
   global:
+  - secure: hLdkN0a0p5hZ+vCzmGFeZmCG1q/z+pJm/pwp+0mS+9fxLtjwAKTWfuDDkZJACM+Qv8vqgKC8TnIvZNoW+9qM3GpKLJLLWD5lDe8JWJ3/N3XQRJ4dw8MvQm9pDc0/UZ4KzdzofaB+FpqZzYuXl2f4oDEbxH54WNewV+s5/fa5vVewnMVFJRbOLpXIDHJZv1ZPLj2WtBCUXOCqlq4lCP0jk52Qs0TNn9o5LMPWrQF61Mk8p3kigzIO3+QkzSYqYOSNpBYB/EHvtDFe73Jomqx0gZjnk1ZnASndjbR+V7uQIq9CGgYPjbOMSkKYU0QzxKRHDQeqqOzq11t+Ba5LWwxJlNCr9B4z5bjqAoImBBIIWk5xAaZt9lYocG2kR108SdWKaejCGc8H+lzKrhivxVBrrKcsG6PyZw3mxsdwx+B0F8AfI2bNtGkTGR5CEBuitGsEiMBdhMOOw9PZXIPB9gkiMXMHM7RRNBJJYC/fg2Prj3uOqAFobQr2ShLQ0rSHtr53DMNqVC36IKAavM7FtKgpxhxWmcxu84EuRMisKVXaSxmFlGSLqoXWTfQT2M/HfPTXjsiAn+kU50MA/XcYZDSuvtG5JOOORRJ/m5TytIjjDJg0bUJbieuKT/ri3LFqtceA6sFCtcXxkmsjG2Txc6I9Y3W+ru5WLvRfoDjLGf1UnFs=
 before_install:
 - git config --global user.email "travis@some.io"
 - git config --global user.name "Travis CI"
```

이 파일을 그대로 커밋하시면 CI에서 `KEY`라는 이름으로 `value`라는 값을
가져올 수 있게 됩니다.

[Reference](https://docs.travis-ci.com/user/environment-variables/#Defining-encrypted-variables-in-.travis.yml)

## Encrypt File

### 준비

- `travis` 젬을 설치하고 로그인된 상태일 것
- `travis` 젬의 버전이 `1.7.0` 이상일 것

### 암호화하기

`super_secret.txt`이라는 파일이 필요하지만 커밋해서는 안되는 상황이라고
가정해봅시다. 이때 `travis encrypt-file`을 사용할 수 있습니다.
이 명령을 사용하면 해당 파일을 `AES-256` 알고리즘을 사용하여 암호화합니다.

```bash
$ travis encrypt-file super_secret.txt
encrypting super_secret.txt for user/repo
storing result as super_secret.txt.enc
storing secure env variables for decryption

Please add the following to your build script (before_install stage in your .travis.yml, for instance):

    openssl aes-256-cbc -K $encrypted_81808ddf186a_key -iv $encrypted_81808ddf186a_iv -in super_secret.txt.enc -out super_secret.txt.rb -d

Pro Tip: You can add it automatically by running with --add.

Make sure to add super_secret.txt.enc to the git repository.
Make sure not to add super_secret.txt.rb to the git repository.
Commit all changes to your .travis.yml.
```

Travis-CLI가 참 친절합니다. 알려주는대로 `.travis.yml`의 어딘가에 암호화된
파일을 복원하는 명령을 추가해보죠.

```bash
openssl aes-256-cbc -K $encrypted_81808ddf186a_key -iv $encrypted_81808ddf186a_iv -in super_secret.txt.enc -out super_secret.txt.rb -d
```

그리고 암호화된 파일 `super_secret.txt.enc`을 커밋합시다.
당연하지만 **`super_secret.txt`를 커밋해서는 안됩니다.**

[Reference](https://docs.travis-ci.com/user/encrypting-files/#Automated-Encryption)
 
## Caption

- 신뢰할 수 없는 빌드(다른 저장소로부터의 PR)인 경우에는 보안상의 이유로 이러한 환경 변수들을 사용하지 않고 무시합니다.
- `.travis.yml`과 저장소 설정에 같은 이름의 변수를 정의하는 경우 전자의 환경 변수를 우선합니다.

## Conclusion

이상으로 Travis에서 민감한 정보를 처리하는 방법에 대해서 알아보았습니다.
더 자세한 설명이 필요하신 경우에는 각 방법 밑에 달아둔 Reference를 참고해주세요.

[System: 공개 저장소에서도 웹훅을 던질 수 있는 스킬을 획득하셨습니다! XD]

