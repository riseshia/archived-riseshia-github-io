---
layout: post
title: "Go Tutorial"
date: 2016-11-19 19:37:35 +0900
categories:
---

[CodeSchool](https://www.codeschool.com/)에서 3일간 전 코스를 무료로 공개하는 이벤트가
진행중(작성 시점)입니다. 이 글은 그 중에서 Golang에 대한 튜토리얼(On Track With Golang)을
수강하며 정리한 노트입니다.

## Go?

구글이 2007년에 만든 오픈소스 언어이며, 간단하고 효율적인 코드를 작성할 수 있게 해줌.

- 실행 가능한 파일로 컴파일함
- 정접 타입 언어
- 동시성 모델이 내장되어 있음
- 쉬운 배포
- 작성하기 즐거움

## 시스템 프로그래밍

Go가 좋은 선택이 될 수 있음

- 사람과 직접 상호작용 -> Application Program
- 프로그램과 상호작용 -> System Program

## Tips

- `src` 가 루트 소스 폴더.
- `main.go`부터 시작하는 게 관습.
- 모든 프로그램은 `main` 패키지와 `func main`이 있어야 함.
- `go build`: 실행 가능한 파일을 생성.
  - 빌드 결과물은 프로젝트명과 동일한 이름의 실행 파일.
- `go run`: 빌드 & 실행을 동시에. 컴파일 결과가 출력되지 않음.
- `gofmt -w file`: 코드를 포매팅.
  - `-w`는 결과를 다시 소스 파일에 피드백.
- 실행시 넘기는 인자는 `os.Args`로 가져올 수 있음(`os` 패키지).
  - 인덱스 0은 실행된 파일 경로.
- `goimports`: 코드에서 사용된 외부 패키지 중, 명시적으로 선언되지 않은 것을 자동으로 추가.
- `import`: 여러 패키지를 사용하는 경우 `()`로 감싸고, 각 패키지 명은 개행으로 구분.
- `os.Exit(int)`로 프로세스를 즉시 종료할 수 있음.
- 할당하지만 사용하지 않는 경우는 `_`라는 이름으로 사용하지 않음을 명시적으로 표기.

## 조건문

- `if`의 조건을 감싸는 `()`가 없음.
- `if`, `else if`, `else`를 사용.

## 변수

### Type Inference

```go
hoge := "string"

// 위와 아래는 동일한 결과

var hoge string
hoge = "string"
```

- `:=`로 타입 추론을 사용할 수 있음.
- 우측의 평가값의 타입을 검출하고 좌측의 변수 선언에 사용.
- 명시적으로 매번 타입을 선언 할 필요 없이 go에서 알아서 추론해줌.
- 선호되는 방식.


### Manual Type Declaration

- `var <변수명> <타입>`: 변수 선언.
- `<이름> = <값>`: 변수 할당.

### Common Data Type

- int
- string
- bool
- []string
- ...

## 함수

```go
func <method signiture> <return type> {
}

// 예를 들자면,
func hoge(name string) bool {
  return false
}

```

## 다중 반환

```go
func hello() (string, int) {
  return "hello", 0
}

result, state := hello()
```

### Zero values

- 선언하는 시점에 기본값으로 할당되는 값.
- 타입별로 다른 값이 사용됨.
  - string: ""
  - int: 0
  - bool: false
  - function: nil
  - ...

## 반복문

- `for`만 제공함.

```go
for <init>; <condition>; <post> {}
for <condition> {}
for {}
```

## Array & Slices

### Array

- 수동 타입 선언시에는 크기를 지정해주어야 함. ex) `var langs [3]string`
- 동적이지 않음.
- 배열을 넘기는 경우, 함수 시그니처에도 크기가 명시되어야 함

```go
func main() {
  var arr [3]string

  hoge(arr)
}

func hoge(arr [3]string) bool {
  // ...
}
```

### Slices

- 배열의 위에서 동작.
- 선언시에 크기 지정할 필요가 없음.
- `append`를 사용하여 값을 추가.
- `[]string{"a", "b", "c"} 와 같은 리터럴을 사용하여 초기화 가능.
- 인덱스도 배열과 동일한 방식으로 사용.
- 슬라이스를 위한 for 문법이 있음.
  - 첫번째는 인덱스, 두번째는 해당 인덱스의 값.

```go
for i := range arr {
  // i will be index
  arr[i]
}

for _, element := range arr {
  el
}
```

## Struct

```go
type <struct name> struct {
  <attr_name> <attr_type>
  // ...
}

func (<variable name> <struct name>) <method signiture> <return type> {
  // ...
}

func main() {
  // init struct
  ins := <struct name>{attr1: value, attr2: value...}
}
```

- 선언은 `main()` 밖에서, 쓰는건 반드시 안에서.
- struct에 함수를 연결할 수 있음.
- "Tell, Don't Ask" 원칙.
- 구조체는 값으로 넘겨진다.
  - 원한다면 `&`연산자를 사용해서 레퍼런스를 넘길 것.

## Interface

```go
type <interface name> interface {
  <method signiture> <return type>
}
```

- 접미사로 정의하는 메소드 + `er`을 사용.

## Package

- `src/<project>/<package name>/main.go`를 사용.
- 외부에 노출해야하는 것의 이름은 대문자로 시작할 것.
- `import "<project>/<package>"`
- 패키지 이름을 통해서 접근 가능.

## goroutine

- 동시성 및 병렬 처리 지원.
- 다른 함수들을 동시(concurrently)에 실행.
- `go` 예약어를 사용하여 함수를 호출 => 비동기 처리
- 메인 함수랑은 별개로 동작 -> 결과를 받기위한 방법(`sync.WaitGroup`)이 필요
- `Add` -> `Wait`으로 블럭을 잡고, `go`가 호출되는 함수 내부에서 `Done`을 호출해서 함수 종료를 전달
- 실행시에 `GOMAXPROCS` 환경 변수로 사용할 프로세서 갯수를 지정 가능

```go
// task의 정의
func (e Hoge) task(*wg sync.WaitGroup) {
  // some logic...
  wg.Done()
}

// 실행 코드
wg := sync.WaitGroup

wg.Add(number)

for _, el := range arr {
  go arr.task(&wg)
}

wg.Wait()
```

## Conclusion

- 문법이 명쾌해서 마음에 들었음.
- goroutine의 사용법이 단순함.
- 구조체가 값으로 넘어가는걸 보면 불변성에 대한 집착을 엿볼 수 있었음.
- C-like 라는 이유도 어느 정도 이해함.
