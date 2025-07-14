---
layout: post
title: Linux - 셸 커스터마이징 & dotfiles
date: 2024-11-21 19:20:23 +0900
category: Linux
---
# 리눅스 22편: 🐚 셸 커스터마이징 & dotfiles

## 🧾 Dotfiles란?

리눅스/유닉스 시스템에서 사용자의 셸 환경을 정의하는 **숨김 설정 파일들(dotfiles)**은 커스터마이징의 핵심입니다.

| 파일명          | 용도 |
|------------------|------|
| `~/.bashrc`       | Bash 인터랙티브 셸 설정 |
| `~/.bash_profile` | 로그인 셸 설정 (bash + 로그인) |
| `~/.zshrc`        | Zsh 셸 설정 |
| `~/.profile`      | POSIX 호환 로그인 설정 |

> `ls -a`로 홈 디렉토리 내 숨김 파일(dotfiles)을 확인할 수 있습니다.

---

## 🧷 alias: 명령어 단축

반복되는 명령을 간단히 줄일 수 있습니다.

### ✅ `.bashrc` / `.zshrc` 예시
```bash
alias ll='ls -alF'
alias gs='git status'
alias py='python3'
```

변경 후에는 셸에 적용해야 합니다:
```bash
source ~/.bashrc     # 또는 ~/.zshrc
```

---

## 🛠️ 함수(Function)로 반복 작업 자동화

```bash
extract() {
  if [ -f "$1" ]; then
    case "$1" in
      *.tar.bz2) tar xvjf "$1" ;;
      *.tar.gz)  tar xvzf "$1" ;;
      *.zip)     unzip "$1" ;;
      *)         echo "압축 형식 알 수 없음: $1" ;;
    esac
  else
    echo "파일이 존재하지 않음"
  fi
}
```

`.bashrc` 또는 `.zshrc`에 추가하여 사용합니다.

---

## 🎨 PS1: 프롬프트 커스터마이징

**PS1**은 프롬프트의 기본 문자열을 설정하는 변수입니다.

### ✅ 기본 예시
```bash
PS1='\u@\h:\w\$ '
```

| 기호 | 의미 |
|------|------|
| `\u` | 사용자 이름 |
| `\h` | 호스트 이름 |
| `\w` | 현재 작업 디렉토리 |
| `\$` | 일반 사용자($), 루트(#) 구분 |

### ✅ 컬러 프롬프트 예시
```bash
PS1='\[\e[1;32m\]\u@\h \[\e[0;34m\]\w \$\[\e[0m\] '
```

> 위 설정은 사용자 이름은 초록색, 디렉토리는 파란색으로 표시됩니다.

---

## 🚀 셸 테마 프레임워크

### 1. **Starship** - 빠르고 크로스쉘 지원

```bash
# 설치
curl -sS https://starship.rs/install.sh | sh

# 설정
echo 'eval "$(starship init bash)"' >> ~/.bashrc
# 또는 zsh: echo 'eval "$(starship init zsh)"' >> ~/.zshrc
```

**구성 파일**: `~/.config/starship.toml`  
테마, 색상, 아이콘, Git 상태 등 상세 조절 가능

```toml
# starship.toml 예시
add_newline = true

[username]
style_user = "yellow bold"
show_always = true

[directory]
style = "cyan"
```

---

### 2. **Powerlevel10k** - Zsh 전용 고급 프롬프트

```bash
# zsh 설치 필요
sudo apt install zsh
chsh -s $(which zsh)     # 기본 셸을 zsh로 변경

# oh-my-zsh 설치
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Powerlevel10k 설치
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k

# .zshrc 설정
ZSH_THEME="powerlevel10k/powerlevel10k"
```

**처음 실행 시** 자동으로 GUI 기반 설정 마법사가 뜨며 커스터마이징 가능 (색상, 아이콘, Git 표시 등)

---

## 🧳 dotfiles 백업 및 공유

`dotfiles`는 깃허브 등에 버전 관리하며 팀 공유나 개인 재설정 시 유용합니다.

### ✅ 예시 디렉토리 구성
```text
~/dotfiles/
├── .bashrc
├── .zshrc
├── .vimrc
├── starship.toml
```

### ✅ 심볼릭 링크로 적용
```bash
ln -s ~/dotfiles/.bashrc ~/.bashrc
ln -s ~/dotfiles/starship.toml ~/.config/starship.toml
```

---

## 🧪 실전 예시

### 예시 1: Git 브랜치까지 표시되는 프롬프트
```bash
PS1='\u@\h:\w$(__git_ps1 " (%s)")\$ '
```

### 예시 2: `update` 명령 하나로 시스템 업데이트
```bash
alias update='sudo apt update && sudo apt upgrade -y'
```

### 예시 3: 배경 색 포함 프롬프트
```bash
PS1='\[\e[47;30m\] \u@\h \w \$ \[\e[0m\]'
```

---

## ✨ 마무리

셸 커스터마이징은 단순한 편의성 그 이상입니다.  
효율적인 작업 흐름, 실수 방지, 시각적 만족까지 제공하므로  
**나만의 dotfiles 관리와 프롬프트 구성은 꼭 익혀야 할 실전 기술**입니다.