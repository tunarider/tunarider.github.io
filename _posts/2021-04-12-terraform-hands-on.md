---
layout: post
title: 'Terraform 당장 써보기(AWS향 첨가)'
categories:
  - Hands-on
tags:
  - Infra
  - Terraform
  - Automation
  - AWS
---
Terraform은 인프라 형상관리도구이다.
코드로 형상을 작성하여 사용하는 플랫폼에 해당 형상을 구현한다.

Ansible과 유사하게 생각할 수 있는데, Ansible은 행위중심적이고 Terraform은 상태중심적이다.
Ansible이 "이 행동을 한다. 이미 했다면 하지 않는다."라고 한다면 Terraform은 "이 형상으로 상태를 만든다. 현재 상태가 형상과 동일하다면 바꾸지 않는다."라고 볼 수 있다.

그렇기 때문에 Ansible의 경우 별도의 *상태*를 저장하지 않는 반면 Terraform은 구현된 형상에 대한 로컬이나 S3, Consul 등 지정한 백엔드에 상태를 저장한다.

종종 Ansible과 Terraform을 비교할때 Terraform은 언제나 동일한 형상을 유지하고 Ansible을 매번 실행시마다 인프라의 실제 상태가 변경되는 것처럼 설명하는 경우가 있는데
이는 Ansible을 잘못 이해하고 있거나 잘못 사용하고 있어서 그렇다.

Ansible은 방식이 다를 뿐 실제 사용시 특정 형상을 유지하도록 개발할 수도 있다.
위에서 언급한 "이 행동을 한다. 이미 했다면 하지 않는다."라는 문구는 실제로는 "(이러한 상태로 만들기 위해)이 행동을 한다. 이미 했다면 (이미 원하는 상태일테니)하지 않는다."라고 볼 수도 있다.

사용하는 환경, 플랫폼 등 필요에 따라 Ansible과 Terraform을 적절히 사용하는 것이 좋다.

아래 실습예제는 Terraform을 통해 AWS 환경을 프로비저닝하는 방법을 전반적으로 보여준다.

실습 전 AWS 계정이 있어야하며 AWS CLI 환경이 미리 준비되어있어야한다.

## 설치

[공식문서](https://learn.hashicorp.com/tutorials/terraform/install-cli)에서 플랫폼별로 설치한다.

맥이라면 `brew`를 통해 쉽게 설치할 수 있다.

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

## 기초 실습

Terraform은 다양한 환경을 프로비저닝할 수 있다.
각 환경에 대한 지원이 Terraform 자체에 포함된 것이 아니라 프로바이더(Provider)라는 플러그인 형태로 제공된다.

AWS를 사용할 예정이므로 AWS 프로바이더를 설정한다.(`versions.tf`)
Terraform 버전 및 프로바이더 버전은 꼭 명시할 필요는 없으나 버전간 차이로 생기는 문제를
방지하기위해 직접 명시하는 것이 좋다.

아래는 패치버전은 무시하고 마이너버전까지만 맞춘 예시이다.

```hcl
terraform {
  required_version = ">= 0.14.4, < 0.15.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 3.24.1, < 3.25.0"
    }
  }
}
```

AWS API를 이용하기 위해 프로그래밍 방식 액세스 가능한 AWS 계정을 연결해줘야한다.
여러가지 방법이 있지만 AWS 크레덴셜 파일을 이용하는 방식을 사용한다.
프로바이더 설정에 크레덴셜 파일 경로와 사용할 프로파일명을 작성해준다.(`providers.tf`)

EKS에 의해 생성된 AWS 리소스는 `kubernetes.io/` 접두사를 사용하는 태그가 생성된다.
별도로 태그 관리를 할 경우 이러한 태그 관리와 설정이 충돌나므로 해당 태그는 무시하도록 `ignore_tags`를 설정한다.

```hcl
provider "aws" {
  region                  = var.region
  shared_credentials_file = pathexpand("~/.aws/credentials")
  profile                 = var.profile

  ignore_tags {
    key_prefixes = ["kubernetes.io/"]
  }
}
```

위 예제에서 `var.*`라는 값들이 보인다.
이 값들은 변수를 통해 적용시마다 유연하게 변경해서 사용할 수 있다.
다만 실제로 변수를 사용하기 위해서는 변수를 미리 선언해줘야한다.
변수 선언은 이후 과정에 다루도록 한다.

`terraform init` 명령어로 Terraform 프로젝트를 초기화한다.

초기화 과정에서 백엔드, 프로바이더를 확인하고 필요한 프로바이더를 설치한다.

```bash
terraform init
```

*출력:*

```
Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Installing hashicorp/aws v3.24.1...
- Installed hashicorp/aws v3.24.1 (signed by HashiCorp)

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

초기화가 완료되면 프로젝트 디렉토리에 `.terraform` 디렉토리와 `.terraform.lock.hcl` 파일이 생성된다.

`.terraform` 디렉토리에는 프로바이더가 설치된다.

`.terraform.lock.hcl` 파일은 `.terraform` 디렉토리 하위에 생성되는 다양한 파일들에 대한 잠금파일이다.
체크섬을 통해 프로바이더 파일 등의 버전이 일관되게 유지되도록 돕는다.

이제 `terraform plan` 명령어를 사용할 수 있다.
실제 형상을 적용하기 전에 무엇이 추가되고 무엇이 수정되는지를 미리 볼 수 있다.

현재는 아무런 형상도 작성되어있지 않기 때문에 이 상태에서 `terraform plan` 명령어를 실행해도 변경이 없을 것이다.

```bash
terraform plan
```

*출력:*

```
No changes. Infrastructure is up-to-date.

This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, no
actions need to be performed.
```

신규로 VPC 리소스를 하나 작성해보자.(`vpc.tf`)

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Provisioner = "terraform"
    Project     = var.project
    Name        = var.project
  }
}
```

Terraform 오브젝트 중에는 `resource` 오브젝트가 존재한다.
실제로 구성하고자하는 형상을 나타내는 오브젝트로 `resource "$resource_type" "resource_name"`의 형태로 작성한다.

위 예제는 `main`이라는 이름의 `aws_vpc`를 생성한다.
이후 다른 리소스에서 참조할때는 `aws_vpc.main`이라는 이름으로 불러올 수 있다.

[프로바이더 문서](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)에서 해당 리소스에 대한 설명을 볼 수 있다.

문서를 보면 인자(argument)와 속성(attribute)이 있다.

인자의 경우 해당 리소스에 적용할 설정값들이다.
최소한 `(Required)`로 표시된 모든 인자들은 필수로 작성해야 해당 리소스를 생성할 수 있다.

속성은 인자가 생성된 뒤 확인할 수 있는 값들이다.
VPC 리소스가 생성되고나면 ARN이나 ID를 알 수 있게 된다.
속성값은 다른 리소스에서 참조하거나 `output` 오브젝트로 출력할 수 있다.

결과적으로 위 코드는 아래와 같이 설명할 수 있다.

> `main`이라는 이름이 붙은 AWS VPC 리소스를 생성하는데
> CIDR은 `vpc_cidr`이라는 변수의 값으로 설정하고
> DNS 서포트와 DNS 호스트네임 기능을 활성화한다.
> `Provisioner` 태그에 `"terraform"`이라는 값을 설정하고
> `project`라는 변수값으로 `Project`, `Name` 태그를 설정한다.

`terraform plan` 명령어를 사용해보자. 에러 때문에 아직 실제로 변경되는 것은 없다.

*출력:*

```
Error: Reference to undeclared input variable

  on providers.tf line 2, in provider "aws":
   2:   region                  = var.region

An input variable with the name "region" has not been declared. This variable
can be declared with a variable "region" {} block.


Error: Reference to undeclared input variable

  on providers.tf line 4, in provider "aws":
   4:   profile                 = var.profile

An input variable with the name "profile" has not been declared. This variable
can be declared with a variable "profile" {} block.
```

기존에 AWS 프로바이더 설정에서 사용한 `var.region`, `var.profile`에 대한 에러가 발생한다.
`vpc.tf` 생성 전까지는 아무런 리소스도 없었기 때문에 변수 관련 에러가 발생하지 않았으나
VPC 리소스에 대한 형상이 생기면서 실제로 AWS 프로바이더가 동작했기 때문에 이번엔 에러가 발생했다.

변수를 정의하기 위한 파일 하나를 생성한다.(`variables.tf`)

```hcl
variable "region" {
  type = string
}

variable "profile" {
  type = string
}

variable "project" {
  type = string
}

variable "vpc_cidr" {
  type = string
}
```

변수를 정의하고 다시 `terraform plan` 명령어를 실행하여 변수를 입력할 수 있도록 대화형 쉘이 나온다.

변수를 매번 입력해줘도 되겠지만 자주 변경되지 않는 변수라면 변수파일을 만들어 미리 변수를 입력해둘 수 있다.

`terraform.tfvars` 파일을 생성한다.

```hcl
region = "ap-northeast-2"
profile = "default"

project = "my-example"
vpc_cidr = "192.168.0.0/16"
```

변수의 값들은 자신의 AWS 환경에 맞게 설정한다.

다시 `terraform plan` 명령어를 실행한다.

```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + arn                              = (known after apply)
      + assign_generated_ipv6_cidr_block = false
      + cidr_block                       = "192.168.0.0/16"
      + default_network_acl_id           = (known after apply)
      + default_route_table_id           = (known after apply)
      + default_security_group_id        = (known after apply)
      + dhcp_options_id                  = (known after apply)
      + enable_classiclink               = (known after apply)
      + enable_classiclink_dns_support   = (known after apply)
      + enable_dns_hostnames             = true
      + enable_dns_support               = true
      + id                               = (known after apply)
      + instance_tenancy                 = "default"
      + ipv6_association_id              = (known after apply)
      + ipv6_cidr_block                  = (known after apply)
      + main_route_table_id              = (known after apply)
      + owner_id                         = (known after apply)
      + tags                             = {
          + "Name"        = "my-example"
          + "Project"     = "my-example"
          + "Provisioner" = "terraform"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

AWS VPC "main"이 어떻게 생성될지 실행계획이 출력된다.
기존에 변수를 사용했던 부분들은 변수의 값으로 보여지게 된다.

`(known after apply)` 항목은 실제로 해당 리소스가 생성된 뒤에야 알게되는 값(속성)이다.

`terraform apply` 명령으를 실행하면 대화형 쉘로 확인과정을 거친 뒤 해당 리소스를 생성할 수 있다.

테라폼 실행을 자동화하거나 실제 계획과 적용 사이에 간격이 좀 있는 경우 실행계획을 파일로 출력하고 출력된 실행계획으로 적용하는 것이 더 편리할 수도 있다.

`terraform plan -out tfplan` 명령어로 계획파일을 출력한다.

*출력:*

```
...

This plan was saved to: tfplan

To perform exactly these actions, run the following command to apply:
    terraform apply "tfplan"
```

아래 명령어를 통해 출력된 계획파일을 통해 **AWS VPC를 실제로 생성한다.**
실행계획을 통해 `terraform apply` 명령어를 사용할 경우 확인과정을 건너뛰므로 **즉시 형상을 적용한다.**

```bash
terraform apply "tfplan"
```

*출력:*

```
aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 2s [id=vpc-01b9c18833d98b434]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate
```

출력 두번째줄을 확인해보면 AWS VPC가 생성되었으며 생성된 VPC의 ID가 `vpc-01b9c18833d98b434`라는 것을 알 수 있다.
AWS CLI나 AWS 콘솔에서 신규 VPC가 생성되었는지 확인해보자.

```bash
aws ec2 describe-vpcs --vpc-ids vpc-01b9c18833d98b434
```

*출력:*

```json
{
    "Vpcs": [
        {
            "CidrBlock": "192.168.0.0/16",
            "DhcpOptionsId": "dopt-57f5613e",
            "State": "available",
            "VpcId": "vpc-01b9c18833d98b434",
            "OwnerId": "<AccountID>",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-0ef8e1a40e0357159",
                    "CidrBlock": "192.168.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "IsDefault": false,
            "Tags": [
                {
                    "Key": "Project",
                    "Value": "my-example"
                },
                {
                    "Key": "Name",
                    "Value": "my-example"
                },
                {
                    "Key": "Provisioner",
                    "Value": "terraform"
                }
            ]
        }
    ]
}
```

VPC가 생성되었으니 해당 VPC에 서브넷을 추가해보자.

파일(`subnets.tf`)을 하나 추가하여 서브넷 리소스에 대해 정의한다.

```hcl
resource "aws_subnet" "public_primary" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "192.168.1.0/24"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true

  tags = {
    Provisioner = "terraform"
    Project     = var.project
    Name        = "public_primary"
  }
}
```

`vpc_id`라는 인자를 확인해보면 `aws_vpc.main.id`로 되어있는 것을 볼 수 있다.
서브넷 생성시 VPC를 지정해줘야하는데 방금 생성했던 `aws_vpc.main`을 지정한 것이다.

`aws_vpc` 리소스에는 `id`라는 속성이 있다.
`aws_vpc.main`을 생성하면서 실제 생성된 VPC의 ID가 해당 속성의 값으로 저장되고
그렇게 저장된 값을 `aws_subnet.public_primary` 리소스에서 가져다 사용하는 것이다.

`terraform apply` 명령어로 바로 적용해보자.

*출력:*

```
aws_vpc.main: Refreshing state... [id=vpc-01b9c18833d98b434]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_subnet.public_primary will be created
  + resource "aws_subnet" "public_primary" {
      + arn                             = (known after apply)
      + assign_ipv6_address_on_creation = false
      + availability_zone               = "ap-northeast-2a"
      + availability_zone_id            = (known after apply)
      + cidr_block                      = "192.168.1.0/24"
      + id                              = (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      + map_public_ip_on_launch         = true
      + owner_id                        = (known after apply)
      + tags                            = {
          + "Name"        = "public_primary"
          + "Project"     = "my-example"
          + "Provisioner" = "terraform"
        }
      + vpc_id                          = "vpc-01b9c18833d98b434"
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_subnet.public_primary: Creating...
aws_subnet.public_primary: Creation complete after 1s [id=subnet-090e26a3a3b6e5967]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

우선 첫번째 라인을 확인해보면 기존에 적용했던 `aws_vpc.main`에 대한 상태를 확인한다는 것을 알 수 있다.

현재 상태파일은 로컬 프로젝트 디렉토리 내에 `terraform.tfstate`로 저장이 되어있다.
계획/적용 단계에서 처음 하는 작업은 현재 저장된 상태와 실제 리소스의 상태를 동기화하는 것이다.

동기화가 끝나면 작성한 형상과 상태를 비교한다.
현재는 신규 생성이기 때문에 전부 `+(create)`로 출력되지만 이후에는 변경되는 부분들을 볼 기회기 올 것이다.

생성된 서브넷을 확인해보자.

```bash
aws ec2 describe-subnets --subnet-ids subnet-090e26a3a3b6e5967
```

*출력:*

```json
{
    "Subnets": [
        {
            "AvailabilityZone": "ap-northeast-2a",
            "AvailabilityZoneId": "apne2-az1",
            "AvailableIpAddressCount": 251,
            "CidrBlock": "192.168.1.0/24",
            "DefaultForAz": false,
            "MapPublicIpOnLaunch": true,
            "MapCustomerOwnedIpOnLaunch": false,
            "State": "available",
            "SubnetId": "subnet-090e26a3a3b6e5967",
            "VpcId": "vpc-01b9c18833d98b434",
            "OwnerId": "<AccountID>",
            "AssignIpv6AddressOnCreation": false,
            "Ipv6CidrBlockAssociationSet": [],
            "Tags": [
                {
                    "Key": "Provisioner",
                    "Value": "terraform"
                },
                {
                    "Key": "Name",
                    "Value": "public_primary"
                },
                {
                    "Key": "Project",
                    "Value": "my-example"
                }
            ],
            "SubnetArn": "arn:aws:ec2:ap-northeast-2:<AccountID>:subnet/subnet-090e26a3a3b6e5967"
        }
    ]
}
```

서브넷이 생성되었으니 이제 서브넷을 변경해보자.

`aws_subnet.public_primary.tags.Name`의 값을 `my-subnet`으로 변경하고 `terraform plan` 명령어를 실행해보자.

```hcl
resource "aws_subnet" "public_primary" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "192.168.1.0/24"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true

  tags = {
    Provisioner = "terraform"
    Project     = var.project
    Name        = "my-subnet"
  }
}
```

*출력:*

```
aws_vpc.main: Refreshing state... [id=vpc-01b9c18833d98b434]
aws_subnet.public_primary: Refreshing state... [id=subnet-090e26a3a3b6e5967]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_subnet.public_primary will be updated in-place
  ~ resource "aws_subnet" "public_primary" {
        id                              = "subnet-090e26a3a3b6e5967"
      ~ tags                            = {
          ~ "Name"        = "public_primary" -> "my-subnet"
            # (2 unchanged elements hidden)
        }
        # (8 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

`aws_subnet.public_primary.tags.Name` 항목의 값이 변경될 계획이다.

`~(update in-place)`는 기존 리소스를 삭제하지 않고 변경사항만 변경한다는 것이다.
물론 이게 **다운타임이 발생하지 않는다는 의미는 아니므로** 각 리소스의 특성을 잘 알고 있어야한다.

`terraform apply` 명령어로 변경사항을 적용하자.

*출력:*

```
aws_vpc.main: Refreshing state... [id=vpc-01b9c18833d98b434]
aws_subnet.public_primary: Refreshing state... [id=subnet-090e26a3a3b6e5967]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_subnet.public_primary will be updated in-place
  ~ resource "aws_subnet" "public_primary" {
        id                              = "subnet-090e26a3a3b6e5967"
      ~ tags                            = {
          ~ "Name"        = "public_primary" -> "my-subnet"
            # (2 unchanged elements hidden)
        }
        # (8 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_subnet.public_primary: Modifying... [id=subnet-090e26a3a3b6e5967]
aws_subnet.public_primary: Modifications complete after 1s [id=subnet-090e26a3a3b6e5967]

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

리소스를 교체 없이 업데이트 했으므로 동일한 ID로 확인해보면 태그가 변경된 것을 확인할 수 있다.

```bash
aws ec2 describe-subnets --subnet-ids subnet-090e26a3a3b6e5967
```

*출력:*

```json
{
    "Subnets": [
        {
            "AvailabilityZone": "ap-northeast-2a",
            "AvailabilityZoneId": "apne2-az1",
            "AvailableIpAddressCount": 251,
            "CidrBlock": "192.168.1.0/24",
            "DefaultForAz": false,
            "MapPublicIpOnLaunch": true,
            "MapCustomerOwnedIpOnLaunch": false,
            "State": "available",
            "SubnetId": "subnet-090e26a3a3b6e5967",
            "VpcId": "vpc-01b9c18833d98b434",
            "OwnerId": "<AccountID>",
            "AssignIpv6AddressOnCreation": false,
            "Ipv6CidrBlockAssociationSet": [],
            "Tags": [
                {
                    "Key": "Provisioner",
                    "Value": "terraform"
                },
                {
                    "Key": "Project",
                    "Value": "my-example"
                },
                {
                    "Key": "Name",
                    "Value": "my-subnet"
                }
            ],
            "SubnetArn": "arn:aws:ec2:ap-northeast-2:<AccountID>:subnet/subnet-090e26a3a3b6e5967"
        }
    ]
}
```

이번엔 교체가 필요한 변경을 진행해볼 것이다.

`aws_subnet.public_primary.cidr_block`의 값을 `192.168.1.0/24`에서 `192.168.2.0/28`로 변경해보자.

```hcl
resource "aws_subnet" "public_primary" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "192.168.2.0/28"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true

  tags = {
    Provisioner = "terraform"
    Project     = var.project
    Name        = "my-subnet"
  }
}
```

`terraform plan` 명령어로 적용 계획을 살펴보자.

*출력:*

```
aws_vpc.main: Refreshing state... [id=vpc-01b9c18833d98b434]
aws_subnet.public_primary: Refreshing state... [id=subnet-090e26a3a3b6e5967]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_subnet.public_primary must be replaced
-/+ resource "aws_subnet" "public_primary" {
      ~ arn                             = "arn:aws:ec2:ap-northeast-2:<AccountID>:subnet/subnet-090e26a3a3b6e5967" -> (known after apply)
      ~ availability_zone_id            = "apne2-az1" -> (known after apply)
      ~ cidr_block                      = "192.168.1.0/24" -> "192.168.2.0/24" # forces replacement
      ~ id                              = "subnet-090e26a3a3b6e5967" -> (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      ~ owner_id                        = "<AccountID>" -> (known after apply)
        tags                            = {
            "Name"        = "public_primary"
            "Project"     = "my-example"
            "Provisioner" = "terraform"
        }
        # (4 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

이전과 다르게 `~(update in-place)`가 아니라 `-/+(destroy and then create replacement)`로 되어있다.
이제 `terraform apply`로 적용시 리소스 ID가 변경될 것이다.

*출력:*

```
aws_vpc.main: Refreshing state... [id=vpc-01b9c18833d98b434]
aws_subnet.public_primary: Refreshing state... [id=subnet-0e4eb1f150ae85db1]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_subnet.public_primary must be replaced
-/+ resource "aws_subnet" "public_primary" {
      ~ arn                             = "arn:aws:ec2:ap-northeast-2:<AccountID>:subnet/subnet-0e4eb1f150ae85db1" -> (known after apply)
      ~ availability_zone_id            = "apne2-az1" -> (known after apply)
      ~ cidr_block                      = "192.168.1.0/24" -> "192.168.2.0/28" # forces replacement
      ~ id                              = "subnet-0e4eb1f150ae85db1" -> (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      ~ owner_id                        = "<AccountID>" -> (known after apply)
        tags                            = {
            "Name"        = "my-subnet"
            "Project"     = "my-example"
            "Provisioner" = "terraform"
        }
        # (4 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_subnet.public_primary: Destroying... [id=subnet-0e4eb1f150ae85db1]
aws_subnet.public_primary: Destruction complete after 0s
aws_subnet.public_primary: Creating...
aws_subnet.public_primary: Creation complete after 1s [id=subnet-0d5d13fe4dcf76ba5]

Apply complete! Resources: 1 added, 0 changed, 1 destroyed.
```

교체 작업이기 때문에 서브넷 `subnet-0e4eb1f150ae85db1`가 삭제되고 서브넷 `subnet-0d5d13fe4dcf76ba5`가 신규로 생성되었다.

이제 Terraform으로 생성한 리소스들을 제거해보자.

`terraform destroy` 명령어를 사용하면 리소스들을 일괄 제거할 수 있다.

*출력:*

```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_subnet.public_primary will be destroyed
  - resource "aws_subnet" "public_primary" {
      - arn                             = "arn:aws:ec2:ap-northeast-2:<AccountID>:subnet/subnet-0d5d13fe4dcf76ba5" -> null
      - assign_ipv6_address_on_creation = false -> null
      - availability_zone               = "ap-northeast-2a" -> null
      - availability_zone_id            = "apne2-az1" -> null
      - cidr_block                      = "192.168.2.0/28" -> null
      - id                              = "subnet-0d5d13fe4dcf76ba5" -> null
      - map_public_ip_on_launch         = true -> null
      - owner_id                        = "<AccountID>" -> null
      - tags                            = {
          - "Name"        = "my-subnet"
          - "Project"     = "my-example"
          - "Provisioner" = "terraform"
        } -> null
      - vpc_id                          = "vpc-01b9c18833d98b434" -> null
    }

  # aws_vpc.main will be destroyed
  - resource "aws_vpc" "main" {
      - arn                              = "arn:aws:ec2:ap-northeast-2:<AccountID>:vpc/vpc-01b9c18833d98b434" -> null
      - assign_generated_ipv6_cidr_block = false -> null
      - cidr_block                       = "192.168.0.0/16" -> null
      - default_network_acl_id           = "acl-0838df29bfce52bb2" -> null
      - default_route_table_id           = "rtb-001b9138fa81ffb45" -> null
      - default_security_group_id        = "sg-038629deef2bf33e9" -> null
      - dhcp_options_id                  = "dopt-57f5613e" -> null
      - enable_dns_hostnames             = true -> null
      - enable_dns_support               = true -> null
      - id                               = "vpc-01b9c18833d98b434" -> null
      - instance_tenancy                 = "default" -> null
      - main_route_table_id              = "rtb-001b9138fa81ffb45" -> null
      - owner_id                         = "<AccountID>" -> null
      - tags                             = {
          - "Name"        = "my-example"
          - "Project"     = "my-example"
          - "Provisioner" = "terraform"
        } -> null
    }

Plan: 0 to add, 0 to change, 2 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_subnet.public_primary: Destroying... [id=subnet-0d5d13fe4dcf76ba5]
aws_subnet.public_primary: Destruction complete after 0s
aws_vpc.main: Destroying... [id=vpc-01b9c18833d98b434]
aws_vpc.main: Destruction complete after 0s

Destroy complete! Resources: 2 destroyed.
```

이전에 생성했던 `aws_vpc.main` 리소스와 `aws_subnet.public_primary` 리소스가 제거되었다.

## 약간 더 나아가기

이번엔 기초 실습에서 진행했던 내용에 좀 더 살을 붙여보겠다.

인프라 환경은 언제나 완벽할 수 없으며 상황이 말그대로 *골때리게* 돌아갈수도 있다.

아래 실습예제들은 실제로 Terraform을 사용하면서 인프라 코드의 확장이나 문제 발생시의 대처를 위해 사용하게
될만한 옵션들의 사용법들을 설명한다.

### 리소스 타겟

리소스 중 일부만 적용하거나 삭제하고 싶은 경우가 있을 수 있다.
어느 정도 규모의 프로젝트를 관리하고 있을 때 다른 리소스는 무시하고 빠르게 특정 리소스만 업데이트하거나
잠깐 특정 리소스의 코드를 수정하고 있는 도중 갑작스럽게 다른 리소스에 발생한 문제에 대응해야 하는 경우가 그렇다.

일부 리소스만 적용하기 위해서는 `-target` 옵션을 사용한다.

```bash
terraform apply -target=aws_vpc.main
```

*출력:*

```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + arn                              = (known after apply)
      + assign_generated_ipv6_cidr_block = false
      + cidr_block                       = "192.168.0.0/16"
      + default_network_acl_id           = (known after apply)
      + default_route_table_id           = (known after apply)
      + default_security_group_id        = (known after apply)
      + dhcp_options_id                  = (known after apply)
      + enable_classiclink               = (known after apply)
      + enable_classiclink_dns_support   = (known after apply)
      + enable_dns_hostnames             = true
      + enable_dns_support               = true
      + id                               = (known after apply)
      + instance_tenancy                 = "default"
      + ipv6_association_id              = (known after apply)
      + ipv6_cidr_block                  = (known after apply)
      + main_route_table_id              = (known after apply)
      + owner_id                         = (known after apply)
      + tags                             = {
          + "Name"        = "my-example"
          + "Project"     = "my-example"
          + "Provisioner" = "terraform"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.


Warning: Resource targeting is in effect

You are creating a plan with the -target option, which means that the result
of this plan may not represent all of the changes requested by the current
configuration.

The -target option is not for routine use, and is provided only for
exceptional situations such as recovering from errors or mistakes, or when
Terraform specifically suggests to use it as part of an error message.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 2s [id=vpc-0455f5cefd013853f]

Warning: Applied changes may be incomplete

The plan was created with the -target option in effect, so some changes
requested in the configuration may have been ignored and the output values may
not be fully updated. Run the following command to verify that no other
changes are pending:
    terraform plan

Note that the -target option is not suitable for routine use, and is provided
only for exceptional situations such as recovering from errors or mistakes, or
when Terraform specifically suggests to use it as part of an error message.


Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

삭제할때도 동일하다.

```bash
terraform destroy -target=aws_vpc.main
```

*출력:*

```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_vpc.main will be destroyed
  - resource "aws_vpc" "main" {
      - arn                              = "arn:aws:ec2:ap-northeast-2:<AccountID>:vpc/vpc-0455f5cefd013853f" -> null
      - assign_generated_ipv6_cidr_block = false -> null
      - cidr_block                       = "192.168.0.0/16" -> null
      - default_network_acl_id           = "acl-09bc1b62bc4b22ac7" -> null
      - default_route_table_id           = "rtb-08cd4578bb5bf8552" -> null
      - default_security_group_id        = "sg-09240b1ecafda77f2" -> null
      - dhcp_options_id                  = "dopt-57f5613e" -> null
      - enable_dns_hostnames             = true -> null
      - enable_dns_support               = true -> null
      - id                               = "vpc-0455f5cefd013853f" -> null
      - instance_tenancy                 = "default" -> null
      - main_route_table_id              = "rtb-08cd4578bb5bf8552" -> null
      - owner_id                         = "<AccountID>" -> null
      - tags                             = {
          - "Name"        = "my-example"
          - "Project"     = "my-example"
          - "Provisioner" = "terraform"
        } -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.


Warning: Resource targeting is in effect

You are creating a plan with the -target option, which means that the result
of this plan may not represent all of the changes requested by the current
configuration.

The -target option is not for routine use, and is provided only for
exceptional situations such as recovering from errors or mistakes, or when
Terraform specifically suggests to use it as part of an error message.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_vpc.main: Destroying... [id=vpc-0455f5cefd013853f]
aws_vpc.main: Destruction complete after 0s

Warning: Applied changes may be incomplete

The plan was created with the -target option in effect, so some changes
requested in the configuration may have been ignored and the output values may
not be fully updated. Run the following command to verify that no other
changes are pending:
    terraform plan

Note that the -target option is not suitable for routine use, and is provided
only for exceptional situations such as recovering from errors or mistakes, or
when Terraform specifically suggests to use it as part of an error message.


Destroy complete! Resources: 1 destroyed.
```

### 리소스 상태 관리

리소스의 상태는 지정한 백엔드에 저장된다.

이 단락에서는 백엔드에 저장된 리소스를 관리하는 방법을 보여준다.

예제에서는 로컬파일에 상태를 저장한다.
실습을 위해 우선 기존에 작성해놓은 `aws_vpc.main`과 `aws_subnet.public_primary`를 Terraform으로 생성해두자.

#### 리소스의 상태 확인

리소스의 상태를 백엔드에 직접 들어가서 내용을 보거나 명령어를 통해 확인할 수 있다.

`terraform.tfstate` 파일을 열어 내용을 확인해보자

```
{
  "version": 4,
  "terraform_version": "0.14.5",
  "serial": 30,
  "lineage": "f944028f-21c2-ba6f-7a16-db168e6df181",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_subnet",
      "name": "public_primary",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "arn": "arn:aws:ec2:ap-northeast-2:<AccountID>:subnet/subnet-045c721b8d8506065",
            "assign_ipv6_address_on_creation": false,
            "availability_zone": "ap-northeast-2a",
            "availability_zone_id": "apne2-az1",
            "cidr_block": "192.168.2.0/28",
            "id": "subnet-045c721b8d8506065",
            "ipv6_cidr_block": "",
            "ipv6_cidr_block_association_id": "",
            "map_public_ip_on_launch": true,
...
```

기존에 생성했던 리소스의 정보를 확인할 수 있다.

긴급한 상황에 극단적인 디버깅을 위해 직접 상태파일을 읽거나 수정할 수도 있지만 보통은 명령어를 통해 쉽게 상태를 관리할 수 있다.

`terraform state list` 명령어로 현재 저장된 리소스 상태들을 확인해보자.

*출력:*

```
aws_subnet.public_primary
aws_vpc.main
```

`terraform state show` 명령어를 사용하여 특정 리소스의 상태를 확인할 수 있다.

```bash
terraform state show aws_vpc.main
```

*출력:*

```
# aws_vpc.main:
resource "aws_vpc" "main" {
    arn                              = "arn:aws:ec2:ap-northeast-2:<AccountID>:vpc/vpc-0f96a643801532e14"
    assign_generated_ipv6_cidr_block = false
    cidr_block                       = "192.168.0.0/16"
    default_network_acl_id           = "acl-00dff78ea3295dba3"
    default_route_table_id           = "rtb-0c6426751880b1b00"
    default_security_group_id        = "sg-0068d74e73045d59f"
    dhcp_options_id                  = "dopt-57f5613e"
    enable_dns_hostnames             = true
    enable_dns_support               = true
    id                               = "vpc-0f96a643801532e14"
    instance_tenancy                 = "default"
    main_route_table_id              = "rtb-0c6426751880b1b00"
    owner_id                         = "<AccountID>"
    tags                             = {
        "Name"        = "my-example"
        "Project"     = "my-example"
        "Provisioner" = "terraform"
    }
}
```

#### 리소스 상태 삭제

더이상 Terraform을 통해 특정 리소스를 관리하지 않게 되거나
긴급상황에 수동으로 리소스를 삭제/교체하는 경우
또는 Terraform이나 프로바이더의 버그에 대응하기 위해
리소스는 그대로 냅두고 저장된 상태만 삭제하고 싶은 경우가 생길 수 있다.

이런 경우 `terrafom state rm` 명령어를 사용한다.

```bash
terraform state rm aws_vpc.main
```

*출력:*

```
Removed aws_vpc.main
Successfully removed 1 resource instance(s).
```

`terraform state list` 명령어로 확인해보면 리소스의 상태가 삭제된 것을 확인할 수 있다.

*출력:*

```
aws_subnet.public_primary
```

하지만 AWS CLI로 확인해보면 리소스는 여전히 남아있다.

```bash
aws ec2 describe-vpcs --vpc-ids vpc-0f96a643801532e14
```

*출력:*

```json
{
    "Vpcs": [
        {
            "CidrBlock": "192.168.0.0/16",
            "DhcpOptionsId": "dopt-57f5613e",
            "State": "available",
            "VpcId": "vpc-0f96a643801532e14",
            "OwnerId": "<AccountID>",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-0d39701ad94958208",
                    "CidrBlock": "192.168.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "IsDefault": false,
            "Tags": [
                {
                    "Key": "Provisioner",
                    "Value": "terraform"
                },
                {
                    "Key": "Project",
                    "Value": "my-example"
                },
                {
                    "Key": "Name",
                    "Value": "my-example"
                }
            ]
        }
    ]
}
```

`terraform plan` 명령어를 사용해보면 VPC에 대한 상태정보가 없기 때문에
해당 리소스가 아직 생성된줄 모르고 VPC를 신규 생성하려고 한다.

```bash
terraform plan
```

*출력:*

```
aws_subnet.public_primary: Refreshing state... [id=subnet-045c721b8d8506065]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_subnet.public_primary must be replaced
-/+ resource "aws_subnet" "public_primary" {
      ~ arn                             = "arn:aws:ec2:ap-northeast-2:<AccountID>:subnet/subnet-045c721b8d8506065" -> (known after apply)
      ~ availability_zone_id            = "apne2-az1" -> (known after apply)
      ~ id                              = "subnet-045c721b8d8506065" -> (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      ~ owner_id                        = "<AccountID>" -> (known after apply)
        tags                            = {
            "Name"        = "my-subnet"
            "Project"     = "my-example"
            "Provisioner" = "terraform"
        }
      ~ vpc_id                          = "vpc-0f96a643801532e14" -> (known after apply) # forces replacement
        # (4 unchanged attributes hidden)
    }

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + arn                              = (known after apply)
      + assign_generated_ipv6_cidr_block = false
      + cidr_block                       = "192.168.0.0/16"
      + default_network_acl_id           = (known after apply)
      + default_route_table_id           = (known after apply)
      + default_security_group_id        = (known after apply)
      + dhcp_options_id                  = (known after apply)
      + enable_classiclink               = (known after apply)
      + enable_classiclink_dns_support   = (known after apply)
      + enable_dns_hostnames             = true
      + enable_dns_support               = true
      + id                               = (known after apply)
      + instance_tenancy                 = "default"
      + ipv6_association_id              = (known after apply)
      + ipv6_cidr_block                  = (known after apply)
      + main_route_table_id              = (known after apply)
      + owner_id                         = (known after apply)
      + tags                             = {
          + "Name"        = "my-example"
          + "Project"     = "my-example"
          + "Provisioner" = "terraform"
        }
    }

Plan: 2 to add, 0 to change, 1 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

#### 리소스 임포트

기존에 다른 방법으로 생성한 리소스를 Terraform으로 관리하기 위해 리소스를 임포트할 수 있다.

마침 이전 실습에서 `aws_vpc.main`의 상태를 삭제해둔 상태이므로 `aws_vpc.main`을 임포트해보자.

```bash
terraform import aws_vpc.main vpc-0f96a643801532e14
```

*출력:*

```
aws_vpc.main: Importing from ID "vpc-0f96a643801532e14"...
aws_vpc.main: Import prepared!
  Prepared aws_vpc for import
aws_vpc.main: Refreshing state... [id=vpc-0f96a643801532e14]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

임포트가 완료되면 다시 `terraform state list`나 `terraform plan`을 실행했을때
상태를 정상적으로 가져온 것을 확인할 수 있다.

### 데이터 오브젝트

몇몇 인프라 환경이나 구성에서는 Terraform으로 관리하기 애매한 리소스가 생기기도 한다.

특정 리소스를 생성할때 함께 생성된 자동 생성 리소스거나
읽기 전용 리소스일 수도 있고
혹은 단순히 관리상 Terraform을 통한 변경을 제한하려는 경우일 수도 있다.

VPC를 수동으로 생성하고 수동으로만 변경하되 서브넷은 Terraform으로 관리하길 원한다고 가정해보자.

서브넷이나 프로젝트의 다른 리소스들이 VPC에 대한 정보를 필요로 한다면 이 모든 것들을 변수로 관리하거나
하드코딩하고서 매번 VPC에 대한 정보가 변경될때마다 신경써줘야할 것이다.

이때 사용하는 것이 데이터 오브젝트이다.
리소스 오브젝트와는 다르게 데이터 오브젝트는 정보를 사용하기 위해 사용한다.

기존 `vpc.tf` 파일을 수정하여 리소스 오브젝트를 데이터 오브젝트로 변경해보자.

```hcl
data "aws_vpc" "main" {
  id = "vpc-0f96a643801532e14"
}
```

코드에서 리소스를 삭제했으니 상태 또한 지워줘야한다.
`terraform state rm aws_vpc.main` 명령어로 기존에 저장되어있던 상태를 삭제해주자.

`terraform plan`이나 `terraform apply` 같은 명령어를 사용해보면 아래와 같은 오류가 발생할 것이다.

```
Error: Reference to undeclared resource

  on subnets.tf line 2, in resource "aws_subnet" "public_primary":
   2:   vpc_id                  = aws_vpc.main.id

A managed resource "aws_vpc" "main" has not been declared in the root module.
```

서브넷에서 `aws_vpc.main.id`를 필요로 하는데 `aws_vpc.main`이라는 **리소스**가 없다는 것이다.

리소스 오브젝트를 참조할때는 리소스 타입과 리소스 이름만 써주면 됐지만
데이터 오브젝트의 경우 앞에 `data.`라는 접두사를 붙여야한다.

`subnets.tf` 파일을 열어 항목을 수정해주자.

```hcl
resource "aws_subnet" "public_primary" {
  vpc_id                  = data.aws_vpc.main.id
  cidr_block              = "192.168.2.0/28"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true

  tags = {
    Provisioner = "terraform"
    Project     = var.project
    Name        = "my-subnet"
  }
}
```

이제 다시 `terraform plan` 명령어를 사용하면 정상적으로 데이터 오브젝트를 사용한다는 것을 알 수 있다.

*출력:*

```
aws_subnet.public_primary: Refreshing state... [id=subnet-045c721b8d8506065]

No changes. Infrastructure is up-to-date.

This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, no
actions need to be performed.
```
