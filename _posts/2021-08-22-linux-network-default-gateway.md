---
layout: post
title: 'Linux Network Default Route'
categories:
- Issue Note
tags:
- Infra
- Linux
- Network
---
Linux 네트워크 스크립트에 `DEFROUTE` 설정이 있다.

`DEFROUTE=yes`로 설정시 해당 인터페이스가 디폴트 게이트웨이로 설정된다.
여러 개의 인터페이스에서 `DEFROUTE` 설정시 마지막 인터페이스가 사용된다.

최근 서버에 신규 네트워크 인터페이스를 추가하다 `DEFROUTE`를 설정해버리는 바람에 서버가 잠시동안 먹통이 된 일이 있었다.
간단한 설정이라도 하나하나 꼼꼼히 살펴볼 필요가 있다.
