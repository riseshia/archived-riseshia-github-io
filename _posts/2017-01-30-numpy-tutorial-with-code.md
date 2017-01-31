---
layout: post
title: "코드로 보는 NumPy Tutorial"
date: 2017-01-30 22:20:52 +0900
categories:
---

NumPy는 파이썬에서 계산을 위해서 사용하는 가장 유명한 라이브러리라고 해도
과언이 아닙니다(사실 이거 말고 아는게 없습니다).
그 중에서 행렬 연산에 관한 기능을 코드와 함께 정리해봤습니다.

### 생성하기

```python
A = np.array([1.0, 2.0, 3.0])
#=> [1. 2. 3.]

type(A)
#=> <class 'numpy.ndarray'>

np.arange(10)
#=> array([0, 1, 2, 3, 4, 5, 6, 7, 8, 9])
np.arange(0, 10, 2)
#=> array([0, 2, 4, 6, 8])

np.ones((2, 3), dtype=int)
# array([[1, 1, 1],
#        [1, 1, 1]])

np.random.random((2, 3))
# array([[ 0.24595208,  0.27814662,  0.5671275 ],
#        [ 0.56680138,  0.97115287,  0.22724825]])

def f(x, y):
    return 10*x + y
A = np.fromfunction(f, (2, 4), dtype=int)
# array([[ 0,  1,  2,  3],
#        [10, 11, 12, 13]])
```

- 기본 array처럼 열거 가능.
- 모든 원소를 별도로 열거하려면 `flat`을 사용.
- 왜 `numpy.array`가 아니고 `numpy.ndarray`였을까.

## 사양 확인

```python
M = np.array([[1, 2], [3, 4]])
# array([[1, 2],
#        [3, 4]])
M.shape
#=> (2, 2)
M.dtype
#=> dtype('int64')
```

- 특이하게 데이터 타입이 존재한다.

## 연산하기

### elementwize

```python
X = np.array([1.0, 2.0, 3.0])
Y = np.array([2.0, 4.0, 6.0])

X + Y
#=> array([ 3.,  6.,  9.])
X - Y
#=> array([-1., -2., -3.])
X * Y
#=> array([  2.,   8.,  18.]) 
X / Y
#=> array([ 0.5,  0.5,  0.5])
X * 2
#=> array([ 2.,  4.,  6.])
```

- 기존 연산자를 사용하면 각 요소들을 1:1로 매칭해서 연산한다.
- 스칼라 값을 사용하면 각 요소들에 대해서 계산하고 돌려준다.
- 이런 방식을 `elementwise`라고 부른다.
- 이처럼 다른 차원의 값 간의 연산을 확장 & 연산해주는 기능을 브로드캐스트라고 부른다.

### 행렬곱

```python
A = np.array([[1, 2], [3, 4]])
B = np.array([[1, 2], [2, 1]])
A.dot(B)
# array([[ 5,  4],
#        [11, 10]])
np.dot(A, B)
# array([[ 5,  4],
#        [11, 10]])
```

### Universal Functions

```python
A = np.random.random((2, 3))
# array([[ 0.06462051,  0.56454357,  0.80359124],
#        [ 0.21630644,  0.05226016,  0.52533345]])
A.sum()
#=> 2.5718191614547998
A.min()
#=> 0.1862602113776709
A.max()
#=> 0.6852195003967595

# axis로 축 방향을 지정할 수도 있음
A.sum(axis=0) # 각 열의 합
#=> array([ 0.28092695,  0.61680373,  1.32892469])
A.min(axis=1) # 각 행의 최소값
#=> array([ 0.06462051,  0.05226016])
aAcumsum(axis=1) # 각 행의 누적합
# array([[ 0.06462051,  0.62916408,  1.43275532],
#        [ 0.21630644,  0.2685666 ,  0.79390005]])
```

- 일반적인 수학 연산 개념(삼각 함수, 지수, 합, 평균, ...)을 제공.
- 각 연산자는 elementwise, 결과값은 행렬로 반환됨.

### 인덱스 연산

```python
A = np.array([[1, 2], [3, 4]])
A[0]
#=> array([1, 2])
A[0][0]
#=> 1

A[np.array([0])]
#=> array([[1, 2]])
A > 1
# array([[False, True],
#        [ True, True]], dtype=bool)
A[A > 1]
#=> array([2, 3, 4])
```

- 일반 배열인것처럼 각 원소에 접근할 수도 있음.
- 배열을 사용하여 요소를 취득할 수 있음.
- 불린 배열인 경우 필터처럼 사용할 수도 있음. 단 결과는 1차원으로 압축된다는 점에 주의.

### dots 연산

```python
A = np.array( [[[  0,  1,  2],
                [ 10, 12, 13]],
               [[100,101,102],
                [110,112,113]]])
A.shape
#=> (2, 2, 3)
A[1, ...]
# array([[100, 101, 102],
#        [110, 112, 113]])
A[1, :, :] # 동치
# array([[100, 101, 102],
#        [110, 112, 113]])
A[1] # 동치
# array([[100, 101, 102],
#        [110, 112, 113]])

A[..., 2]
# array([[  2,  13],
#        [102, 113]])
A[:, :, 2] # 동치
# array([[  2,  13],
#        [102, 113]])
```

`...`은 `:`로 나머지를 채움.

## 변환하기

```python
A = np.arange(12)
#=> array([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11])
B = A.reshape(4, 3) #=> 변환된 행렬을 반환
# array([[ 0,  1,  2],
#        [ 3,  4,  5],
#        [ 6,  7,  8],
#        [ 9, 10, 11]])
A.resize((3, 4)) #=> 자체를 변환함(파괴적)
# array([[ 0,  1,  2,  3],
#        [ 4,  5,  6,  7],
#        [ 8,  9, 10, 11]])
A.flatten()
#=> array([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11])
A.ravel()
#=> array([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11])
A.T # 전치 행렬
# array([[ 0,  4,  8],
#        [ 1,  5,  9],
#        [ 2,  6, 10],
#        [ 3,  7, 11]])
A.shape = 2, 6 # 파괴적
# array([[ 0,  1,  2,  3,  4,  5],
#        [ 6,  7,  8,  9, 10, 11]])
A.reshape(3, -1) #=> 변환된 행렬을 반환
# array([[ 0,  1,  2,  3],
#        [ 4,  5,  6,  7],
#        [ 8,  9, 10, 11]])
```

- -1을 사용하면 멋대로 사이즈에 맞춰서 변환함
- 파괴적인 함수들은 반환값이 없다(여기에서는 변경된 상태를 보여주기 위해 반환값이 있는 것처럼 표기하고 있음).

```python
A = np.array([[1, 2], [3, 4]])
# array([[1, 2],
#        [3, 4]])
B = np.array([[5, 6], [7, 8]])
# array([[5, 6],
#        [7, 8]])

np.vstack((A, B))
# array([[1, 2],
#        [3, 4],
#        [5, 6],
#        [7, 8]])

np.hstack((A, B))
# array([[1, 2, 5, 6],
#        [3, 4, 7, 8]])

A = np.array((1, 2, 3))
B = np.array((2, 3, 4))
np.column_stack((A, B))
# array([[1, 2],
#        [2, 3],
#        [3, 4]])
```

- `column_stack`은 1차원 배열에 대해서만.
- `hstack`은 2차원을 기준으로, `vstack`은 1차원을 기준으로 수행.

```python
A = np.arange(24).reshape(2, 12)
# array([[ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11],
#        [12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23]])

np.hsplit(A, 3) # 열 방향으로 3조각.
# [array([[ 0,  1,  2,  3],
#        [12, 13, 14, 15]]), array([[ 4,  5,  6,  7],
#        [16, 17, 18, 19]]), array([[ 8,  9, 10, 11],
#        [20, 21, 22, 23]])]
np.hsplit(A, (3, 4)) # 3번째 열을 기준으로 쪼갬
# [array([[ 0,  1,  2],
#        [12, 13, 14]]), array([[ 3],
#        [15]]), array([[ 4,  5,  6,  7,  8,  9, 10, 11],
#        [16, 17, 18, 19, 20, 21, 22, 23]])]
```

## 뷰

```python
A = np.arange(12).reshape(2, 6)
# array([[ 0,  1,  2,  3,  4,  5],
#        [ 6,  7,  8,  9, 10, 11]])

C = A.view()
C is A
#=> False
C.base is A # 실 데이터는 A와 동일함
#=> True
C.flags.owndata
#=> False
C.shape = 3, 4
A.shape
#=> (2, 6)
C[0, 3] = 99 # A와 동일한 데이터를 보고 있으므로 A도 변경됨
A
# array([[ 0,  1,  2, 99,  4,  5],
#        [ 6,  7,  8,  9, 10, 11]])
```

- 기본적으로 얕은 복사.
- 뷰를 통해 원 데이터는 그대로 두고 다른 형태의 행렬인것처럼 사용할 수 있음.
- 깊은 복사가 필요하면 `copy()`를 사용할 것.

## Reference

- [Quickstart tutorial](https://docs.scipy.org/doc/numpy-dev/user/quickstart.html)
- ぜロから作るDeep Learning