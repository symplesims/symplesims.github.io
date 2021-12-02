---
layout: post
title: "Mac OS 개발자를 위한 로컬 개발 환경 구성"
date:  2021-12-2 17:00:00 +0900
categories: 
  - Development
  - Setup
  - MacOS
---

## Setup MacOs
새로운 Mac OS 장비를 갖게 되면 이것 저것 설치할 앱들이 너무 많습니다.  
셋업에 하루가 꼬박 걸리기도 하는데 그때 마다 구글링 하며 이곳 저곳 찾아 다니면 아쉬게 생각이 드네요 ^^

### homebrew 설치
MacOs 의 애플리케이션 설치 및 관리를 위한 대표적인 패키지 매니저 입니다. 
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### oh-my-zsh 설치
Zsh 터미널을 위한 필수 오픈 소스 입니다.
```
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

# 터미널 유틸리티 추가 
brew install zsh-autosuggestions
brew install zsh-syntax-highlighting
```

### 개발자 Font
글자도 이쁘고 자간도 일정한 폰트를 추천 합니다.
```
D2Coding
Droid Sans Mono Dotted for Powerline
```

<br/>

### 개발 관련 오픈 소스

#### git 
```shell
brew install git
```

#### ansible
AMI 빌드 및 리모트 OS 관리를 위한 오픈 소스 입니다.
```shell
brew install ansible
```

#### tfswitch 패키지 매니저
tfswitch 명령을 통해 terraform 의 다양한 버전을 관리 합니다.

```shell
brew install warrensbox/tap/tfswitch
tfswitch -l
terraform --version
ln -s /usr/local/bin/terraform /usr/local/bin/tf
```

#### sdkman 패키지 매니저
Java 및 관련 오픈소스 버전들을 관리 합니다.
```shell
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
```

- java / maven / gradle 등 여러 버전 관리 예시
```shell
sdk list java 
sdk install java <Version_Identifier>
sdk install maven
sdk install gradle
```

#### Node 개발 환경을 위한 nvm 패키지 매니저 
```shell
brew install nvm

# [vi ~/.zshrc]
---
export NVM_DIR="$HOME/.nvm"
. "$(brew --prefix nvm)/nvm.sh"
---

nvm -v
```

- Node 버전 관리 예시

```shell
# node & npm 설치 
nvm install node
nvm install --lts
nvm ls
node -v
npm -v

# yarn 플러그인 추가
npm install -global yarn
yarn -v
```

#### python 개발 환경을 위한 pyenv 패키지 매니저

```shell
brew install pyenv
```

- [pyenv](https://www.daleseo.com/python-pyenv/) 을 통한 python 버전 관리 예시
```shell
pyenv install 3.10.0
pyenv install 3.6.9
python3 -version
pyenv versions
pyenv global 3.10.0
python3 -version
# 참고로 python-2 버전은 2020 에 EOS 되었습니다.
pyenv install 2.7.18
```

#### go-lang
```shell
brew install go
```

<br>

### AWS 관련 툴

```shell
# aws cli v2 설치
https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-cliv2.html

# aws-iam-authenticator for AWS EKS
brew install aws-iam-authenticator
```

<br>

### Kubernetes 관련 툴

```shell
# docker
brew install docker

# kubectl
https://kubernetes.io/ko/docs/tasks/tools/install-kubectl-macos/

# istoctl (istio)
https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/

# helm
brew install helm

# Kubernetes 로컬 테스트를 하고자 하는 경우에만 
brew install minikube
```

### 편집 툴

#### MS Office
- App Store 를 통해 설치 

#### IntelliJ
```
https://www.jetbrains.com/idea/
```

#### Jetbrain Toolbox
```
https://www.jetbrains.com/ko-kr/toolbox-app/
```

#### IntelliJ Plug-In
```
# Jetbrain Toolbox 로도 편리하게 설치 가능
SonarLint
Terraform and HCL
Rainbow Brackets
Grep Console
```

#### Postman
```
https://www.postman.com/downloads/
```

#### Sublime Text
```
https://www.sublimetext.com/
```

#### Authy OTP 프로그램 설치 (직접 다운로드)
```
https://authy.com/download/
```
