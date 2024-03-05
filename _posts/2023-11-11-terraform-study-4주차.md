---
layout: post
title: 'Terraform Study #4'
date: 2023-11-11 15:17 +0900
ddescription: 'Terraform Study 4주차'
category: [Study, Terraform]
tags: [Terraform, IaC, Study]
image:
  path: /assets/img/logo/Terraform/logo.png
  alt: Terraform Logo
pin: false
math: true
mermaid: true
---
‘테라폼으로 시작하는 IaC’ 책으로 진행하는 Terraform 스터디[T101] 4주차 정리내용입니다.
<!--more-->

# 4주차
이번 주차에서는 기본 문법을 넘어, 코드를 구조화하고 협업하는 방법에 대해 배운다. 구체적으로는 module과 state에 대해 학습하며, 협업과 관련된 내용은 5주차에서 더 자세하게 다룬다.

## State


아래는 1주차 정리내용이다. 테라폼에서는 State 파일을 Serial을 기준으로 backup 관리한다.

> Terraform의 **`.tfstate`** 파일 내의 **`serial`** 값은 상태 파일의 버전을 나타내며, 동시성 제어와 데이터 무결성 확인에도 중요한 역할을 합니다. 이 값은 Terraform 명령이 실행될 때마다 자동으로 증가하여, 상태 파일의 최신성과 일관성을 유지합니다. 그렇기에 backup 파일이 현재 state 파일보다 serial 번호가 낮다.
> 

**이론적인 내용**

상태 파일은 배포할 때마다 변경되는 프라이빗 API이며, 오직 테라폼 내부에서 사용용도이니 직접 편집하거나 작성해서는 안된다. (파일 내부 데이터를 통해, API를 요청하는 것 같다.)

만약, 테라폼을 통해 협업을 진행해야 한다면 state 파일을 관리해야 한다. 이때는 1주차때 진행했던 원격 백엔드를 사용한다. 

팀 단위 운영시 필요한 점은 다음과 같다.

- state 파일의 공유 스토리지
- Locking(한명에 한명씩)
- 파일 격리(dev, stage 등 환경 별 격리가 필요)

아래는 VCS를 사용할 때, 발생하는 문제점이다.

- VCS: 수동으로 상태파일을 push, pull 해야 하니 휴먼에러가 발생할 수 있다.++ Lock 기능이 없다.

결국, 테라폼을 지원하는 원격 백엔드를 사용해야 한다. S3, Terraform Cloud 등이 있다.

## 관련 실습

State 파일에는 리소스에 대한 모든 것이 담겨있다. 아래의 실습을 통해, 상태 파일 관리의 중요성을 확인해본다.

- 패스워드 리소스 코드

```go
resource "random_password" "mypw" {
  length           = 16
  special          = true
  override_special = "!#$%"
}
```

아래의 명령어를 통해 리소스 생성

```bash
terraform init && terraform plan && terraform apply -auto-approve
```

테라폼 명령어를 통해 리소스를 확인하면, 시크릿 정보는 알려주지 않는다.

```go
❯ terraform state show random_password.mypw

# random_password.mypw:
resource "random_password" "mypw" {
    bcrypt_hash      = (sensitive value)
    id               = "none"
    length           = 16
    lower            = true
    min_lower        = 0
    min_numeric      = 0
    min_special      = 0
    min_upper        = 0
    number           = true
    numeric          = true
    override_special = "!#$%"
    result           = (sensitive value)
    special          = true
    upper            = true
}
```

하지만, state 파일을 확인해보면 리소스에 대한 정보가 전부 존재한다. 

```bash
{
  ...
  "resources": [
    {
      "mode": "managed",
      "type": "random_password",
     ...
      "instances": [
        {
          "schema_version": 3,
          "attributes": {
            "bcrypt_hash": "$2a$10$pLMnmRKSY52ageoVumPlpuP5dyo2GZpomOxo6MsQetO/F28dR2ge2",
            "id": "none",
          ...
            "result": "CLTOsYB9zWifY9WT",
            "special": true,
            "upper": true
          },
          "sensitive_attributes": []
        }
      ]
    }
  ],
  "check_results": null
}
```

테라폼 콘솔에서도 sensitive value는 보이지 않는다.

```bash
echo "random_password.mypw" | terraform console
{
  "bcrypt_hash" = (sensitive value)
  "id" = "none"
  "keepers" = tomap(null) /* of string */
  "length" = 16
  ...
  "override_special" = "!#$%"
  "result" = (sensitive value)
  "special" = true
  "upper" = true
}
```

## .tfstate

Terraform의 **`.tfstate`** 파일 내의 **`serial`** 값은 상태 파일의 버전을 나타내며, 동시성 제어와 데이터 무결성 확인에도 중요한 역할을 합니다. 이 값은 Terraform 명령이 실행될 때마다 자동으로 증가하여, 상태 파일의 최신성과 일관성을 유지합니다. 그렇기에 backup 파일이 현재 state 파일보다 serial 번호가 낮다.

| 유형 | 구성 리소스 정의 | State 구성 데이터 | 실제 리소스 | 기본 예상 동작 |
| ---- | ---------------- | ----------------- | ----------- | -------------- |
| 1    | 있음             |                   |             | 리소스 생성    |
| 2    | 있음             | 있음              |             | 리소스 생성    |
| 3    | 있음             | 있음              | 있음        | 동작 없음      |
| 4    |                  | 있음              | 있음        | 리소스 삭제    |
| 5    |                  |                   | 있음        | 동작 없음      |
| 6    | 있음             |                   | 있음        |                |
- `-refresh=false` 옵션을 사용하면, 현재의 state파일과 테라폼코드를 비교하여 그대로 적용한다. 원격 리소스의 실제 상태는 확인하지 않음. 그렇기에 만약 원격리소스가 제거되었어도 다시 생성하지 않는다.

**유형6번 실습진행**

- IAM user를 추가하는 리소스 확인

```go
locals {
  name = "mytest"
}

resource "aws_iam_user" "myiamuser1" {
  name = "${local.name}1"
}

resource "aws_iam_user" "myiamuser2" {
  name = "${local.name}2"
}
```

- 배포진행

```bash
terraform apply -auto-approve
```

- 배포 상태 확인

```bash
aws iam list-users | jq '.Users[] | .UserName'
"admin"
"mytest1"
"mytest2"
```

- tfstate 파일 삭제

```bash
rm -rf terraform.tfstate*
❯ ls terraform.tfstate*
zsh: no matches found: terraform.tfstate*
```

- Plan 명령을 실행하면, 아래와 같이 이미 존재하는 리소스를 파악하지 못하고 새롭게 생성하려고 함.

```bash
$ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_user.myiamuser1 will be created
  + resource "aws_iam_user" "myiamuser1" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "mytest1"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }
```

- apply 명령어를 실행하면, **EntityAlreadyExists** 에러 발생

```bash
$ terraform apply
...
Plan: 2 to add, 0 to change, 0 to destroy.
aws_iam_user.myiamuser2: Creating...
aws_iam_user.myiamuser1: Creating...
╷
│ Error: creating IAM User (mytest1): **EntityAlreadyExists: User with name mytest1 already exists.**
│       status code: 409, request id: e32ae858-e9eb-4c3a-a6ab-d7dba9f8bbd8
│ 
│   with aws_iam_user.myiamuser1,
│   on main.tf line 5, in resource "aws_iam_user" "myiamuser1":
│    5: resource "aws_iam_user" "myiamuser1" {
│ 
╵
```

- 이럴때는 import 명령어를 통해 해결할 수 있다, IAM user의 ID는 유저의 이름이므로 아래의 명령어를 실행한다.
    - `terraform import [options] ADDRESS ID`

```bash
terraform import aws_iam_user.myiamuser1 mytest1                     
aws_iam_user.myiamuser1: Importing from ID "mytest1"...
aws_iam_user.myiamuser1: Import prepared!
  Prepared aws_iam_user for import
aws_iam_user.myiamuser1: Refreshing state... [id=mytest1]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

- tfstate 확인, myiamuser1의 상태파일이 추가되었다.

```bash
{
  "version": 4,
  "terraform_version": "1.5.6",
  "serial": 4,
  "lineage": "0e52f56e-fe1e-d6e7-acb3-f521a3c2f365",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_iam_user",
      "name": "myiamuser1",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
						...
            "id": "mytest1",
            "name": "mytest1",
            
          },
          "sensitive_attributes": [],
...
}
```

## 워크스페이스

State를 관리하는 논리적인 가상 공간을 워크스페이스라고한다.

개발용 환경, 스테이징 환경, 운영환경은 대부분 가지고 있다. 거의 유사한 환경을 구축한다고 하면 여러 개의 프로젝트를 통해 운영할 수 있지만 동일한 환경을 구성한다면 이는 일관성 유지에 좋지 않다. 테라폼에서는 이를 위해 workspace를 지원한다. 하나의 코드를 기반으로 여러 개의 환경을 구성할 수 있으며  `terraform.workspace` 변수를 통해 환경마다 리소스를 조정할 수 있다. 

**vs 아래는 워크스페이스가 아닌 여러 개의 프로젝트로 구성한 모습이다.** 

![](https://velog.velcdn.com/images/han-0315/post/c982c2ac-14dc-4fff-a019-805643728fc2/image.png)


<aside>
💡 설명해주신 워크스페이스의 장단점

- **장점**
    - 하나의 **루트 모듈**에서 다른 환경을 위한 리소스를 동일한 **테라폼 구성**으로 프로비저닝하고 관리
    - 기존 프로비저닝된 환경에 영향을 주지 않고 변경 사항 실험 가능
    - **깃의 브랜치 전략**처럼 동일한 구성에서 서로 다른 리소스 결과 관리 - [참고 : [화해 - Git 브랜치 전략 수립을 위한 전문가의 조언들](https://blog.hwahae.co.kr/all/tech/9507)]
- **단점**
    - State가 동일한 저장소(로컬 또는 백엔드)에 저장되어 State 접근 권한 관리가 불가능(어려움)
    - 모든 환경이 동일한 리소스를 요구하지 않을 수 있으므로 테라폼 구성에 분기 처리가 다수 발생 가능
    - 프로비저닝 대상에 대한 **인증 요소를 완벽히 분리하기 어려움**
        
        → 가장 **큰 단점은 완벽한 격리가 불가능**
        
        ⇒ 해결방안 1. 해결하기 위해 루트 모듈을 별도로 구성하는 디렉터리 기반의 레이아웃을 사용할 수 있다.
        ⇒ 해결방안 2. **Terraform Cloud 환경의 워크스페이스를 활용**
        
</aside>

## Module

모듈은 대부분의 프로그래밍 언어에서 쓰이는 라이브러리나 패키지와 역할이 비슷하다.

중복되거나, 자주쓰는 코드를 모듈화해서 편하게 재사용할 수 있다. 

모듈 디렉터리 형식은 `terraform-<프로바이더 이름>-<모듈 이름>` 형식을 제안한다.

> 이 형식은 Terraform Cloud, Terraform Enterprise에서도 사용되는 방식으로 
1) 디렉터리 또는 레지스트리 이름이 테라폼을 위한 것이고, 2) 어떤 프로바이더의 리소스를 포함하고 있으며, 3) 부여된 이름이 무엇인지 판별할 수 있도록 한다.
> 

## 구조

아래와 같이 루트 모듈에서 자식 모듈을 참조한다. 자식모듈이 라이브러리 역할을 수행하고, 루트 모듈이 main 함수이다. 자식 모듈을 호출할 때, 변수도 맞게 대입한다.

<p align="center">
  <img src="https://velog.velcdn.com/images/han-0315/post/e40ad0fb-7354-493b-ae62-975fe6ef5018/image.png" alt="text" width="3![](https://velog.velcdn.com/images/han-0315/post/b1252151-2465-4ea2-b288-edae797005e6/image.png)
00" />
	출처: https://jloudon.com/cloud/Azure-Policy-as-Code-with-Terraform-Part-1/
</p>

간단한 예시를 확인해보면, 아래의 자식 모듈을 사용한다고 가정해보자. isDB라는 변수의 값을 대입해줘야하고, id와 pw를 출력할 수 있다.

```go
# main.tf
resource "random_pet" "name" {
  keepers = {
    ami_id = timestamp()
  }
}

# DB일 경우 Password 생성 규칙을 다르게 반영 
resource "random_password" "password" {
  length           = var.isDB ? 16 : 10
  special          = var.isDB ? true : false
  override_special = "!#$%*?"
}
# variable.tf
variable "isDB" {
  type        = bool
  default     = false
  description = "패스워드 대상의 DB 여부"
}
# output.tf
output "id" {
  value = random_pet.name.id
}

output "pw" {
  value = nonsensitive(random_password.password.result) 
}
```

**루트 모듈**

"mypw1"의 경우는 변수를 대입하지 않았으니, `isDB`의 기본값이 들어간다. 다음과 같이 모듈을 통해 코드를 구조화하고 재사용할 수 있다.

```go
module "mypw1" {
  source = "../modules/terraform-random-pwgen"
}

module "mypw2" {
  source = "../modules/terraform-random-pwgen"
  isDB   = true
}

output "mypw1" {
  value  = module.mypw1
}

output "mypw2" {
  value  = module.mypw2
}
```

## 프로바이더 정의

루트모듈에서 프로바이더를 정의하는 것이 좋다. 만약, 자식모듈에서 프로바이더를 정의하면 루트 모듈에 버전이 다르면 오류가 발생하고 모듈에 반목문을 쓸 수 없다.

아래의 module “example”과 같이 루트 모듈에 프로바이더를 선언한다.

```go
# The default "aws" configuration is used for AWS resources in the root
# module where no explicit provider instance is selected.
provider "aws" {
  region = "us-west-1"
}

# An alternate configuration is also defined for a different
# region, using the alias "usw2".
provider "aws" {
  alias  = "usw2"
  region = "us-west-2"
}

# An example child module is instantiated with the alternate configuration,
# so any AWS resources it defines will use the us-west-2 region.
module "example" {
  source    = "./example"
  providers = {
    aws = aws.usw2
  }
}
```

모듈은 위에서 진행하듯이 로컬파일로 가능하며 테라폼 레지스트리, 깃허브 등에서 가져와서 쓸 수 있다.

ex) Terraform registry에서 가져오기

```go
module "consul" {
  source = "hashicorp/consul/aws"
  version = "0.1.0"
}
```

## 협업

S3를 통해, 백엔드를 구성하는 것은 1주차때 진행했다. 여기서는 Terraform Cloud를 이용하여 TFC 백엔드를 구성해본다. 당연히 Lock 기능과 버전관리도 지원한다.

## Terraform Cloud

state 관리를 진행하는 TFC는 무상이라고 한다. 

<aside>
💡 TFC

- 제공 기능 : 기본 기능 무료, State 히스토리 관리, State lock 기본 제공, State 변경에 대한 비교 기능
- Free Plan 업데이트 : 사용자 5명 → 리소스 500개, 보안 기능(SSO, Sentinel/OPA로 Policy 사용) - [링크](https://dev.classmethod.jp/articles/update-terraform-cloud-pricing-plan/)
</aside>


**워크스페이스 생성**
- [https://app.terraform.io/](https://app.terraform.io/) 링크 접속 후 계정 생성
- workflow 선택 화면에선 **Create a new organization** 선택
- Connect
- GitHub와 같은 버전관리 시스템에 연결할거면 VCS를 선택한다.
- CLI-driven 선택
    
**terraform login을 진행한다.**

```bash
$ terraform login
[토큰 입력]
```

토큰 확인

```bash
cat ~/.terraform.d/credentials.tfrc.json | jq
{
  "credentials": {
    "app.terraform.io": {
      "token": "YMgr4VM...EWGuw"
    }
  }
}
```

provider.tf에서 테라폼 클라우드를 정의한다.

```bash
terraform {
  cloud {
    organization = "kane-org"         # 생성한 ORG 이름 지정
    hostname     = "app.terraform.io" # default

    workspaces {
      name = "terraform-stduy" # 없으면 생성됨
    }
  }
}
```

이후, terraform init 명령어를 실행하면 `.terraform` 디렉터리가 생성되고 안에 상태파일이 생성된다. 아래는 상태파일 세부내용이다.

```bash
{
    "version": 3,
    "serial": 1,
    ...
    "backend": {
        "type": "cloud",
        "config": {
            "hostname": "app.terraform.io",
            "organization": "kane-org",
            "token": null,
            "workspaces": {
                "name": "terraform-stduy",
                "tags": null
            }
        },

```

이제 init && plan 명령어를 실행하면, 다음과 같이 클라우드에서 동작한다. [terraform cloud local 설정x]

> 설정을 통해, terraform 작업을 모두 로컬에서 돌리고 state 파일만 업로드할 수 있다.
> 

![](https://velog.velcdn.com/images/han-0315/post/072748bd-aa1d-4a96-9dcb-444d032ecc16/image.png)


Plan이 모두 동작하면, 아래와 같이 UI로 배포될 리소스를 알려준다.

![](https://velog.velcdn.com/images/han-0315/post/8baf8844-b60a-4804-bcab-063b4418a147/image.png)


확실히 GUI로 보니 깔끔한 것 같다. 특히 리소스가 많아졌을 때 보기편할 것 같다.


## 도전과제3

> 각자 사용하기 편리한 **리소스를 모듈화** 해보고, 해당 모듈을 활용해서 **반복 리소스**들 배포해보세요!
> 

VPC, Subnet 등 EC2에 필요한 리소스를 모듈화해봤다. 보안그룹의 포트는 SSH, HTTP를 열어놨다.

- module/main.tf
    
    ```go
    locals {
      additional_tags = {
        Name = var.namespace
      }
    }
    
    resource "aws_vpc" "vpc" {
      cidr_block = "192.169.0.0/16"
      tags       = local.additional_tags
    }
    
    data "aws_availability_zones" "available" {
      state = "available"
    }
    
    resource "aws_subnet" "public_subnet" {
      vpc_id                  = aws_vpc.vpc.id
      cidr_block              = "192.169.1.0/24"
      availability_zone       = data.aws_availability_zones.available.names[0]
      map_public_ip_on_launch = true
      tags                    = local.additional_tags
    }
    
    resource "aws_internet_gateway" "igw" {
      vpc_id = aws_vpc.vpc.id
      tags   = local.additional_tags
    }
    resource "aws_route_table" "public_route_table" {
      vpc_id = aws_vpc.vpc.id
      route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.igw.id
      }
      tags = local.additional_tags
    }
    
    resource "aws_route_table_association" "public_rtb_assoc" {
    
      subnet_id      = aws_subnet.public_subnet.id
      route_table_id = aws_route_table.public_route_table.id
    }
    
    resource "aws_security_group" "web_sg" {
      name   = var.namespace
      vpc_id = aws_vpc.vpc.id
    
      ingress {
        from_port   = var.ssh_port
        to_port     = var.ssh_port
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
    
      ingress {
        from_port   = var.http_port
        to_port     = var.http_port
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
    
      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
    
    }
    data "aws_ami" "default" {
      most_recent = true
      owners      = ["amazon"]
    
      filter {
        name   = "owner-alias"
        values = ["amazon"]
      }
    
      filter {
        name   = "name"
        values = ["amzn2-ami-hvm*"]
      }
    }
    
    resource "aws_instance" "app" {
      ami                    = data.aws_ami.default.id
      instance_type          = var.ec2_instance_type
      key_name               = var.key_name
      vpc_security_group_ids = [aws_security_group.web_sg.id]
      subnet_id              = aws_subnet.public_subnet.id
      tags                   = local.additional_tags
    }
    ```
    
- module/output.tf
    
    ```go
    output "instance_public_ip" {
      value       = aws_instance.app.public_ip
      description = "The public IP address of the App instance"
    }
    ```
    
- module/variable.tf
    
    ```go
    variable "ssh_port" {
      default     = 22
      type        = number
      description = "SSH port"
    }
    variable "http_port" {
      default     = 80
      type        = number
      description = "HTTP port"
    }
    variable "ec2_instance_type" {
      type        = string
      description = "The type of EC2 instance to launch"
    }
    variable "key_name" {
      type        = string
      description = "The key name to use for an EC2 instance"
    }
    variable "namespace" {
      type        = string
      description = "env namespace"
    }
    ```
    

- root/main.tf
    
    ```go
    locals {
      env = {
        dev = {
          instance_type = "t3.micro"
          key_name      = "m1"
          namespace     = "dev"
        }
        prod = {
          instance_type = "t3.medium"
          key_name      = "m1"
          namespace     = "prod"
        }
      }
    }
    
    provider "aws" {
      region = "ap-northeast-2"
    }
    
    module "ec2_aws_amazone" {
      for_each          = local.env
      source            = "../../module/ec2"
      key_name          = each.value.key_name
      ec2_instance_type = each.value.instance_type
      namespace         = each.value.namespace
    }
    
    # output.tf
    output "module_output_instance_public_ip" {
      value = [
        for k in module.ec2_aws_amazone : k.instance_public_ip
      ]
    }
    ```
    

이제, 테라폼 명령어를 실행하여 리소스를 배포한다.

```bash
$ tf apply -auto-approve
module.ec2_aws_amazone["dev"].data.aws_availability_zones.available: Reading...
module.ec2_aws_amazone["prod"].data.aws_availability_zones.available: Reading...
module.ec2_aws_amazone["prod"].data.aws_ami.default: Reading...
module.ec2_aws_amazone["dev"].data.aws_ami.default: Reading...
module.ec2_aws_amazone["dev"].data.aws_availability_zones.available: Read complete after 1s [id=ap-northeast-2]
module.ec2_aws_amazone["prod"].data.aws_availability_zones.available: Read complete after 1s [id=ap-northeast-2]
module.ec2_aws_amazone["dev"].data.aws_ami.default: Read complete after 1s [id=ami-0ec77cfb1037681eb]
module.ec2_aws_amazone["prod"].data.aws_ami.default: Read complete after 1s [id=ami-0ec77cfb1037681eb]
Apply complete! Resources: 14 added, 0 changed, 0 destroyed.

Outputs:

module_output_instance_public_ip = [
  "3.35.235.54",
  "13.124.205.208",
]
```

**AWS 콘솔에서 배포된 목록을 확인한다.**

- vpc
    ![](https://velog.velcdn.com/images/han-0315/post/e156e759-8f17-4602-acdf-51fb7c2402f6/image.png)

    
- EC2
    ![](https://velog.velcdn.com/images/han-0315/post/f601c557-299a-495c-8733-3037a7b08abc/image.png)

    

환경 별로, 배포가 잘 된 모습을 확인할 수 있다.