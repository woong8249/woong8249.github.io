---
title: SSH Agent automation in Github
categories: [Tools,Github]
tags: [github,ssh,automation]
toc: true
image: /ssh.webp
---


<blockquote class="prompt-tip">
SSH의 개요 또는 SSH를 이용한 github 인증 방식에 대한 포스트는 넘치고 공식문서에도 너무 자세히 설명하니 이에대해서는 생략하겠다.
<b>SSH agent와 shell script를 이용한 passphrae를 반자동화하는 과정도 담아보려한다.</b>
</blockquote>

## SSH Agent

---

SSH Agent는 SSH 키를 관리하는 프로그램으로, 사용자가 매번 비밀번호를 입력하지 않고도 SSH 키를 사용할 수 있게 도와줍니다.

```bash
eval "$(ssh-agent -s)" #ssh-agent 실행
echo $SSH_AGENT_PID  # ssh-agent의 PID 출력
kill $SSH_AGENT_PID  # ssh-agent 프로세스 종료
pgrep ssh-agent # 실행중인 ssh-agent PID 반환
ssh-add -k ~/.ssh/woong8249 #agent에 ssh key 등록
ssh-add -l # # agnet에 등록된 key목록 확인
```

- `eval`을 여러번 실행하면 `singleton`으로 실행되지 않고 프로세스가 여러개 켜진다… 정확한 확인시에는 `echo $SSH_AGENT_PID` 보다는 `pgrep ssh-agent`를**권장한다**
- 터미널의 세션이 끝나면 `ssh-agent`는 종료된다.. 때문에 이 과정을 자동화하기 위해 셸 초기화 스크립트(**`.bashrc`**, **`.zshrc`**)에 **`ssh-agent`**와 **`ssh-add`** 명령어를 추가해 자동화 해보자
- 아래 스크립트는 하나의 머신에서 여러깃허브 계정을 사용하는경우는 약간 손볼 필요가 있을 수 있다.또 스크립트내에 비밀번호는 담지 않는게 낫다고 판단했다.

```bash
SSH_AGENT_PID=$(pgrep ssh-agent)

# ssh-agent가 실행 중이지 않으면, 새로 시작
if [ -z "$SSH_AGENT_PID" ]; then
    eval $(ssh-agent -s)
fi

# 특정 SSH 키를 ssh-agent에 추가
SSH_KEY_PATH="$HOME/.ssh/woong8249"
if [ -f "$SSH_KEY_PATH" ]; then
    ssh-add "$SSH_KEY_PATH"
else
    echo "SSH 키 파일이 존재하지 않습니다: $SSH_KEY_PATH"
fi
```

## Reference

---

- [SSH 정보 - GitHub Docs](https://docs.github.com/ko/authentication/connecting-to-github-with-ssh/about-ssh)

- [새 SSH 키 생성 및 ssh-agent에 추가 - GitHub Docs](https://docs.github.com/ko/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
