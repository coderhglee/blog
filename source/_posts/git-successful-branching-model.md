---
title: 성공적인 Git 브랜치 모델
date: 2019-11-18 16:59:22
description : 성공적인 Git History를 만들기 위한 브랜치 모델
category: [git]
tags: [git]
---

## 성공적인 브랜치 모델

<https://nvie.com/posts/a-successful-git-branching-model/>
이 글은 다른 외국 블로거가 1년의 프로젝트 동안 경험하고 적용한 브랜치 전략에 대해서 정리한 글이다. 우리 팀에서도 이러한 브랜치 모델을 지향하고 있기때문에 포스팅을 통해 개념 및 사용법을 정리하고자 한다.

![image](https://user-images.githubusercontent.com/24283191/69035324-be8a6f80-0a26-11ea-941a-2ebc378de241.png)

### 왜 Git을 사용하는가

이전 회사에서는 svn을 통해 별도의 브랜치를 나누어 개발하지않고 master 브랜치를 통해 `commit` `update` 이 명령어 외 사용을 해본 기억이 많이 없다. 그러다 보니 필요성을 많이 느끼지 못하였고 왜 많은 회사에서 git을 사용하는지 공감하지못했다. 하지만 이직을 하고 사용해보니 이 좋은것을 왜 그동안 안썼는지 의문이었다.

Git은 개발자들의 merging and branching에 대한 인식을 바꾸어놓았다. 기존 CVS/SVN은 merging and branching은 두려웠으며 가끔씩만 하는 행동들이었다.

하지만 git은 이 행등돌을 매우 간단하고 쉽게, 그리고 workflow의 핵심부분중 하나로 생각했다.

merging and branching은 더이상 두려움의 대상이 아니다.

이글에서 제시하는 모델은 모든 팀 구성원이 따라야하는 개발 절차이다.

#### Decentralized but centralized

#### main branches

하나의 repository에는 2개의 `main branches` 존재한다.

* master
* develop
