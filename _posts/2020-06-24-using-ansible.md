---
layout: post
title: 'Ansible을 사용해보았다'
summary: 처절한 실패 속에 해답이 있다
categories:
- Note
tags:
- Infra
- Ansible
- CI/CD
- Automation
---

현 직장에 Ansible을 도입한지 1년이 넘었다.
처음엔 거의 억지로 사용하던 Ansible도 이제 인프라팀 내에서는 어느정도 자리를 잡게되었다.

데브옵스 엔지니어로서 커리어를 시작했을 때부터 Ansible을 사용했으니 Ansible에 대한 사용이력과 실패의 역사는 곧 내 커리어 전반의 이력과 실패의 역사이다.
최근 Ansible 프로젝트를 정리하고 다시 적극적으로 활용하기 시작하면서 한번쯤 이 역사를 기록할 필요성을 느꼈다.
그동안 사내에서 Ansible을 사용하면서 했던 실패와 결정들을 이야기해보려 한다.

## 첫 도입

내가 개발자에서 데브옵스 엔지니어로 인프라팀에 들어갔을 때 인프라팀은 개발자들이 가지고 있던 운영 권한을 최대한 인프라팀으로 이전하고자 했다.
이를 위해 자동화는 필수불가결했으며 우리는 Ansible을 포함해 여러가지 도구들을 고려했다.

레드햇에 대한 이유를 알 수 없는 신뢰와 최신 트랜드 보고서는 내가 Ansible에 큰 관심을 가지게 만들었다.

나는 주도적으로 Ansible을 회사에 도입하고 Ansible의 필요성을 역설했다.
기존의 서버 구성과 운영 문서들을 Ansible 플레이북으로 옮겨적었고 AWX를 설치하여 Ansible 프로젝트를 시작했다.

## 구성과 운영은 다를까

Ansible은 멱등성을 보장한다.
만약 내가 플레이북을 제대로 짰다면 말이다.

나중을 고려하지 않은 플레이북은 기존 서버 설정을 박살내거나 설정파일을 나도 모르는 사이에 롤백시켜버린다.

처음 서버 초기 구성 플레이북을 작성했을 때 나는 한 번 플레이북이 실행된 타겟서버에 다시 해당 플레이북을 실행할 상황을 고려하지 않았다.
좀 더 정확히 설명하자면 고려는 했으나 정말 무슨 상황이 발생할 수 있을지에 대해서는 고민하지 않았다.

서버 초기 구성 플레이북은 일부 설정파일을 온전히 가지고 있다가 서버에 냅다 복사해버리는 방식을 사용했다.
이후 운영 과정에서 해당 설정파일에 몇가지 다른 설정이 들어가고 다시 해당 플레이북이 실행되면 설정파일은 다시 초기 상태로 돌아가린다.
다행히 그런 상황이 벌어지지는 않았지만 불행히도 나는 그런 상황이 발생할 수 있다는 것을 뒤늦게야 깨달았다.
결국 해당 플레이북은 한 번 실행된 이후에는 동일한 타겟서버를 대상으로 실행될 수 없도록 '인간적으로' 강제했다.

이러한 상황에서 플레이북의 일부 태스크는 운영중에도 몇번씩 다시 실행될 필요가 있었다.
나는 플레이북을 나눠 '설치용 플레이북'과 '운영용 플레이북'을 만들었다.
설치용 플레이북은 서버마다 최초에 한 번씩만 실행되어야했고 운영용 플레이북은 세부 데이터를 변경하여 몇번이고 재실행할 수 있었다.

곧 여러 서비스들에 대한 플레이북과 롤이 생성되면서 필요 이상으로 프로젝트의 규모가 커졌고 나는 이런 프로젝트를 감당하는 것에 어려움을 느끼기 시작했다.
설치용 플레이북을 한번만 실행시키기 위해 인간이 알아서 매번 수정해줘야하는 인벤토리는 더 큰 부담이었다.

최근 나는 Ansible 프로젝트를 재정리하며(거의 새로 만들며) 구성 플레이북이 운영 과정에 몇번씩 재시작되더라도 문제가 발생하지 않도록 수정했다.
최대한 세세하게 서버 구성을 수정하고 Ansible을 통하지 않은 구성 변경을 손상시키지 않도록 수정했다.

서버를 변경하는 모든 행위가 Ansible로 이루어진다면 이렇게까지 하지 않아도 됐을 것이다.
매번 파일을 통째로 카피하더라도 문제가 없었을 것이고 플레이북은 더 가벼웠을 것이다.

하지만 그런 허망한 꿈은 이루어질 수 없다는 것을 나는 뒤늦게야 알게되었다.

## 개발자 욕하기

교과서와 이론은 중요하다.
확실히 존중할 필요가 있다.
하지만 회사를 교과서로 만드는 것은 불가능하다.

Ansible을 도입하면서 나는 개발자들이 Ansible 프로젝트에 적극 동참하길 바랬다.
인프라팀은 개발자들의 플레이북을 보고 쉽게 어떤 오퍼레이션이 발생하는지 확인하고 동일한 작업을 쉽게 반복할 수 있을 것이라 생각했다.
예상대로라면 더이상 업데이트 되지 않는 문서로 싸울 일 없이 매뉴얼이 곧 실제 동작이 될 터였다.

이제부터 어떤 문제들이 내 계획에 섬광탄을 끼얹었는지 얘기해보자.

### 자원이 부족합니다

바로 옆자리에 개발자들이 앉아있고 나도 개발자였지만 내가 인프라팀에 속한 뒤 개발자들이 얼마나 적은 자원으로 활동하고 있는지 알지 못했다.
팀 변경, 포지션 변경, 그냥 이직이나 그냥 퇴직으로 인해 개발팀은 내가 개발팀에 속해있을 때보나 작아져있었다.

하지만 놀랍게도 프로젝트는 줄어들긴 커녕 늘어만 가고 있었다!

그 늘어난 프로젝트 중 하나가 내가 그렇게나 강요하던 Ansible 프로젝트이기도 했다.

Ansible이 아무리 쉽다고해도 학습비용이 전혀 없는 것은 아니다.
개발자들은 수많은 프로젝트에 둘러쌓여 추가로 학습비용을 지불하는 대신 비록 귀찮지만 익숙한 방식인 '각 서버에 직접 접근해서 직접 커맨드를 때리는 방식'을 선호했다.

열심히 Ansible 파이프라인과 유저들의 권한을 고민하고 구성했지만 개발자들은 Ansible Tower 접속할 시간조차 없었다.

### 같은 서비스, 다른 환경, 다른 구성?

개발 환경과 스테이지 환경, 프로덕션 환경은 가능하면 최대한 비슷하게 구성할 필요가 있다.
그렇지 않다면 당장 개발 환경에서 없던 문제가 환경 차이로 스테이지나 프로덕션에서 동작하지 않을 수도 있기 때문이다.

그리고 데브옵스 엔지니어로서 가장 큰 문제는 하나의 서비스를 위한 플레이북을 작성할 때 모든 구성을 위한 코드가 들어가야한다는 것이었다.

어느 정도의 차이냐면 CentOS 5와 CentOS 7 정도의 차이가 난다.
비유가 아니라 정말로 OS 메이저 버전이 다르다.

설정파일이나 로그파일의 위치 등은 서비스별로 다르고 역시 환경별로도 달랐다.
이러한 차이는 매뉴얼에 기록되기보단 며느리에게도 가르쳐주지 않기 방침으로 각 담당자들과 각 팀의 개발자들끼리만 구두로 공유되곤 했다.

이러한 문제가 발생한 원인이나 해결방법은 명확했다.
하지만 그전에 이전 단락의 제목을 힘차게 외쳐보자.

### 누가 내 설정을 수정하였는가?

Ansible 플레이북으로 `/etc/hosts` 파일을 관리하기 시작했다.
별도의 DNS 서버 없이 hosts 파일을 필요한 모든 서버에 집어넣고 동기화하여 도메인을 통한 통신과 코드 변경 없는 타겟서버 변경을 이뤄냈다.

이제 섬광탄을 던져보자.

레디스 서버 하나가 다운되었다.
급하게 슬레이브 레디스를 마스터로 승격시키고 호스트 파일을 수정하여 Ansible로 배포했다.

각 서버에서 다시 정상적으로 레디스에 연결하여 서비스를 이어나갔다.

한 곳만 빼고.

나도 모르는 사이에 `/etc/hosts` 파일이 수정되어있었다.
내가 배포하는 BLOCK 위쪽에 레디스를 바라보는 동일한 호스트가 기록되어있었고 해당 서버에서 동작하는 서비스는 여전히 다운된 레디스에게 연결을 시도하고 있었다.

나는 정말 모든 서버들에 내가 배포한 설정이 정상적으로 적용되고 있을지 더이상 신뢰할 수 없게 되었다.

### 실패와 좌절 그리고 포기, 하지만 다시 시작하기

이러한 상황에서는 서비스를 위한 플레이북을 개발하고 활용하는 것이 불가능함을 깨달았다.
결국 나는 서비스를 위한 플레이북 개발을 포기했다.

최근 나는 신규 프로젝트에 투입될 때 최대한 Ansible을 활용하며 실제 서버의 구성이나 상태를 개발자들에게 알리지 않고 있다.
개발자가 서버에서 할 수 있는 행위도 많이 줄여버렸다.
(이는 쉽게 달성할 수 있는 목표이다. 애시당초 개발자에게 root 권한을 주지말자.)

덕분에 사사로운 변경에도 내가 필요해졌지만 나는 좀 더 걱정없이 서버를 관리할 수 있게 되었고
개발자들의 리소스와 관계없이 내 리소스만으로 플레이북을 구성/관리할 수 있게 되었다.

## Ansible Tower 또는 AWX

CLI보다 GUI가 편하다는 얘기를 종종 듣는다.
확실히 GUI는 직관적인 표현을 통해 해당 도구의 사용법에 익숙하지 않은 이용자들도 기능을 쉽게 이해하고 사용할 수 있도록 도와준다.
실제로 어떠한 기능이나 서비스를 도입할 때 CLI 환경만 구성했을때보다 GUI를 제공했을 때 개발자들의 호응이 더 좋았다.

CLI에 익숙하지 않은 사용자들은 커맨드 사용에 어려움을 느끼거나 커맨드의 옵션들을 배우고 익히는 시간을 부정적으로 생각하기도한다.
Windows 사옹자의 경우 CLI 사용환경을 구성하는 것 자체를 어려워하기도 한다.
GUI를 주로 사용하는 사람들은 CLI는 GUI에 비해 학습비용이 크고 사용하기는 불편한 환경이라고 얘기한다.

다만 CLI에 익숙한 유저라면 얘기가 좀 달라진다.
주로 macOS나 리눅스를 사용하는 경우, 언제나 화면 한 켠에 터미널 에뮬레이터가 떠있으며 자신만의 쉘 구성이 존재하는 그런 경우 말이다.
CLI를 자주 다뤄본 유저에게 CLI는 빠르게 명령을 실행하고 쉽게 반복할 수 있는 환경이다.

Ansible Tower(커뮤니티 버전은 AWX)는 Ansible을 GUI에서 환경에서 사용할 수 있게 해주는 도구다.
웹상에서 클릭하는 것만으로 Ansible 플레이북을 실행하고 인벤토리를 관리할 수 있다.
Ansible 구성이나 커맨드를 알 필요 없이 데브옵스 엔지니어가 구성만 해놓으면 누구나 쉽게 클릭을 통해 Ansible을 다룰 수 있다.

처음 Ansible을 도입할 때부터 우리는 Ansible Tower를 염두해두고 있었다.
Ansible에서는 불가능한 Ansible Tower만의 기능에 끌렸던 것은 아니다.
우리는 CLI에 익숙하지 않은 개발자들의 적극적인 참여를 위해, 자신이 설정한 플레이북과 인벤토리 등을 다른 데브옵스 엔지니어가 쉽게 활용할 수 있도록 Ansible Tower를 도입하기로 했다.

그리고 나는 최근 Ansible Tower로 작업을 하는 것을 피곤해하고 있다.

### Ansible Tower의 UI 반응속도

우선 Ansible Tower는 생각보다 느리다.
작업속도 자체는 인스턴스의 속도 그대로지만 UI 조작의 반응이 생각보다 굼뜨다.
버튼을 클릭하고 빠르게 화면이 전환되지 않는다.

플레이북을 실제로 사용하기까지 꽤 많은 작업을 필요로 한다.
인벤토리에 서버를 추가하고 그룹을 만들어 넣고 프로젝트를 동기화하여 플레이북에 대한 잡 템플릿을 생성하고...
사실 잡 템플릿을 생성하는 것 외에는 커맨드라인에서도 동일한 작업을 하게 되겠지만 문제는 위에서 말한 것처럼 Ansible Tower의 UI 반응이 느리다.
특히나 여러대의 서버를 설정해야하는 경우는 답답한 UI의 반응속도를 버티기가 어렵다.

인벤토리를 스크립트로 관리하고 동기화할 수도 있으나 그렇게 할 경우 웹상에서의 인벤토리 조작은 포기해야한다.
나는 결국 웹을 통한 인벤토리 조작의 이점을 포기하고 스크립트 파일을 통해 인벤토리를 관리할지
아니면 내 편의를 위해 인벤토리를 스크립트로 관리할지 딜레마에 빠졌다.

### 프로젝트 동기화

Ansible은 Infrastructure as Code를 위한 도구이다.
플레이북과 인벤토리를 코드로 관리한다.
그럼 이제 Ansible Tower를 사용한다면?

플레이북과 롤, 인벤토리가 저장된 프로젝트를 저장소에 푸시하고 Ansible Tower에 접속하여 프로젝트를 동기화한다.
프로젝트가 동기화되는 것을 기다린 후 동기화가 끝나면 실제로 사용하기 위해 플레이북에 대한 잡 템플릿을 생성한다.

만약 내가 Ansible Tower를 사용하지 않고 CLI로 바로 작업한다면 `ansible-playbook` 커맨드를 실행하면 그것만으로 작업이 끝났을 것이다.

플레이북이나 롤에 문제가 있었다면 어떨까?
다시 코드를 수정하고 푸시하여 Ansible Tower에서 프로젝트를 동기화해야한다.

이 일련의 과정들이 위에서 언급한 끔찍한 반응속도와 함께 나를 힘들게 만든다.

### 배포를 위한 Ansible Tower

우리는 애플리케이션의 빌드/배포를 위해 Jenkins를 사용중이다.
나는 빌드는 Jenkins를 통해 하더라도 배포는 Ansible Tower를 사용하는 구성을 생각해봤다.

서버 구성과 운영을 위해 Ansible Tower는 이미 서버에 대한 정보를 가지고 있을 것이고 배포를 위한 접근권한 또한 가지고 있을 것이다.
Jenkins는 빌드만 진행하고 배포를 위한 권한은 제거한 뒤 Ansible Tower에게 위탁하면 Jenkins와 Ansible Tower의 권한과 역할이 명확해질 것이다.

그렇게 생각하던 시절에 나에게도 있었다.

Jenkins로 애플리케이션을 배포할때는 rsync를 사용한다.
배포는 간단하고 빠르게 진행된다.

하지만 Ansible은 배포를 위해 서버에 접속하고 접속한 서버의 정보를 취득하고 배포시에도 멱등성을 위한 처리가 들어간다.
애플리케이션 배포에 필요하지 않은 과정들이 너무 많다.
배포는 더 느려졌으며 나는 물론 개발자들 또한 이 과정을 한눈에 파악하기가 쉽지 않았다.

처음 이 작업을 진행할 때는 서버 리스트를 Ansible Tower가 인벤토리로 관리하고 배포할 서버들을 쉽게 변경하고 교체할 것을 예상했다.
아마 회사에서 사용하는 서비스당 복제서버가 정말 많거나 자주 교체된다면 약간의 단점을 감안하고 Ansible Tower를 통한 배포의 이점을 누렸을 것이다.
하지만 실제로 우리 회사의 서비스는 많아봐야 3개 정도의 고정된 서버에 배포되었고 서버가 변경되는 경우는 거의 없었다.

좋은 아키텍처란 상황에 맞는 아키텍처이지 교과서에 나오는 모범사례가 아니라는 것을 깨닫는 순간이었다.

### 그래서 Ansible Tower는 정말 필요할까

Ansible Tower에 대해 처음 이야기를 꺼냈을때는 GUI와 Ansible Tower를 사탄의 오픈소스 프로젝트인 것처럼 설명하긴 했지만
Ansible Tower는 충분히 매력적이며 누군가가 Ansible을 통한 시스템 구성과 운영의 자동화를 한다면 꼭 사용해보라고 얘기할 것이다.(긍정적인 의미로)

약간의 단점이나 혼란스러운 점이 있지만 어느 아키텍처나 도구도 각자 그들만의 단점이 있다.
그들이 힘을 발휘할만한 환경과 그렇지 않은 환경이 있고 그걸 활용할 수 있는 사람과 활용하지 못하는 사람, 그것을 활용할 수 있는 시기라는 게 있다.
적절히 필요한 곳에서 필요한 도구를 사용할 필요가 있다.

다만 내가 그랬던 것처럼 Ansible Tower를 사용한다고 해서 모든 작업을 Ansible Tower로 구성하려는 것은 말리고 싶다.
필요에 따라 Ansible Tower와 `ansible-*` 커맨드를 적절히 활용해주자.
사실 위에 적은 Ansible Tower에 대한 문제점?들은 내가 적절히 Ansible Tower를 활용하지 못했거나 너무 Ansible Tower에 집착해서 발생한 문제들이기도 하다.

(참고로 Ansible Tower는 CLI 도구가 존재한다! 하지만 별로 추천하진 않는다. ansible-playbook 커맨드에 비해 느리면서 관리는 GUI로 하는 것보다 불편하다.)

### 균형을 수호하기 위한 성공사례: cron 대신 Ansible Tower

우리는 주기적인 작업을 위해 cron을 사용한다.
cron은 아주 간편한 스케줄링 도구지만 리눅스의 철학이 너무 과감하게 들어갔다고 해야할까.
정말 작업 스케줄링만 한다.

cron으로 데이터베이스와 로그파일들을 백업하고 오래된 파일들을 정리하면서 가장 귀찮은 것은 정말 작업이 제시간에 정상적으로 동작하고 있는지 확인하는 것이다.
cron 자체적으로는 그런 기능이 없기 때문에 별도의 스크립트를 추가하여 해당 작업이 제대로 동작하는지 알림을 받거나 로그를 확인하거나 직접 결과물을 확인해야했다.

이런 과정들을 중간중간 삽입하는 것은 너무 피곤한 일이기 때문에 나는 이런 작업들을 Ansible Tower로 옮기기로 결정했다.

Ansible Tower를 통해 잡을 스케줄링하고 해당 잡의 결과를 Email과 Slack으로 받도록 처리했다.

결과는 아주 만족스러웠다.
매일 잡들의 실행 결과를 확인할 수 있으며 특정 잡이 실패했을 경우 어떤 과정에서 어떤 이유로 실패했는지를 확인할 수 있었다.

만약 서버를 교체할 일이 있더라도 서버의 크론을 찾아내 복사할 필요가 없다.
그저 인벤토리에서 기존 서버를 제거하고 새 서버를 등록하면 그만이다.

## Re:제로부터 시작하는 Ansible 환경

이러나 저러나 Ansible은 좋다.
아주 작은 부분부터 꽤 큰 부분까지 여러 영역에서 활용할 수 있다.
무엇보다 엔지니어의 리소스를 절약시켜주고 귀찮은 반복작업으로부터 도망칠 수 있게 해준다.
Ansible로 동시에 여러 대의 서버를 구축하며 '내가 동시에 3명분의 일을 하고 있으니 연봉도 3배로 달라'고 연봉협박을 해보자.
