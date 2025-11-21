---
layout: post
title: Linux - 셸 커스터마이징 & dotfiles
date: 2024-11-23 19:20:23 +0900
category: Linux
---
# 셸 커스터마이징 & dotfiles

## Dotfiles란? — 어디에 무엇을 넣는가

### 핵심 파일과 역할

| 파일 | 용도(권장 내용) |
|---|---|
| `~/.bashrc` | Bash **인터랙티브** 셸 설정(프롬프트, alias, 함수, completion) |
| `~/.bash_profile` | Bash **로그인** 셸(환경 변수, PATH, 런처) — 보통 끝에서 `source ~/.bashrc` |
| `~/.zshrc` | Zsh 인터랙티브 설정(프롬프트, 플러그인, zle, completion) |
| `~/.zprofile` | Zsh 로그인 셸 환경(환경 변수/경로) |
| `~/.profile` | POSIX 로그인 초기화(가능하면 최소화) |
| `~/.config/…` | XDG 기반 설정(권장). 예: `~/.config/starship.toml`, `~/.config/git/config` |

> 팁: **로그인 vs 인터랙티브**를 분리하면 속도/안정성/가독성이 좋아진다. `PATH`, `LANG`, 프록시 등은 로그인 파일, alias/프롬프트/완성은 인터랙티브 파일에 둔다.

현재 셸에 재적용:
```bash
source ~/.bashrc     # Bash
source ~/.zshrc      # Zsh
```

---

## alias — 타이핑 비용을 줄이는 최소 단위

### 기본 예시 (`~/.bashrc` 또는 `~/.zshrc`)

```bash
alias ll='ls -alF --group-directories-first --time-style=long-iso'
alias gs='git status -sb'
alias gc='git commit'
alias gp='git push'
alias py='python3'
alias v='nvim'
alias ports='ss -tulpen | less -S'
```

### 조건부 alias (환경에 따라 다르게)

```bash
if command -v eza >/dev/null 2>&1; then
  alias ls='eza --group-directories-first --icons=never'
fi
```

### 주의(실수 방지)

- **파괴적 명령**은 명시적으로: `rm`, `mv`, `cp`에 무조건 `-i`를 붙이는 습관은 스크립트 호환성이 떨어질 수 있다. 대신 **인터랙티브에서만** 옵션을 붙이거나 `trash-cli` 도입을 고려.
- alias 이름은 **짧되 의미 유지**(팀 공유 시 특히 중요).

---

## — 반복 작업 자동화

### 만능 압축 해제

```bash
extract() {
  local f="$1"
  [ -f "$f" ] || { echo "파일이 존재하지 않음: $f" >&2; return 1; }
  case "$f" in
    *.tar.bz2) tar xvjf "$f" ;;
    *.tar.gz)  tar xvzf "$f" ;;
    *.tar.xz)  tar xvJf "$f" ;;
    *.zip)     unzip "$f" ;;
    *.7z)      7z x "$f" ;;
    *)         echo "지원하지 않는 형식: $f" >&2; return 2 ;;
  esac
}
```

### 경로 북마크

```bash
mark() { mkdir -p "$HOME/.marks"; ln -snf "$(pwd)" "$HOME/.marks/$1"; }
jump() { cd -P "$HOME/.marks/$1" 2>/dev/null || echo "마크 없음: $1"; }
```

### 안전한 PATH 추가

```bash
path_prepend() { case ":$PATH:" in *":$1:"*) ;; *) PATH="$1:$PATH";; esac; }
path_append()  { case ":$PATH:" in *":$1:"*) ;; *) PATH="$PATH:$1";; esac; }

path_prepend "$HOME/.local/bin"
```

---

## — 보기 좋고, **깨지지 않는** 프롬프트

### Bash 기본

```bash
# 색상은 비출력 시퀀스 $$ $$ 로 감싸야 줄바꿈/편집이 깨지지 않는다.

PS1='$$\e[1;32m$$\u@\h $$\e[0;34m$$\w$$\e[0m$$ \$ '
```

| 이스케이프 | 의미 |
|---|---|
| `\u` | 사용자 |
| `\h` | 호스트 |
| `\w` | 현재 디렉터리 |
| `\t` | 시간 |
| `\$` | 일반 $, 루트 # |

### Git 브랜치 자동 표시 (`git-prompt`)

```bash
# ~/.bashrc

if [ -f /usr/share/git/completion/git-prompt.sh ]; then
  source /usr/share/git/completion/git-prompt.sh
  export GIT_PS1_SHOWDIRTYSTATE=1
  export GIT_PS1_SHOWUPSTREAM=auto
  PS1='$$\e[1;32m$$\u@\h $$\e[0;34m$$\w$$\e[33m$$$(__git_ps1 " (%s)")$$\e[0m$$ \$ '
fi
```

### 명령 시간/종료코드 표시 (Bash `PROMPT_COMMAND`)

```bash
timer_start() { export __TIMER=${EPOCHREALTIME:-$(date +%s.%N)}; }
timer_stop()  { awk "BEGIN{print $(date +%s.%N) - ${__TIMER:-0}}" 2>/dev/null; }

export PROMPT_COMMAND='EC=$?; D=$(timer_stop); timer_start; PS1="$$\e[90m$$[$EC|${D:-0.00}s]$$\e[0m$$ \u@\h \w \$ ";'
timer_start
```
- 종료코드가 0이 아니면 색을 바꿔 강조하는 식으로 응용 가능.

### 터미널 제목 갱신

```bash
case "$TERM" in xterm*|rxvt*)
  PROMPT_COMMAND='echo -ne "\033]0;${USER}@${HOSTNAME}: ${PWD}\007";'$PROMPT_COMMAND
esac
```

### Zsh 프롬프트(간단)

```zsh
autoload -Uz colors && colors
PROMPT='%{$fg_bold[green]%}%n@%m %{$fg_bold[blue]%}%~%{$reset_color%} %# '
```

---

## — “탭 두 번”이 주는 차이

### Bash

```bash
# 패키지: bash-completion (배포판별 이름 상이)

if [ -f /usr/share/bash-completion/bash_completion ]; then
  . /usr/share/bash-completion/bash_completion
fi
```
사용자 정의 완성:
```bash
# docker 서브커맨드 예시 (간단)

_docker_subs() { COMPREPLY=( $(compgen -W "ps images run exec logs rm rmi" -- "${COMP_WORDS[1]}") ); }
complete -F _docker_subs docker
```

### Zsh — 강력한 기본기

```zsh
autoload -Uz compinit && compinit
# 메뉴 완성, 색

zstyle ':completion:*' menu select
zstyle ':completion:*:descriptions' format '%F{yellow}-- %d --%f'
```

### fzf 연동(강력 추천)

```bash
# 설치 후
# 키바인딩/완성 스크립트

eval "$(fzf --bash)"
# 또는 zsh
# eval "$(fzf --zsh)"

# Ctrl-R: 히스토리 검색, Ctrl-T: 파일 선택, Alt-C: 디렉터리 탐색

```

---

## 히스토리 — “어제 그 명령”을 정확히

### Bash (`~/.bashrc`)

```bash
export HISTSIZE=50000
export HISTFILESIZE=100000
export HISTCONTROL=ignoredups:erasedups
export HISTTIMEFORMAT='%F %T '
shopt -s histappend   # 세션 종료 시 append
PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
# -a: 새 명령 추가, -c/-r: 동기화(멀티 탭 간 충돌 줄임)

```

### Zsh (`~/.zshrc`)

```zsh
HISTSIZE=50000
SAVEHIST=100000
HISTFILE=~/.zsh_history
setopt APPEND_HISTORY
setopt SHARE_HISTORY
setopt HIST_IGNORE_ALL_DUPS
setopt HIST_REDUCE_BLANKS
```

---

## 키 바인딩/편집 — Readline & ZLE

### Bash(Readline)

```bash
# ~/.inputrc

set editing-mode vi
# 또는 emacs 모드 유지 + 일부 단축키

"\e[5~": history-search-backward
"\e[6~": history-search-forward
```

### Zsh(ZLE)

```zsh
bindkey -v             # vi 모드
bindkey '^P' up-line-or-search
bindkey '^N' down-line-or-search
```

---

## 안전한 셸 옵션 — 스크립트에서 생존율 높이기

### Bash 강모드(스크립트 상단)

```bash
set -Eeuo pipefail
# -e: 실패 시 종료, -u: 미정의 변수 에러, -o pipefail: 파이프 실패 전파

trap 'echo "에러: ${BASH_SOURCE[0]}:$LINENO: $BASH_COMMAND" >&2' ERR
```

### Zsh 유사 설정

```zsh
set -euo pipefail
```

---

## 테마/프롬프트 프레임워크 — Starship & Powerlevel10k

### Starship(크로스 셸)

설치:
```bash
curl -sS https://starship.rs/install.sh | sh
echo 'eval "$(starship init bash)"' >> ~/.bashrc
# 또는

echo 'eval "$(starship init zsh)"'  >> ~/.zshrc
```

설정: `~/.config/starship.toml`
```toml
add_newline = true

[character]
success_symbol = "[➜](bold green)"
error_symbol = "[✗](bold red)"

[username]
style_user = "yellow bold"
show_always = true

[directory]
style = "cyan"
truncate_to_repo = true

[git_branch]
symbol = " "
truncation_length = 24

[time]
disabled = false
format = "[$time]($style) "
time_format = "%F %T"
style = "bright-black"
```

### Powerlevel10k(Zsh 전용)

```bash
sudo apt install zsh -y
chsh -s "$(command -v zsh)"
# oh-my-zsh

sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# p10k

git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k
# ~/.zshrc

ZSH_THEME="powerlevel10k/powerlevel10k"
```
- 첫 실행 시 구성 마법사. **Nerd Font** 기반 아이콘이 필요(터미널 폰트 설정 포함).

---

## 플러그인 관리 — 가볍고, 재현 가능한 방식

### Zsh 플러그인(oh-my-zsh 또는 경량 매니저)

- oh-my-zsh: 간편하지만 무겁다는 평.
- 대안: **antidote**, **zinit**, **zplug** 등(빠르고 선언적).

예) antidote
```zsh
# 설치

git clone --depth=1 https://github.com/mattmc3/antidote.git ~/.antidote
# ~/.zshrc

source ~/.antidote/antidote.zsh
antidote load <<'EOF'
zsh-users/zsh-syntax-highlighting
zsh-users/zsh-autosuggestions
zsh-users/zsh-completions
romkatv/powerlevel10k
EOF
```

---

## — 현대적 CLI

설치(배포판별 패키지명 상이):
```bash
# 예: Debian/Ubuntu

sudo apt install fzf fd-find ripgrep bat eza git-delta -y
# 일부는 이름이 fd-find/batcat로 깔림 → alias로 보정

alias fd='fdfind'
alias bat='batcat'
```

추천 alias:
```bash
alias cat='bat --style=plain --paging=never'
alias findf='fd --hidden --follow --exclude .git'
alias grep='rg --hidden --smart-case -n'
```

fzf 바인딩:
```bash
# 파일 선택 후 편집

ff() { local f; f="$(fzf)"; [ -n "$f" ] && ${EDITOR:-nvim} "$f"; }
# Git 브랜치 체크아웃

gcof() { git branch --all | sed 's/.* //; s#remotes/[^/]*/##' | sort -u | fzf | xargs git checkout; }
```

---

## 컬러/LS_COLORS/dircolors

```bash
# GNU dircolors

if command -v dircolors >/dev/null 2>&1; then
  [ -f ~/.dircolors ] && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
  alias ls='ls --color=auto'
fi
```

샘플 `~/.dircolors`는 Solarized/Dracula 등 테마를 가져다 쓰면 좋다.

---

## 환경/버전 관리자 — asdf, pyenv, nvm

### asdf(다언어)

```bash
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
# 플러그인

asdf plugin add nodejs https://github.com/asdf-vm/asdf-nodejs.git
asdf install nodejs 20.11.1
asdf global nodejs 20.11.1
```

---

## 디렉터리별 환경 — direnv

프로젝트마다 환경변수가 달라야 하면:
```bash
sudo apt install direnv
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
# 또는 zsh

echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
```
프로젝트 루트에 `.envrc` 작성:
```bash
layout python3
export DJANGO_DEBUG=True
```
최초 `direnv allow` 후 자동 로드.

---

## 성능/지연 분석 — 느린 셸을 빠르게

### Zsh 프로파일링

```zsh
zmodload zsh/zprof
# rc 마지막에

zprof
```

### Bash 진입 지연 확인

```bash
PS4='+ ${BASH_SOURCE}:${LINENO}:${FUNCNAME[0]}() '
bash -xlic 'exit' 2>bash_profile_trace.log
```
출력에서 느린 플러그인/네트워크 호출 제거/지연 로드.

---

## dotfiles 버전 관리 — 구조·부트스트랩

### 구조 예시

```text
~/dotfiles/
├── bash/
│   ├── .bashrc
│   └── .bash_profile
├── zsh/
│   ├── .zshrc
│   └── .zprofile
├── git/
│   ├── .gitconfig
│   └── .gitignore_global
├── config/
│   ├── starship.toml
│   ├── nvim/
│   └── tmux/
└── bootstrap.sh
```

### GNU Stow — 심볼릭 링크로 배포

```bash
# 설치

sudo apt install stow -y
# 루트에서

cd ~/dotfiles
stow -v bash
stow -v zsh
stow -v config
```
언스톨:
```bash
stow -v -D zsh
```

### chezmoi — 선언형/멀티머신

```bash
sh -c "$(curl -fsLS get.chezmoi.io)"
chezmoi init git@github.com:you/dotfiles.git
chezmoi apply
```
- 머신별 조건/템플릿(Go 템플릿)로 유연.

### yadm — Git만으로 간단히

```bash
yadm clone git@github.com:you/dotfiles.git
yadm status
```

### 부트스트랩 스크립트(샘플)

```bash
#!/usr/bin/env bash

set -euo pipefail
REPO="${HOME}/dotfiles"

if ! command -v stow >/dev/null; then
  sudo apt update && sudo apt install -y stow
fi

mkdir -p "${HOME}/.config"
cd "$REPO"
stow -v bash zsh config git
echo "Done. Open a new shell."
```

---

## 비밀/민감 정보 — 안전하게 분리

- **절대 커밋하지 말 것**: 토큰/SSH 개인키/API Key.
- `.gitignore_global`에 다음 포함:
```text
.env
.secrets
*.key
*.pem
*.age
*.gpg
```
- 필요시 **sops(age/gpg)**, **1Password CLI(op)**, **Pass**로 암호화 후 런타임 복호/주입.
- 머신별 값은 `~/.localrc` 등 **별도 파일**에 두고 rc에서 조건 로드:
```bash
[ -f ~/.localrc ] && source ~/.localrc
```

---

## 호스트/OS별 조건부 설정

```bash
case "$(uname -s)" in
  Linux)   export EDITOR=nvim ;;
  Darwin)  export EDITOR=vim ; alias ls='ls -G' ;;
esac

case "$HOSTNAME" in
  prod-*) export PS1="$$\e[41;97m$$ PROD $$\e[0m$$ $PS1" ;;
esac
```
- 프로덕션에서 프롬프트 배경을 빨강으로 — **실수 예방**.

---

## tmux/터미널과 연동

### tmux 기본

```bash
# ~/.tmux.conf

set -g mouse on
set -g history-limit 100000
setw -g mode-keys vi
bind r source-file ~/.tmux.conf \; display "reloaded"
```
rc에서 자동 진입(원할 때만):
```bash
# ~/.bash_profile

if [ -z "$TMUX" ] && [ -n "$PS1" ]; then
  tmux attach -t main || tmux new -s main
fi
```

---

## 실전 시나리오 모음

### 시나리오 1: “어제 그 긴 kubectl” — 히스토리+fzf

```bash
# Ctrl-R 로 fuzzy 검색 → Enter

```
추가: 동일 접두사 탐색
```bash
bind '"\e[A":history-search-backward'
bind '"\e[B":history-search-forward'
```

### 시나리오 2: “프로젝트마다 Python 버전/ENV 다름” — asdf+direnv

`.tool-versions`:
```
python 3.11.9
nodejs 20.11.1
```
`.envrc`:
```bash
layout python3
export DJANGO_SETTINGS_MODULE=conf.settings.local
```

### 시나리오 3: “팀에 새 노트북 지급 — dotfiles 5분 내 배포”

```bash
git clone git@github.com:org/dotfiles.git ~/dotfiles
~/dotfiles/bootstrap.sh
```

### 시나리오 4: “느려진 Zsh” — zprof로 범인 색출

```zsh
zmodload zsh/zprof
# rc 끝에서

zprof   # 실행 후 상단에 오래 걸린 함수/스크립트 확인
```

---

## 점검 체크리스트

- [ ] 로그인/인터랙티브 분리 (`.bash_profile` → `source ~/.bashrc`)
- [ ] 프롬프트 색상은 **비출력 시퀀스 감싸기**(Bash `$$ $$`)
- [ ] bash-completion/compinit 활성화
- [ ] 히스토리 공유/중복 제거/타임스탬프
- [ ] `set -Eeuo pipefail`로 스크립트 안전성 확보
- [ ] fzf/fd/rg/bat/eza/delta 등 도구 세트 배치
- [ ] dotfiles는 Stow/chezmoi/yadm로 **재현 가능 배포**
- [ ] 비밀/머신별 설정은 **분리/암호화**
- [ ] 성능 프로파일링(zprof/bash -x)으로 병목 제거
- [ ] 프로덕션 프롬프트 경고색 적용(사고 예방)

---

## Bash/Zsh 시작 파일 로딩 순서(요약)

### Bash

- 로그인 셸: `/etc/profile` → `~/.bash_profile`(또는 `~/.bash_login`/`~/.profile`) → 내부에서 `source ~/.bashrc` 권장
- 인터랙티브 비로그인: `~/.bashrc`
- 비인터랙티브: `BASH_ENV`가 지정되면 그 파일 로드

### Zsh

- 로그인: `/etc/zprofile` → `~/.zprofile`
- 인터랙티브: `/etc/zshrc` → `~/.zshrc`
- 로그아웃: `~/.zlogout`

---

## 안전한 Git 전역 설정 예시

```ini
# ~/.gitconfig

[user]
  name = Your Name
  email = you@example.com
[core]
  editor = nvim
  excludesfile = ~/.gitignore_global
[push]
  default = simple
[color]
  ui = auto
[init]
  defaultBranch = main
[diff]
  tool = delta
[pager]
  diff = delta
  log = delta
  reflog = delta
```

---

## 마무리

셸 커스터마이징은 미관의 문제가 아니라 **속도/정확도/안정성**을 좌우하는 엔지니어링 작업이다.
- **구조화(로그인 vs 인터랙티브)**, **안전 옵션**, **유지보수 쉬운 dotfiles 배포**, **현대적 CLI 도구**를 적용하라.
- 처음엔 가볍게 시작하되, 팀/머신/프로덕션까지 확장 가능한 **표준화된 설계**를 목표로 하라.
