---
layout: post
title: Linux - 셸 커스터마이징 & dotfiles
date: 2024-11-23 19:20:23 +0900
category: Linux
---
# 셸 커스터마이징 & dotfiles

## Dotfiles 이해: 구성 요소와 역할

사용자별 셸 환경을 정의하는 숨김 설정 파일을 dotfiles라고 한다. 홈 디렉터리에 위치하며, 셸 동작, 별칭, 환경 변수, 프롬프트 등을 결정한다.

| 파일 | 주요 용도 |
|------|----------|
| `~/.bashrc` | Bash 대화형 셸 설정 (프롬프트, 별칭, 함수) |
| `~/.bash_profile` | Bash 로그인 셸 설정 (환경 변수, PATH) – 보통 마지막에 `source ~/.bashrc` 호출 |
| `~/.zshrc` | Zsh 대화형 셸 설정 |
| `~/.zprofile` | Zsh 로그인 셸 환경 설정 |
| `~/.profile` | POSIX 호환 로그인 초기화 파일 |
| `~/.config/…` | XDG 기반 설정 디렉터리 (현대적) |

**실용적 조언**: 로그인 셸과 대화형 셸 설정을 분리하면 속도와 가독성이 좋아진다. 환경 변수는 로그인 파일에, 별칭·프롬프트는 대화형 파일에 넣는다.

변경사항 적용:
```bash
source ~/.bashrc     # Bash
source ~/.zshrc      # Zsh
```

---

## 별칭(Alias): 타이핑 줄이기

자주 쓰는 긴 명령어를 짧게 대체한다.

```bash
alias ll='ls -alF --group-directories-first'
alias gs='git status -sb'
alias ports='ss -tulpen | less -S'
```

조건부 별칭 (명령어 존재 여부에 따라):
```bash
if command -v eza >/dev/null 2>&1; then
  alias ls='eza --group-directories-first'
fi
```

**주의**: `rm`에 무조건 `-i`를 붙이면 스크립트 호환성에 문제가 생길 수 있다. 대화형 세션에만 적용하거나 `trash-cli`를 고려하라.

---

## 셸 함수: 자동화

복잡한 작업을 함수로 묶는다.

```bash
# 압축 해제 함수
extract() {
  local f="$1"
  [ -f "$f" ] || { echo "파일 없음: $f" >&2; return 1; }
  case "$f" in
    *.tar.bz2) tar xvjf "$f" ;;
    *.tar.gz)  tar xvzf "$f" ;;
    *.zip)     unzip "$f" ;;
    *)         echo "지원 안 함: $f" >&2; return 2 ;;
  esac
}
```

```bash
# 디렉터리 북마크
mark() { mkdir -p "$HOME/.marks"; ln -snf "$(pwd)" "$HOME/.marks/$1"; }
jump() { cd -P "$HOME/.marks/$1" 2>/dev/null || echo "없음: $1"; }
```

---

## 프롬프트(Prompt) 커스터마이징

프롬프트는 정보를 제공하되 편집에 방해되지 않아야 한다.

### Bash 기본

```bash
PS1='\[\e[1;32m\]\u@\h \[\e[0;34m\]\w\[\e[0m\] \$ '
```
색상 코드는 `\[ \]`로 감싸야 줄바꿈 문제가 없다.

### Git 정보 표시 (bash)

```bash
if [ -f /usr/share/git/completion/git-prompt.sh ]; then
  source /usr/share/git/completion/git-prompt.sh
  export GIT_PS1_SHOWDIRTYSTATE=1
  PS1='\[\e[1;32m\]\u@\h \[\e[0;34m\]\w\[\e[33m\]$(__git_ps1 " (%s)")\[\e[0m\] \$ '
fi
```

### Zsh 기본

```zsh
autoload -Uz colors && colors
PROMPT='%{$fg_bold[green]%}%n@%m %{$fg_bold[blue]%}%~%{$reset_color%} %# '
```

---

## 자동완성(Completion)과 fzf

### Bash 자동완성

```bash
if [ -f /usr/share/bash-completion/bash_completion ]; then
  . /usr/share/bash-completion/bash_completion
fi
```

### Zsh 자동완성

```zsh
autoload -Uz compinit && compinit
zstyle ':completion:*' menu select
```

### fzf 통합

```bash
eval "$(fzf --bash)"   # Bash
eval "$(fzf --zsh)"    # Zsh
```

- `Ctrl+R`: 히스토리 검색
- `Ctrl+T`: 파일 선택
- `Alt+C`: 디렉터리 이동

---

## 명령어 히스토리 관리

### Bash

```bash
export HISTSIZE=50000
export HISTFILESIZE=100000
export HISTCONTROL=ignoredups:erasedups
export HISTTIMEFORMAT='%F %T '
shopt -s histappend
PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
```

### Zsh

```zsh
HISTSIZE=50000
SAVEHIST=100000
HISTFILE=~/.zsh_history
setopt APPEND_HISTORY SHARE_HISTORY HIST_IGNORE_ALL_DUPS
```

---

## 현대적 프롬프트 도구

### Starship (다중 셸)

```bash
curl -sS https://starship.rs/install.sh | sh
echo 'eval "$(starship init bash)"' >> ~/.bashrc
```

설정 파일 `~/.config/starship.toml`:

```toml
add_newline = true
[character]
success_symbol = "[➜](bold green)"
error_symbol = "[✗](bold red)"
[git_branch]
symbol = " "
```

### Powerlevel10k (Zsh 전용)

```zsh
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k
ZSH_THEME="powerlevel10k/powerlevel10k"
```
첫 실행 시 설정 마법사가 실행된다.

---

## 현대적 CLI 도구 통합

```bash
sudo apt install fzf fd-find ripgrep bat eza
alias cat='bat --style=plain --paging=never'
alias findf='fd --hidden --follow --exclude .git'
alias grep='rg --hidden --smart-case -n'
```

fzf 함수 예:
```bash
ff() { local f; f="$(fzf)"; [ -n "$f" ] && ${EDITOR:-nvim} "$f"; }
gcof() { git branch --all | sed 's/.* //' | sort -u | fzf | xargs git checkout; }
```

---

## dotfiles 버전 관리 및 배포

### 권장 구조

```
~/dotfiles/
├── bash/            # .bashrc, .bash_profile
├── zsh/             # .zshrc, .zprofile
├── git/             # .gitconfig
├── config/          # XDG 설정 (starship.toml 등)
└── bootstrap.sh     # 설치 스크립트
```

### GNU Stow로 배포

```bash
sudo apt install stow
cd ~/dotfiles
stow -v bash zsh config git
```

### 부트스트랩 스크립트 예시

```bash
#!/usr/bin/env bash
set -euo pipefail
REPO="${HOME}/dotfiles"
cd "$REPO"
stow -v bash zsh config git
echo "완료. 새 셸 세션을 시작하세요."
```

---

## 보안 및 머신별 설정

- 민감 정보(API 키, 비밀번호)는 **절대 커밋하지 않는다**.
- 머신별 설정은 `~/.localrc` 등으로 분리:

```bash
[ -f ~/.localrc ] && source ~/.localrc
```

- 환경별 조건:

```bash
case "$(uname -s)" in
  Linux)   export EDITOR=nvim ;;
  Darwin)  export EDITOR=vim ;;
esac
```

---

## 결론

셸 커스터마이징은 생산성과 작업 효율을 높이는 핵심 작업이다.  
- **구조화**: 로그인 셸과 대화형 셸을 분리하고, 별칭·함수·환경 변수를 적절히 배치한다.  
- **안전성**: `set -Eeuo pipefail` 같은 옵션으로 스크립트 견고성을 높인다.  
- **현대적 도구**: fzf, ripgrep, bat, Starship 등을 활용해 기존 경험을 개선한다.  
- **재현성**: Stow, chezmoi 등으로 dotfiles를 버전 관리하고 여러 머신에 일관되게 배포한다.  
- **보안**: 민감 정보는 절대 저장소에 넣지 않고, 환경 변수나 암호화된 도구로 관리한다.

처음에는 기본 설정부터 시작해 점진적으로 고급 기능을 추가하면, 안정적이면서도 강력한 셸 환경을 구축할 수 있다.