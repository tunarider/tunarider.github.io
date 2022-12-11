---
layout: post
title: '"truncate"와 Null 캐릭터'
summary: 파일을 무의미한 Null 캐릭터로 도배하기
categories:
  - Troubleshooting
tags:
  - Infra
  - Linux
  - File System
  - log4j
  - logrotate
  - du
  - df
  - ls
---

## 결론

프로세스가 파일을 오픈할 때 `O_APPEND` 플래그를 사용하지 않으면
해당 파일의 내용을 비워버리더라도 FD 상의 오프셋이 유지되어 파일의 앞부분을
널 캐릭터로 도배할 수 있다.

## 진짜 문제

특정 로그파일이 종종 널 캐릭터(보통 `^@`로 표시된다.)로 도배되는 경우가 있었다.
널 캐릭터가 너무 많이 찍혀있었기 때문에 로그 파일을 보기도 힘들고 대체 무슨 일이 일어나고 있는지도 확인하기가 어려운 상황.

우선 파일을 읽는 것부터가 고역이었고 해당 로그 파일이 용량을 너무 차지하고 있었으므로(`ls` 커맨드로 확인했음.) `fallocate` 커맨드를 사용했다.
`fallocate` 커맨드는 파일 공간을 할당하거나 해제한다.

```bash
fallocate -c -l 10G out.log
# -c OFFSET부터 LENGTH만큼 공간을 지운다.
# -l LENGTH 설정
```

놀랍게도 효과 없음.
동일한 파일을 복사해서 시도해보니 잘 됨.
이쯤에서 오픈된 파일에선 동작이 다를 것임을 짐작.

### 기묘한 통찰력으로 원인 파악하기

프로세스가 파일 오픈 시 `O_APPEND` 플래그를 사용하면 쓰기 작업시마다 파일의 끝으로 포인터를 옮기게 된다.
하지만 `O_APPEND` 플래그를 사용하지 않을 경우 별도로 포인터의 위치를 변경하지 않는 이상 기존 포인터에서 쓰기 작업을 하게 된다.

파일 플래그를 확인해보자.

```bash
$ sudo lsof +fg out.log
... FILE-FLAG ...
...      W,LG ...
```

프로세스에서 `O_APPEND` 플래그없이 기존 오프셋을 유지한채로 로그를 써버리니 로그 파일을 아무리 비워도
파일을 오프셋만큼 할당받아 로그를 쓰고 "비워졌을" 앞부분은 널 캐릭터로 채워지고 있는 거였다.

### 그럼 누가 파일의 내용을 비웠을까

`logrotate`에는 `copytruncate`라는 옵션이 있다.
파일을 지우고 새로 만드는 대신 원본 파일을 별도로 카피한 다음 원본 파일을 비워버리는 기능이다.
원본 파일을 계속 유지함으로써 안전하고 지속적으로 로그파일을 쓰겠다는 건데...

매일 로그가 로테이션될 때마다 "파일을 카피하고 -> 파일 내용을 비웠지만 -> FD의 오프셋이 그대로...! -> 널 캐릭터 파티다!"라는 상황이었다.

### 해결(했다고 주장 중)

결국엔 프로그램 쪽에서 로그 파일을 오픈하는 방식이 좀 달라져야할 것 같아 개발팀에 해당 내용 전달했다.

`logroate`는 의미가 없으므로 현재 비활성화 시킨 상태.
(설정파일 확장자를 `.disabled`로 하면 해당 설정파일은 동작하지 않는다. 이 외에도 무시하는 확장자가 몇 개 있는데 그건 매뉴얼 참고.)

## 가짜 문제

이전에 '용량이 부족해서'라는 설명을 했는데 여기엔 내 착각이 포함되어있다.

70GB가 넘는 로그 파일을 다른 파티션으로 옮긴 뒤 `df`로 남은 디스크 용량을 확인해봤다.
남은 용량 9GB.
어라?

`du`로 해당 파일이 들어있는 디렉토리(다른 파일들도 있었음.) 용량을 확인한다.
30GB.
어라?

`ls`는 해당 파일이 "할당받은" 용량을 표시하고
`du`는 해당 파일이 실제로 디스크에서 사용하는 용량을 표시한다.
파일 내에서 널 캐릭터 부분까지 파일이 공간을 할당받았으나 실제로 디스크에는
널 캐릭터를 포함한 부분은 디스크에 할당된 상태는 아니었던 것이다.(sparse file 참고)

## 이건 별개

비슷하게 `du`와 `df`에도 차이가 있는데 `df`는 파일시스템의 디스크 블럭을 참고하여 용량을 계산하고
`du`는 해당 디렉토리의 파일트리를 따라 이동하면서 파일에 할당된 블럭 수를 더하여 계산한다.
프로세스에서 파일을 오픈한 상태에서 해당 파일을 삭제하면 해당 파일이 여전히 디스크 상으론 남아있으나
디렉토리 상에는 지워져서 `du`보다 `df`의 용량이 더 크게 표시되는 경우가 있다.

이런 경우 해당 파일을 참조하는 FD를 찾아 관련 프로세스를 재시작해주면 된다.

## 참고

[fallocate manual](https://man7.org/linux/man-pages/man1/fallocate.1.html)

[how to find out which flasgs the process used to open a file](https://www.linuxquestions.org/questions/programming-9/how-to-find-out-which-flags-the-process-used-to-open-a-file-4175542258/)

[why ls and du show different size](https://superuser.com/questions/94217/why-ls-and-du-show-different-size)

[async log4j2 and logrotate not working correctly truncate is broken](https://stackoverflow.com/questions/71128523/async-log4j2-and-logrotate-not-working-correctly-truncate-is-broken)

[df/du 명령어 차이점](https://support.bespinglobal.com/ko/support/solutions/articles/73000560685--linux-df-du-명령어-차이점-차이날-때-해결-방법)

[find and remove large files that are open  have been deleted](https://unix.stackexchange.com/questions/68523/find-and-remove-large-files-that-are-open-but-have-been-deleted)

