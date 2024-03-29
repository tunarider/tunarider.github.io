---
layout: post
title: 'Snowflake Server & Phoenix Server'
categories:
- Note
tags:
- Infra
- Virtual Machine
- Snowflake Server
- Phoenix Server
---

## Snowflakes Server

서버를 관리하다보면 계속 설정을 변경하고 새 프로그램을 설치하거나 기존 프로그램을 업데이트하게 된다.
시간이 지나면서 서버의 변경 이력은 계속 쌓이게 되는데 이러한 서버를 Snowflake Server 라고 부른다.

서버에 설치되는 프로그램들과 설정들은 서로 연결되거나 영향을 주고받게 되는데 미세한 차이만으로도 프로그램들과 설정들이 충돌하며 정상적인 작동을 보장할 수 없게되는 경우가 있다.
동일한 구성의 서버를 새로 구축하려고할 때 변경 이력이 누락되면 서버간의 구성이 약간씩 달라질 수 있으며 결국 새로 구축한 서버가 기존과 다르게 동작하거나 아예 동작 자체를 못할 수도 있다.

변경 이력 중 이제와서는 사실상 필요없는 과정이 있더라도 해당 과정이 다른 프로그램이나 설정에 영향을 주고 그 차이로 인해 프로그램이 다르게 동작한다면 정확한 문제를 판단하기 어려워지며
결국 그것 하나만을 위해 해당 과정을 반복하게될 수도 있다.

최악의 경우 변경 이력이 제대로 남지 않아 아예 기존에 어떤 식으로 서버가 구성되었는지를 추측조차 하기 힘들어져서 신규 서버를 거의 새로운 구성으로 구축하게될 수도 있다.
당연히 구축 과정에서 설정을 다시 처음부터 하게 되고 기존과 다르게 동작할 가능성은 커지게 된다.

또한 서버가 이중화되어 있는 경우 구성 변경시 모든 서버에 해당 변경사항을 반영해주어야한다.
이때 이중화된 서버간 설정이 약간씩 다르게 적용될 경우 디버깅에 많은 시간을 소모하게 된다.

Puppet, Chef 와 같은 자동화 솔루션을 통해 서버 구성을 코드로 관리하여 이러한 문제들을 어느 정도 해소할 수 있다.
서버를 구성하는 코드는 버전 관리 시스템을 통해 이력이 관리되며 구성을 적용할 모든 서버는 동일하게 구축된다.

다만 긴급상황에서 서버에 직접 연결하여 무언가를 변경해야하는 상황은 결국 발생하기 마련이며 이러한 변경사항을이 이후에 영향을 줄 수 있다는 것을 생각해야한다.

## Phoenix Server

Phoenix Server 는 서버에 변경이 필요할 때 기존 서버에 변경사항을 추가하는대신 아예 새로운 서버를 생성한다.
OS 설치부터 프로그램 설치, 설정 변경까지 다시 처음부터 진행하게 된다.
새롭게 생성된 서버는 테스트를 거친 뒤 로드밸런서 등을 통해 기존 서버와 교체된다.

Terraform, Packer, Ansible 같은 툴을 활용하여 프로비저닝 과정을 자동화하고 Foundation Image 를 생성해두면 빠르고 간단하게 서버를 구성하고 이력을 관리할 수 있다.

## 참고

[조대협 - Phoenix (피닉스) 서버 패턴](https://bcho.tistory.com/1224)  
[Martin Fowler - Snowflake Server](https://martinfowler.com/bliki/SnowflakeServer.html)  
[Martin Fowler - Phoenix Server](https://martinfowler.com/bliki/PhoenixServer.html)  
