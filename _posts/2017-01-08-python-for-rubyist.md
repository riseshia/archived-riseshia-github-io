---
layout: post
title: "Python tutorial for Rubyist"
date: 2017-01-08 18:35:19 +0900
categories:
---

이 글은 [파이썬 3에 뛰어들기](https://codeonweb.com/course/5c550b25-9638-4d0f-8043-97ac01415f62)를 읽으며 배운 문법에 대해서 정리했습니다.
평범하게 정리하면 재미가 없기 때문에, 루비 개발자가 보았을때 낯설어보일 수 있는
부분을 중점적으로 정리해봤습니다.

## Function argument

- 인자는 항상 키워드 인수로 취급됩니다.
- 인수명을 지정한 경우, 그 뒤에 나오는 모든 인수에 이름을 붙여야 합니다.

```python
# Success to call
approximate_size(4000, a_kilobyte_is_1024_bytes=False)
approximate_size(size=4000, a_kilobyte_is_1024_bytes=False)
approximate_size(a_kilobyte_is_1024_bytes=False, size=4000)

# SyntaxError
approximate_size(a_kilobyte_is_1024_bytes=False, 4000)
approximate_size(size=4000, False)
```

## Docstring

- HEREDOC
- 따옴표 3개를 합니다.
- 따옴표 3개를 사용한 줄에서부터 시작하고, 3개를 다시 한번 사용한 줄에서 끝난다. 다시말해 아래와 작성하면 앞뒤로 개행문자를 하나 가지는 문자열이 됩니다.
- 문서화에도 사용합니다.

```python
document = '''
query {
  task
}
'''
document # => '\nquery {\n  task\n}\n'
```

## Ternary Operator

파이썬에는 일반적으로 많이 사용하는 삼항 연산자가 없습니다. 대신 다음과 같이 작성할 수는 있습니다.

```python
value = True if condition else False
```

## `in`

```python
some_list = ['el', 'em', 'ent']
'el' in some_list # => True

## 열거 가능
for e in some_list:
    print(e)
```

## List

- `some_list[0:3]`는 뒤에 지정된 인덱스 앞까지만을 반환합니다(여기에서는 3개의 원소를 돌려준다).
- Python의 `list.extend`는 Ruby의 `array.concat`
- `list.index`는 탐색에 실패한 경우 에러를 던집니다.
- `list.remove(value)` or `del list[idx]`로 값을 삭제 가능합니다.

## Tuple

- Immutable List
- 리스트보다 성능이 좋습니다.
- 읽기 전용이므로 방어적 코딩에 적합합니다.
- 딕셔너리의 키로 사용할 수 있습니다.
- 예약 기호가 소괄호라서 여러가지로 귀찮습니다.
- 값이 하나만 있는 경우 반드시 뒤에 쉼표를 붙이세요.

```python
one_element_tuple = (1,)
some_tuple = (1, 2, 3)
some_tuple.append(4)
## AttributeError: 'tuple' object has no attribute 'append'
```

## Set

- 예약 기호가 중괄호라서 여러가지로 귀찮습니다.
- `set.discard`는 silence deletion, `set.remove`는 존재하지 않는 값인 경우 에러를 던집니다.

```python
a_set = {1}
a_set = set([1])
it_is_dict = {}
it_is_set = set()
```

## Comprehension

- 열거 -> 가공 -> 반환을 한번에 할 수 있는 표현 방식입니다.
- 뒤에 `if`를 붙이면 필터링도 가능합니다.
- 딕셔너리의 경우, `items()`를 통해 뷰를 받아야 합니다.

```python
a_list = [1, 2, 3, 4]
[elem * 2 for elem in a_list] # => [2, 4, 6, 8]

a_set = {1, 2, 3, 4}
{elem * 2 for elem in a_set} # => {8, 2, 4, 6}

a_dict = {'a': 1, 'b': 2, 'c': 3, 'd': 4}
{key:value * 2 for key, value in a_dict.items()} # => {'a': 2, 'b': 4, 'c': 6, 'd': 8}
```

## String

- interpolation: [자세한 설명은 생략합니다](https://pyformat.info). 나중에 생각나면 한번 정리할지도.
- Python 3.x는 소스의 기본 인코딩이 UTF-8입니다.

## Regexp

- 기본으로 이스케이프 되므로 정규식을 사용할 때에는 raw string을 권장합니다.

```python
s = '100 BROAD'
re.sub('\\bROAD$', 'RD.', s)
re.sub(r'\bROAD$', 'RD.', s)
```

- docstring을 사용하여 주석을 처리할 수 도 있습니다. 단 이를 알려주기 위한 옵션을 넘겨야 합니다.

```python
pattern = '''
    ^                   # 문자열의 시작
    M{0,3}              # 천의 자리 - 0-3 개의 M
    (CM|CD|D?C{0,3})    # 백의 자리 - 900(CM), 400(CD), 0-300(0-3 개의 C),
                        #           500-800(D 하나와, 뒤이은 0 to 3 개의 C)
    (XC|XL|L?X{0,3})    # 십의 자리 - 90(XC), 40(XL), 0-30(0-3 개의 X),
                        #           50-80(L 하나와, 뒤이은 0-3 개의 X)
    (IX|IV|V?I{0,3})    # 일의 자리 - 9(IX), 4(IV), 0-3(0-3 개의 I),
                        #           5-8(V 하나와 뒤이은 0-3 개의 I)
    $                   # 문자열의 끝
    '''
re.search(pattern, 'M', re.VERBOSE)
```

## Generator

- 문법 레벨에서 제너레이터를 지원합니다.
- `yield`는 코드블럭을 실행하는 것이 아니라 제너레이터를 위한 예약어입니다.
- 제너레이터 표현식이라는 것이 있습니다. Comprehension과 유사하지만 소괄호를 사용합니다.

```python
def fib(max):
    a, b = 0, 1
    while a < max:
        yield a
        a, b = b, a + b

[f for f in fib(10)]
list(fib(10))

# Generator Expression
unique_characters = {'E', 'D', 'M', 'O', 'N', 'S', 'R', 'Y'}
gen = (c for c in unique_characters)

next(gen) # => 'E'
next(gen) # => 'D'
```

## Class

- `__init__`는 생성자입니다.
- 인스턴스 메소드에는 `self`라는 인수로 자기 자신이 넘어옵니다. 이 변수명은 임의이지만 대부분의 개발자가 `self`를 사용합니다.
- 클래스에 직접 인자를 넘기면 인스턴스를 생성할 수 있습니다.
- private method가 문법 레벨에 존재하지 않습니다. 관습적으로는 `__` 접두사를 사용합니다.

```python
class AClass:
    def __init__(self, attr):
        self.attr = attr

    def this_is_an_instance_method(self):
        return 'Hoge'

    def this_is_a_class_method():
        return 'Hige'

    def __this_is_a_private_instance_method(self):
        pass

ins = AClass('some attr')
ins.this_is_an_instance_method()
AClass.this_is_a_class_method()
```

## Iterator

- `__iter__`와 `__next__`를 사용해서 구현됩니다.
- 각각은 `iter`와 `next`에서 호출됩니다.
- 열거를 멈추고 싶은 경우에는 `StopIteration`를 던져주세요.

```
class Fib:
    def __init__(self, max):
        self.max = max

    def __iter__(self):
        self.a = 0
        self.b = 1
        return self

    def __next__(self):
        fib = self.a
        if fib > self.max:
            raise StopIteration
        self.a, self.b = self.b, self.a + self.b
        return fib

for n in Fib(1000):
    print(n, end=' ')
## 0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987
```

## Etc

- 들여쓰기는 문법입니다.
- 콜론으로 블럭을 구분합니다.
- `try`, `except`, `raise`라는 예외 문법을 사용합니다.
- `true`, `false`가 아닌 `True`, `False`입니다.
- 형변환하면 `True` -> 1, `False` -> 0입니다.
- `//`는 나눈 뒤 소수점 이하를 버립니다.
- `/`는 실수형만을 반환합니다.
- `nil` -> `None`
- 빈 메소드/클래스에는 `pass`라고 적어야 합니다.
