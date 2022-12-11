---
layout: post
title: 'Terraform Docker 프로바이더: 네트워크 별칭 이슈'
categories:
- Issue Note
tags:
- Infra
- Terraform
- Automation
- Docker
---
Terraform Docker 프로바이더에 있는 `docker_service` 리소스는 서비스 네트워크를 설정하는 것은 가능하나 별칭을 걸 수 있는 방법이 없다.

만약 기존에 Terraform으로 관리하지 않던 Docker 서비스를 테라폼으로 임포트해서 사용할 경우 문제가 발생할 수 있다.

해당 서비스에서 별칭 사용하고 있다면 Terraform으로 다른 부분을 업데이트할 때 별칭 부분에는 변경이 없을 것처럼 표시되지만 실제로는 별칭을 삭제해버린다.

전반적으로 Terraform으로 생성하지 않은 리소스를 임포트해서 사용하는 것이 좀 불안하지 않나 싶다.
