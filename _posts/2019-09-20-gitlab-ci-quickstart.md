---
layout: post
title: 'Gitlab CI Quickstart'
categories:
- Quickstart
tags:
- Infra
- Gitlab
- CI/CD
---

## Gitlab CI

Gitlab CI는 Gitlab에서 제공하는 CI/CD 기능이다.
Jenkins, Circle CI와 같이 빌드, 테스트, 배포 과정을 코드로 관리하고 자동화할 수 있다.

## Gitlab Runner

Gitlab CI는 작업중 상당한 메모리를 사용하게 되며 하나의 서버에 여러 서비스를 운영하는 것은 보안상으로도 좋지 않기 때문에
Gitlab과 다른 서버에서 실행하는 것이 좋다.

이를 위해 Gitlab Runner라는 에이전트를 사용한다.
Gitlab Runner를 빌드 작업을 수행할 인스턴스에 설치한 뒤 Gitlab과 연결한다.
Gitlab이 설치된 서버는 저장소 역할만 수행하고 Gitlab Runner가 설치된 인스턴스가 CI/CD 작업을 대신 수행한다.
굳이 비교하자면 Jenkins Slave와 비슷하다고 볼 수 있다.

Gitlab Runner는 Windows, macOS, Linux, Docker, Kubernetes 등 여러 플랫폼에 설치될 수 있다.

실제로 빌드 과정은 Executor가 수행하게 되는데 Gitlab Runner는 다양한 Executor를 구현하고 있다.
간단하게 Shell, SSH 등의 Executor가 있으며 Docker, Kubernetes 등의 Executor도 존재한다.
Executor마다 할 수 있는 것들도 약간씩 다르기 때문에 [문서](https://docs.gitlab.com/runner/executors/README.html#compatibility-chart)를
확인하고 현재 환경이나 빌드/배포 방법에 따라 원하는 Executor를 선택하면 된다.

## Hands-on

### Environment

Gitlab Instance: CentOS 7.6
Gitlab(self-hosted): v11.9.1

Gitlab Runner Instance: Fedora 30
Gitlab Runner: v12.2.0

### Install
Gitlab은 이미 self-hosted로 설치되어있고 Gitlab 어드민 권한을 가지고 있다.

Gitlab Runner는 페도라 데스크탑에 설치한다.
주요 리눅스 배포판들은 패키지를 제공하고 있다.
macOS나 Windows도 쉽게 설치할 수 있으니 [매뉴얼](https://docs.gitlab.com/runner/install/)을 보고 따라하면 된다.


[작성 시점에 Gitlab Runner 패키지는 Fedora 30을 지원하지 않기 때문에](https://gitlab.com/gitlab-org/gitlab-runner/issues/4401)
수동으로 Gitlab Runner를 설치한다.

```shell
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
sudo chmod +x /usr/local/bin/gitlab-runner
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

`sudo gitlab-runner status` 커맨드를 통해 Gitlab Runner가 정상적으로 동작중인지 확인 가능하다.

```console
[user@runner-instance ~]$ sudo gitlab-runner status
Runtime platform arch=amd64 os=linux pid=28859 revision=a987417a version=12.2.0
gitlab-runner: Service is running!
```

### Register

Gitlab Runner가 정상적으로 설치되었다면 Gitlab에 등록해야한다.
`sudo gitlab-runner register` 커맨드를 입력하면 등록 과정이 진행된다.
해당 과정은 대화형으로 진행된다.

```console
[user@runner-instance ~]$ sudo gitlab-runner register

Runtime platform arch=amd64 os=linux pid=28982 revision=a987417a version=12.2.0
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://gitlab.exampledomain.com/
Please enter the gitlab-ci token for this runner:
8sbd7jExNBEx36kOcdNX
Please enter the gitlab-ci description for this runner:
[runner-instance]:
Please enter the gitlab-ci tags for this runner (comma separated):
test,generic
Registering runner... succeeded runner=v3kDv9L2
Please enter the executor: custom, docker, parallels, virtualbox, docker-ssh+machine, docker-ssh, shell, ssh, docker+machine, kubernetes:
docker
Please enter the default Docker image (e.g. ruby:2.6):
python:3.7.4-slim-stretch
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

Gitlab Admin area > Overview > Runners 페이지에서 Gitlab-CI Coordinator URL과 Gitlab-CI Token은 그리고 현재 등록된 Gitlab Runner 목록을 확인할 수 있다.

### Pipeline

이제 실제로 CI/CD Pipeline 을 작성하고 실행해본다.

새 프로젝트를 생성한다.
좀 더 편하게 진행하기 위해 프로젝트 생성 페이지에서 `Create from template` 탭을 선택한 뒤 적당한 프로젝트를 생성한다.
`Pages/Jekyll` 템플릿을 사용하면 자동으로 파이프라인 파일까지 생성해준다.

###### .gitlab-ci.yml
```yaml
image: ruby:2.3

variables:
  JEKYLL_ENV: production
  LC_ALL: C.UTF-8

before_script:
  - bundle install

test:
  stage: test
  script:
  - bundle exec jekyll build -d test
  artifacts:
    paths:
    - test
  except:
  - master

pages:
  stage: deploy
  script:
  - bundle exec jekyll build -d public
  artifacts:
    paths:
    - public
  only:
  - master
```

Project > CI/CD > Pipelines 페이지에서 파이프라인 상태를 볼 수 있다.
파이프라인 파일은 샘플로 생성되어있으니 바로 `Run Pipeline` 버튼으로 파이프라인을 실행한다.

Docker를 사용하는 Pipeline/Executor 이므로 Gitlab Runner 인스턴스에 Dockerd가 실행중이어야한다.

***

###### Problem. 1

파이프라인을 수동으로 실행하였으나 job이 진행되지 않는다.

> This job is stuck because the project doesn't have any runners online assigned to it. Go to Runners page

파이프라인을 실행할 온라인 상태의 Gitlab Runner가 존재하지 않는다.
Gitlab Runner 등록시 태그를 지정하였으나 프로젝트에는 태그가 등록되지 않아서 발생한 문제이다.
Gitlab Runner 설정에서 `Run untagged jobs` 옵션을 활성화하거나 태그를 지정해주면 된다.

***

###### Problem. 2

처음 Gitlab Runner 를 등록했을 때에는 shared 상태였고 이후 특정 프로젝트에서만 사용할 수 있게 제한을 건 뒤로 specific
상태가 되었는데 이후 해당 Gitlab Runner 를 다시 shared 상태로 전환하려했으나 전환되지 않는다.

> You can't make this a shared Runner.

> If a runner was once used as a specified for some project, then it can't be moved back to a shared state.

[보안을 위해 의도된 동작인 모양이다.](https://gitlab.com/gitlab-org/gitlab-foss/issues/34827)
이 경우 기존 Gitlab Runner 을 제거하고 다시 등록해야한다.

***

파이프라인이 정상적으로 동작한다면 실시간으로 로그메세지를 볼 수 있다.
실행이 완료되면 해당 job은 passed 상태가 된다.

파이프라인은 수동으로 실행할수도 있지만 변경이 발생할때 자동으로 파이프라인을 실행한다.
단순히 마스터 브랜치를 바로 변경해도 되겠지만 Issue, Merge Request 생성하여 진행한다.

이슈 생성 뒤 해당 이슈에서 머지 리퀘스트를 생성한다.
브랜치가 추가되는데 실제로 무언가 수정을 하려는 것은 아니므로 Web IDE를 사용해 README.md 파일을 수정한다.

수정 사항을 커밋하고 파이프라인을 확인해보면 2개의 job이 생성된 것을 볼 수 있다.
하나는 브랜치 생성 시에 실행되었고, 하나는 README.md 파일이 수정되었을 때 실행되었다.

이전에 마스터 브랜치에서 수동으로 파이프라인을 실행했을 때와 이슈 브랜치에서 파이프라인이 실행되었을 때를 비교해보면
서로 다른 작업을 하는 것을 알 수 있다.

이슈 브랜치는 test 스테이지를 진행하고 마스터 브랜치는 deploy 브랜치를 실행했다.
.gitlab-ci.yml 를 보면 브랜치에 따라 서로 다른 작업을 하도록 설정되어있기 때문이다.

## Note

> You have multiple options: rsync, scp, sftp and so on. For now, we will use scp.

[Gitlab Doc에 나온 예시를 보면](https://docs.gitlab.com/ee/ci/examples/deployment/composer-npm-deploy.html)
실서버에 배포할때는 scp, rsync 등의 일반적인 파일 전송 도구를 쓰는 것으로 보이며
보안을 위해 별도의 계정을 생성하여 해당 계정을 통해 전송하는 것이 좋다.
