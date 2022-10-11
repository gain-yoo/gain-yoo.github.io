---

published: true
layout: single
title: "[Kubernetes/Git] 쿠버네티스 공식문서 한글화 기여"
excerpt: "[ko]Translate untranslated sentence in ko/Deployments page #37120"
categories: Kubernetes
tag: [Kubernetes, DevOps, 쿠버네티스 공식문서 한글화 기여]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---

쿠버네티스 공식문서를 읽는 중에 한글 번역이 한 줄만 안되어 있는 걸 발견했다!  
마침 같은 팀원 중에 쿠버네티스 한글화팀으로 활동 중인 분이 있어서 이를 말씀드렸더니 직접 PR을 올려 보라고 권유해 주셨다.
![image](https://user-images.githubusercontent.com/100563973/195141377-06293caf-3123-43c2-a2fc-a2814ae1984d.png)

처음해 보는 Github에서 Pull Request 라니…..처음해 보는 쿠버네티스 한글화 기여라니…. 으악!  
사실 해보고 싶었다 ㅎㅎ (씨익)

처음해 보는 것 투성이니까! 블로그에 기록용으로 남겨 본다!  
작업하면서 [진이님 블로그 주소](https://jinnypark9393.github.io/posts/OSSCA)를 제일 많이 참고했다. (감사합니다 진이님 ㅎㅎ)

## 내가 헷갈렸던 용어

- `origin` : fork한 내 원격 저장소 *(= gain-yoo/website)*
- `upstream` : 원본 원격저장소 *(= kubernetes/website)*

## 내 로컬에서 사전 작업

먼저 [쿠버네티스 공식 웹사이트 repository](https://github.com/kubernetes/website)를 내 repository에 **fork**하여 아래 단계로 진행하였다.

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website
$ git clone https://github.com/gain-yoo/website.git
	Cloning into 'website'...
	remote: Enumerating objects: 304588, done.
	remote: Total 304588 (delta 0), reused 0 (delta 0), pack-reused 304588
	Receiving objects: 100% (304588/304588), 351.09 MiB | 20.62 MiB/s, done.
	Resolving deltas: 100% (217446/217446), done.
	Updating files: 100% (7526/7526), done.
```

내 local에 방금 **fork**한 repository를 **clone**했다.

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website
$ cd website/

rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (main)
$ git checkout dev-1.25-ko.1
	Switched to a new branch 'dev-1.25-ko.1'
	branch 'dev-1.25-ko.1' set up to track 'origin/dev-1.25-ko.1'.
```

website 디렉토리로 이동하고 현재 한글화팀에서 작업하고 있는 `dev-1.25-ko.1` 브랜치로 **checkout**해 준다. *(22년 10월 05일 기준)*

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (dev-1.25-ko.1)
$ git checkout -b gain-yoo/deployment/v0.1
	Switched to a new branch 'gain-yoo/deployment/v0.1'
```

한글화 작업을 하기 위한 `gain-yoo/deployment/v0.1` branch를 별도로 생성해 주었다.

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (gain-yoo/deployment/v0.1)
$ git remote add upstream https://github.com/kubernetes/website.git

rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (gain-yoo/deployment/v0.1)
$ git remote -v
	origin  https://github.com/gain-yoo/website.git (fetch)
	origin  https://github.com/gain-yoo/website.git (push)
	upstream        https://github.com/kubernetes/website.git (fetch)
	upstream        https://github.com/kubernetes/website.git (push)
```

**kubernetes/website** 프로젝트를 upstream에 추가한다.

그리고 **orgin** (내 로컬의 원격저장소)와 **upstream** (원본 원격저장소)를 확인한다.

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (gain-yoo/deployment/v0.1)
$ git fetch upstream
	remote: Enumerating objects: 53, done.
	remote: Counting objects: 100% (47/47), done.
	remote: Compressing objects: 100% (6/6), done.
	remote: Total 53 (delta 41), reused 41 (delta 41), pack-reused 6
	Unpacking objects: 100% (53/53), 33.77 KiB | 126.00 KiB/s, done.
	From https://github.com/kubernetes/website
	//...(중략)...
	 * [new branch]            dev-1.25             -> upstream/dev-1.25
	 * [new branch]            dev-1.25-ko.1        -> upstream/dev-1.25-ko.1
	 * [new branch]            main                 -> upstream/main
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (gain-yoo/deployment/v0.1)
$ git rebase upstream/dev-1.25-ko.1
	Successfully rebased and updated refs/heads/gain-yoo/deployment/v0.1.
```

**fetch**로 upstream의 `dev-1.25-ko.1` 브랜치를 최신화하고(커밋 내역을 가져 오고) 내 작업 브랜치를 upstream 브랜치 끝 위치로 **rebase**해 준다.

## PR(Pull Request)올리기

호다다닥 필요한 부분을 한글화하고 나서~

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (gain-yoo/deployment/v0.1)
$ git add .

rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (gain-yoo/deployment/v0.1)
$ git commit -m "Translate /ko/docs/concepts/workloads/controllers/deployment/#이전-수정-버전으로-롤백 into Korean"
	[gain-yoo/deployment/v0.1 0af0b86463] Translate /ko/docs/concepts/workloads/controllers/deployment/#이전-수정-버전으로-롤백 into Korean
	 2 files changed, 6 insertions(+), 1 deletion(-)
```

로컬저장소에 있는 파일을 내 원격 저장소(origin)에 업로드하기 위해 `git add .` 로 현재 폴더에 있는 파일을 Staged 상태로 만든다.

그러고 커밋메세지를 적어 주고 Staging Area에 있는 변경사항을 commit 한다.

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (gain-yoo/deployment/v0.1)
$ git push
	fatal: The current branch gain-yoo/deployment/v0.1 has no upstream branch.
	To push the current branch and set the remote as upstream, use
	
	    git push --set-upstream origin gain-yoo/deployment/v0.1
	

rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_website/website (gain-yoo/deployment/v0.1)
$ git push --set-upstream origin gain-yoo/deployment/v0.1

	Enumerating objects: 19, done.
	Counting objects: 100% (19/19), done.
	Delta compression using up to 8 threads
	Compressing objects: 100% (10/10), done.
	Writing objects: 100% (10/10), 1.02 KiB | 1.02 MiB/s, done.
	Total 10 (delta 8), reused 0 (delta 0), pack-reused 0
	remote: Resolving deltas: 100% (8/8), completed with 8 local objects.
	remote:
	remote: Create a pull request for 'gain-yoo/deployment/v0.1' on GitHub by visiting:
	remote:      https://github.com/gain-yoo/website/pull/new/gain-yoo/deployment/v0.1
	remote:
	To https://github.com/gain-yoo/website.git
	 * [new branch]            gain-yoo/deployment/v0.1 -> gain-yoo/deployment/v0.1
	branch 'gain-yoo/deployment/v0.1' set up to track 'origin/gain-yoo/deployment/v0.1'.
```

**origin** `gain-yoo/website`에 있는 `gain-yoo/deployment/v0.1` 작업 브랜치에 **push**한다.

이제 내 Github Repository에 들어가서 작업 브랜치가 잘 생성되었는지 확인하고 `Compare & Pull Request` 버튼을 누른다.

처음 PR은 직접 생성해 줘야 하지만 한번 연동해 두면 다음부터 진행하는 push는 Github에서 **자동으로 기존 생성한 PR에 변경사항을 업데이트** 해준다…!!!

여러 개의 commit을 한 상황이라면 반드시 거쳐줘야 할 과정이 있다.  
쿠버네티스 공식문서 한글화팀에서는 PR을 한글화팀 작업 브랜치에 merge할 때 1개의 PR당 1개의 commit만 남도록 정리하고 있으니 아래를 참조하자!

## PR 최종 반영 시 commit squashing하기 (amend 또는 rebase)

![Untitled](https://user-images.githubusercontent.com/100563973/195145050-61396745-6691-4ccf-91d4-d02b4edee558.png)

처음엔 `rebase`로 commit squashing을 해 보려다가 커밋이 꼬이는 바람에 좀 더 쉬운 `amend`를 사용해 보았다.

- `rebase`는 히스토리를 깔끔하게 유지하기 위해 사용하며, checkout한 브랜치의 커밋 내역을 지정하는 브랜치의 마지막 커밋 뒤에 재배치(rebase)하는 것을 말한다.

[참고 자료1](https://hajoung56.tistory.com/5)
[참고 자료2](https://git-scm.com/book/ko/v2/Git-%EB%B8%8C%EB%9E%9C%EC%B9%98-Rebase-%ED%95%98%EA%B8%B0)

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_docs_ko/website (gain-yoo/deployment/v0.2)
$ git remote -v
	origin  https://github.com/gain-yoo/website.git (fetch)
	origin  https://github.com/gain-yoo/website.git (push)
	upstream        https://github.com/kubernetes/website.git (fetch)
	upstream        https://github.com/kubernetes/website.git (push)
```

현재 orgin (내 로컬의 원격저장소)와 upstream (원본 원격저장소)를 확인한다.

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_docs_ko/website (gain-yoo/deployment/v0.2)
$ git fetch upstream
	remote: Enumerating objects: 625, done.
	remote: Counting objects: 100% (552/552), done.
	remote: Compressing objects: 100% (181/181), done.
	remote: Total 625 (delta 436), reused 446 (delta 369), pack-reused 73
	Receiving objects: 100% (625/625), 275.65 KiB | 11.03 MiB/s, done.
	Resolving deltas: 100% (443/443), completed with 153 local objects.
	From https://github.com/kubernetes/website
	   fb32f12411..0df28eecd5  dev-1.25-ko.1 -> upstream/dev-1.25-ko.1
	   edbfb712b9..9d625f4583  main          -> upstream/main
```

upstream의 브랜치를 최신화한다. (upstream의 커밋 내역들을 가져온다)


```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_docs_ko/website (gain-yoo/deployment/v0.2)
$ git add .

rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_docs_ko/website (gain-yoo/deployment/v0.2)
$ git commit --amend --no-edit
	[gain-yoo/deployment/v0.2 254dd472f9] Translate untranslated sentence in ko/Deployments page
	 Date: Sat Oct 1 11:47:48 2022 +0900
	 1 file changed, 1 insertion(+), 1 deletion(-)
```

처음 커밋할 때와 비슷해 보이지만 다른 부분은 커밋메세지를 또 주지 않는다는 점이다.

`--amend --no-edit` 옵션을 주게 되면 커밋 자체에는 **아무런 변경사항 없도록** 커밋 상태를 변경해 준다.

```java
rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_docs_ko/website (gain-yoo/deployment/v0.2)
$ git push
	To https://github.com/gain-yoo/website.git
	 ! [rejected]              gain-yoo/deployment/v0.2 -> gain-yoo/deployment/v0.2 (non-fast-forward)
	error: failed to push some refs to 'https://github.com/gain-yoo/website.git'
	hint: Updates were rejected because the tip of your current branch is behind
	hint: its remote counterpart. Integrate the remote changes (e.g.
	hint: 'git pull ...') before pushing again.
	hint: See the 'Note about fast-forwards' in 'git push --help' for details.

rkdls@DESKTOP-94ANG27 MINGW32 ~/OneDrive/바탕 화면/github/kubernetes_docs_ko/website (gain-yoo/deployment/v0.2)
$ git push -f origin gain-yoo/deployment/v0.2
	Enumerating objects: 17, done.
	Counting objects: 100% (17/17), done.
	Delta compression using up to 8 threads
	Compressing objects: 100% (9/9), done.
	Writing objects: 100% (9/9), 857 bytes | 857.00 KiB/s, done.
	Total 9 (delta 7), reused 0 (delta 0), pack-reused 0
	remote: Resolving deltas: 100% (7/7), completed with 7 local objects.
	To https://github.com/gain-yoo/website.git
	 + 3ad2059657...254dd472f9 gain-yoo/deployment/v0.2 -> gain-yoo/deployment/v0.2 (forced update)
```

origin `gain-yoo/website`에 `gain-yoo/deployment/v0.2` 작업 브랜치에 강제로 `-f` 옵션으로 push해 주면 깔끔하고 간-단하게 commit squashing 할 수 있게 된다!

## 내가 올린 PR

[https://github.com/kubernetes/website/pull/37120](https://github.com/kubernetes/website/pull/37120)

![Untitled 1](https://user-images.githubusercontent.com/100563973/195144985-a958cd14-205a-4a1f-93ec-5ad70c3434e9.png)
