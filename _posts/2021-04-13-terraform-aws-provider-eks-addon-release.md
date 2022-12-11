---
layout: post
title: 'Terraform AWS 프로바이더 3.37.0: EKS Addon 리소스 추가'
categories:
- Note
tags:
- Infra
- Terraform
- Automation
- AWS
---
Terraform AWS 프로바이더 3.37.0 마일스톤에 EKS Addon 리소스를 추가하는 PR이 등록되었다.

현재(3.36.x 이하)는 EKS Addon 리소스가 없어서 `aws-node`에 VPC CNI 정책을 Terrafrom으로 부여할 수가 없다.
Terraform으로 온전한 EKS 클러스터를 구축할 수 없으며 노드 그룹 생성 전후로 수동으로 EKS Addon을 설정해줘야한다.

3.37.0 릴리즈에 해당 리소스가 추가되면 이제 Terraform만으로 온전한 EKS 클러스터를 생성할 수 있다.

# 참고

[Add eks addon resource, data_source](https://github.com/hashicorp/terraform-provider-aws/pull/16972)
