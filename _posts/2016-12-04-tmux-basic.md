---
layout: post
title: "Tmux basic"
date: 2016-12-04 13:02:53 +0900
categories:
---

[이 책](https://pragprog.com/book/bhtmux2/tmux-2)을 읽으면서 배운 내용을
정리했습니다.

## 세션

- 이름 있는 세션 열기: `tmux new-session -s basic`, `tmux new -s basic`
  - `-d` 옵션을 사용하면 열어서 바로 detach
  - `-s` 세션 이름
  - `-n` 윈도우 이름
- 시계 열어보기: `prefix` + `t`
- Detach: `prefix` + `d`
- Attach: `tmux attach`
  - `-t` 옵션을 사용해서 열 세션 이름을 지정할 수 있음.
- 현재 세션 목록: `tmux list-sessions`, `tmux ls`
- 세션 종료하기:
  - inside session: `exit`, `prefix` + `&`
  - outside of session: `tmux kill-session -t <session_name>`

## 윈도우

- 새 윈도우 열기: `prefix` + `c`
- 윈도우 이름변경하기: `prefix` + `,`
- 다음 윈도우로 이동하기: `prefix` + `n`
- 이전 윈도우로 이동하기: `prefix` + `p`
- `n`번 윈도우로 이동하기: `prefix` + `<n>`. ex) 0번 윈도우라면 `prefix` + `0`
- 윈도우 목록으로 이동하기: `prefix` + `w`
- 이름 검색으로 이동하기: `prefix` + `f` 후 문자열 검색

## 패널

- 좌우로 쪼개기: `prefix` + `%`
- 상하로 쪼개기: `prefix` + `"`
- 포커스 변경하기: `prefix` + `o`, `prefix` + `방향키`

### 기본 레이아웃

보통은 키매핑해서 사용.

- even-horizontal: 패널을 왼쪽에서 오른쪽으로 배치
- even-vertical: 패널을 위에서 아래로 배치
- main-horizontal: 큰 패널을 위에 하나 만들고, 나머지를 아래에 배치
- main-vertical: 큰 패널을 왼쪽에 만들고 나머지를 오른쪽에 배치
- tiled: 모든 패널을 동일한 크기로 배치
- 레이아웃을 변경: `prefix` + `spacebar`
- 패널을 지우기: `prefix` + `x`

## Command Mode

- 실행하기: `prefix` + `:`
- 새 윈도우 만들기: `new-window -n <name>`
- 새 윈도우 만들기 & 프로세스 실행하기: `new-window -n <name> "<cmd>"` => 프로세스가 종료되면 윈도우도 사라진다(설정에서 변경할 수 있음)

## `.tmux.conf`

- global: `/etc/tmux.conf`
- local: `~/.tmux.conf`
- 설정 다시 불러오기: 세션 전부를 재시작 or cmd 모드에서 `source-file ~/.tmux.conf`
- prefix를 `C-a`로 변경하기

```
set -g prefix C-a
```

- 키 바인딩을 제거하기

```
unbind C-b
```

- prefix와 명령 사이의 딜레이 설정하기

```
set -s escape-time 1
```

- 윈도우의 시작 인덱스 값을 변경하기

```
set -g base-index 1
```

- 패널의 시작 인덱스 값을 변경하기

```
# setw 는 윈도우와 관련된 설정 변경시에 사용
setw -g pane-base-index 1
```

- 키 바인딩 하기

```
# prefix는 생략한다.
bind r source-file ~/.tmux.conf \; display "Reloaded!"
```

- None-prefix 키 바인딩

```
# Ctrl - r을 사용
bind-key -n C-r source-file ~/.tmux.conf
```

- 애플리케이션에 prefix 예약 단축키를 전송하기

```
# C-a 를 두 번 누르면 애플리케이션에 C-a가 입력된다.
bind C-a send-prefix
```

- 패널 생성 단축키 변경하기

```
# -h -> 오른쪽에 새 패널 생성
# -v -> 아래에 새 패널 생성
bind v split-window -h
bind s split-window -v
```

- 패널 이동 방향키 변경하기

```
# VIM Style
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R
```

- 윈도우 이동키 변경하기

```
# 왼쪽/오른쪽
# -r -> Repeatable
bind -r C-h select-window -t :-
bind -r C-l select-window -t :+
```

- 패널 리사이징 단축키

```
# Pane resizing panes with Prefix H,J,K,L
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5
```

## Tip

- 이름을 지정하지 않고 새 윈도우를 만들었을 경우, 현재 실행중인 프로세스명을 사용한다.
- `prefix` + `?`: 키 바인딩 목록 확인하기
