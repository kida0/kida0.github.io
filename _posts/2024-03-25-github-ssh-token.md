---
created: 2024-03-25
title: Github SSH Key 등록하기
layout: post
tags: [git]
category: 프로젝트

---




# Github SSH Key 등록하기

하나의 장비에서 여러 깃계정을 사용하다보니 충돌이 발생했다. Codecommit credential을 잘못 설정하면서 발생한 문제인 것 같은데, 끝내 에러의 원인은 찾지 못하고 SSH를 이용한 방식으로 문제를 해결했다. Git SSH 설정 및 사용 방법을 공유한다.



#### 1. SSH 키 생성하기

```bash
# ssh 키 생성
cd ~/.ssh
ssh-keygen -t rsa -C "sample1@email.com" -f "key-name1"
ssh-keygen -t rsa -C "sample2@email.com" -f "key-name2"
```

* 위의 이메일과 키 이름을 변경 후 SSH 키를 생성한다
  * `-t`: 생성할 키 유형
  * `-C`: 키 식별을 위한 이메일
  * `-f`: 키 이름



#### 2. ssh-agent에 새로 생성한 SSH Key 추가하기

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/key-name1
ssh-add ~/.ssh/key-name2

# 기존 키를 제거하고 싶은 경우
ssh-add -D

# 키가 정상적으로 추가되었는지 확인
ssh-add -l
```

* 키를 추가하기 위해 `ssh-add` 뒤에 위에서 생성한 키를 넣어준다.



#### 3. Github에 SSH Key 추가하기

```bash 
cat key-name1.pub
cat key-name2.pub
```

* 출력된 키 값을 전체 복사하여 다음 경로로 이동 후 붙어넣고, SSH Key를 생성한다
  * `github setting` >` SSH and GPG keys` >` New SSH Key` > `Add SSH key`



#### 4. config 파일 생성

```bash

# vim ~/.ssh/config
##################################################
Host github.com-key1
    HostName github.com
    User github-username1
    IdentityFile ~/.ssh/key-name1

Host github.com-key2
    HostName github.com
    User github-username2
    IdentityFile ~/.ssh/key-name2    
##################################################
```

* `Host`: 앞으로 사용할 호스트 이름
  * 앞으로 사용해야 하기 때문에 간단하고 식별 가능한 형태로 만드는 것이 좋다.

* `User`: 깃허브 유저 이름
* `IdentityFile`: 생성한 SSH Key 위치



#### 5. 작동 테스트

```bash
ssh -T git@github.com-key1
ssh -T git@github.com-key2
```



#### 6. 사용 방법

![image-20240325185803864](/Users/kida/personal/kida0.github.io/_posts/images/image-20240325185803864.png)

```bash
git remote add origin git@github.com-key1:kida0/test-repo.git
```

* 원격 저장소를 사용할 때 `git@github.com-key1:kida0/test-repo.git`과 같이 중간에 호스트 이름을 넣어주면 된다.


