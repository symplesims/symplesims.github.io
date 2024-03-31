---
layout: post
title: "Mac OS 개발자를 위한 로컬 개발 환경 구성"
date:  2021-12-2 17:00:00 +0900
categories: 
  - Development
  - Setup
  - MacOS
---

# Setup MacOs
새로운 Mac OS 를 가지게 되면 이것 저것 설치할 애플리케이션들이 많습니다.  
셋업에 하루가 꼬박 걸리기도 하는데 그때 마다 구글링 하며 이곳 저곳 찾아 다니면 낭비되는 시간이 아쉽습니다 ^^

## [homebrew 설치][#homebrew]

[Brew](https://brew.sh/index_ko) 는 MacOS 의 애플리케이션 설치 및 관리를 위한 필수 패키지 매니저 입니다. 
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

<br/>

## [oh-my-zsh 설치][#oh-my-zsh]

Zsh 터미널을 위한 필수 오픈 소스 입니다.

```
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

# 터미널 유틸리티 추가 
brew install zsh-autosuggestions
brew install zsh-syntax-highlighting
```

<br/>


## [개발자 Font][#dev-font]

글자도 이쁘고 자간도 일정한 폰트를 추천 합니다.

```
D2Coding
Droid Sans Mono Dotted for Powerline
```

<br/>

## 개발 관련 오픈 소스

### [git][#git]

```shell
brew install git
```


<br/>


### [ansible][#ansible]


AMI 빌드 및 리모트 OS 관리를 위한 오픈 소스 입니다.

```shell
brew install ansible
```

<br/>


### [tfswitch 테라폼 패키지 매니저][#tfswitch]

tfswitch 명령을 통해 terraform 의 다양한 버전을 관리 합니다.

```shell
brew install warrensbox/tap/tfswitch
tfswitch -l
terraform --version
ln -s /usr/local/bin/terraform /usr/local/bin/tf
```


<br/>

### [sdkman 패키지 매니저][#sdkman]

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


<br/>

### [Node 패키지 매니저][#node]

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
nvm ls-remote --lts 

# install node 
nvm install --lts
nvm install v18.18.2

# uninstall node
nvm uninstall v18.18.2

# select node version
nvm use v18.18.2
nvm alias default v18.18.2
nvm ls
node -v
npm -v

# yarn 플러그인 추가
npm install -global yarn
yarn -v
```

<br/>

### [python 패키지 매니저][#pyenv]

```shell
brew install pyenv
```

- [pyenv](https://www.daleseo.com/python-pyenv/) 을 통한 python 버전 관리 예시


```shell

# 3 으로 시작하는 버전 확인 
pyenv install -list | grep '^[ ]*3'

pyenv install 3.8.11
pyenv install 3.9.15

# 로컬에 설치된 python 버전 확인 
pyenv versions

# 글로벌 버전을 3.9.15 으로 설정 
pyenv global 3.9.15
python3 -version

# 참고로 python-2 버전은 2020 에 EOS 되었습니다.
pyenv install 2.7.18
```

<br/>

### [aws-vault][#pyenv]

```
brew install --cask aws-vault

# shimson 프로파일을 추가 합니다.
aws-vault add shimson

Enter Access Key Id: ABDCDEFDASDASF
Enter Secret Key: ************

# shimson 프로파일에 해당하는 AWS 관리 콘솔에 로그인 합니다. 
aws-vault login shimson

# shimson 프로파일을 대상은 aws cli 명령을 실행 합니다.
aws-vault exec shimson -- aws s3 ls

# aws-vault 에 등록된 프로파일 목록을 확인합니다. 
aws-vault list
```

<br>

### [go-lang][#go-lang]

```shell
brew install go
```

<br>

## [AWS CLI][#aws-cli]


```shell
# aws cli v2 설치
https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-cliv2.html

# aws-iam-authenticator for AWS EKS
brew install aws-iam-authenticator
```

<br>

## [Kubernetes 관련][#k8s]

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

<br/>


## [기타 툴][#etc-tools]


### MS Office
- App Store 를 통해 설치 

<br/>

### IntelliJ
```
https://www.jetbrains.com/idea/
```

<br/>

### Jetbrain Toolbox
```
https://www.jetbrains.com/ko-kr/toolbox-app/
```

<br/>

### IntelliJ Plug-In
```
# Jetbrain Toolbox 로도 편리하게 설치 가능
SonarLint
Terraform and HCL
Rainbow Brackets
Grep Console
```

<br/>


### Sublime Text
```
https://www.sublimetext.com/
```

<br/>


### [Authy OTP (Manually download)][#authy]

```
https://authy.com/download/
```


### [Postman][#postman]

```
https://www.postman.com/downloads/
```

<br/>