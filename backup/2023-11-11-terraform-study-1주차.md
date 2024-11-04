---
layout: post
title: 'Terraform Study #1'
date: 2023-11-11 15:17 +0900
description: 'Terraform Study 1주차'
category: [Study, Terraform]
tags: [Terraform, IaC, Study]
image:
  path: /assets/img/logo/Terraform/logo.png
  alt: Terraform Logo
pin: false
math: true
mermaid: true
---
‘테라폼으로 시작하는 IaC’ 책으로 진행하는 Terraform 스터디[T101] 1주차 정리내용입니다.
<!--more-->
# 1주차

평소에 테라폼에 관심이 많아, 자격증도 취득하고 공부를 했다. 공부를 했지만, 아직 이것저것 헷갈리는 게 많다. 이를 구체화시키고, 실무에 대한 조언을 들을 겸 스터디에 참가하게 되었다. 스터디는 [CloudNet](https://www.notion.so/CloudNet-Blog-c9dfa44a27ff431dafdd2edacc8a1863?pvs=21)에서 주관하고 유형욱님과 윤서율님이 진행해주신다. 

1주차에서는 테라폼에 대해 알아보고, 실행 환경을 세팅한다. 이후 EC2를 배포해보면서 기본 문법과 명령어에 대해 학습한다. 

## 테라폼 제공유형

1. **On-premise** : Terraform이라 불리는 형태로, 사용자의 컴퓨팅 환경에 오픈소스 바이너리툴인 테라폼을 통해 사용
    
    > [라이선스를 변경] **오픈소스 → 커뮤니티 에디션**으로 변경된다.
    > 
2. **Hosted SaaS** : Terraform Cloud로 불리는 **SaaS**로 제공되는 구성 환경으로 하시코프가 관리하는 서버 환경이 제공
3. **Private Install** : Terraform Enterprise로 불리는 서버 설치형 구성 환경으로, 기업의 사내 정책에 따라 프로비저닝 관리가 외부 네트워크와 격리 - [링크](https://www.hashicorp.com/blog/announcing-hashicorp-private-terraform-enterprise)

2,3 번은 기본적으로 GUI가 제공되며 Terraform Cloud는 Free 티어가 있다. 

**테라폼 클라우드 가격정책 비교**

- Free : 리소스 500개 까지 무료 → 커뮤니티 버전
- Standard : Free + 워크플로우 기능 추가 + 동시실행(Concurrency 개수 3개)

## AWS 옵션(실습 환경 구성)

**AWS_PAGER 옵션 제거**

- 하나의 페이지처럼, 작동함 → 나갈려면 :q 옵션을 입력해야 하고, 기타 옵션도 쓸 수 있는다.

페이저를 비활성화하는 이유는 아래와 같다.

1. **Scripting and Automation**: AWS CLI 명령어의 출력을 스크립트나 다른 프로그램에서 파싱해야 할 경우, 페이저가 불필요한 중간 단계를 추가할 수 있습니다.
2. **Non-Interactive Environments**: CI/CD 파이프라인이나 배치 작업과 같은 비대화형(non-interactive) 환경에서는 페이저가 문제를 일으킬 수 있습니다.

`export AWS_PAGER=""`

적용하면, PAGER가 비활성화된다.

**AWS 계정 선택**

여러 AWS 계정을 쓰는 경우, Profile을 환경변수로 선택할 수 있다. 

`export AWS_PROFILE="study"`

Terraform에서도 별도의 provider block에서 세팅해줘야한다.

```bash
provider "aws" {
  profile = "eks"
  region  = "ap-northeast-2"
}
```

추가) **vscode aws toolkit 설정으로, 현재의 profile을 확인할 수 있다. 여러 AWS 계정을 쓸 경우, 편리하다.**

## HCL

HCL은 JSON을 본따 만든 언어이며, JSON보다 사람 친화적인 언어이다. 

- 인프라가 코드로 표현되고, 이 코드는 곧 인프라이기 때문에 **선언적(declarative)** 특성을 갖게 되고 튜링 완전한 Turing-complete **언어적** 특성을 갖는다. [참고: [튜링완전](http://wiki.hash.kr/index.php/%ED%8A%9C%EB%A7%81%EC%99%84%EC%A0%84)]
- 즉, 일반적인 프로그래밍 언어의 **조건문** 처리 같은 동작이 가능하다. 자동화와 더불어, 쉽게 **버저닝해 히스토리**를 관리하고 **함께 작업** 할 수 있는 기반을 제공. → 확실한 차이점

## Terraform Command 옵션

- `validate`
    - **`-no-color`** : 대부분의 명령과 함께 사용 가능, 로컬이 아닌 외부 실행 환경(젠킨스, Terraform Cloud, Github Action 등)을 사용하는 경우, 색상 표기 문자가 표기 될 수 있다. 이 경우 -no-color 옵션으로 **색상 표기 문자 없이 출력**함. [[참고](https://developer.hashicorp.com/terraform/cli/commands/plan#no-color)]
- `plan`
    - `-detailed-exitcode` : **plan 추가 옵션**으로, 파이프라인 설계에서 활용 가능, exitcode가 환경 변수로 구성됨
- `apply` or `destroy`
    - `-auto-approve`: 자동 승인 기능 부여 옵션

## EC2 배포

우선, 해당 실습에서는 default VPC를 사용한다. 만약 사용하는 리전의 default VPC가 없다면 아래의 명령어를 실행하여 생성한다. 

```bash
$ aws ec2 create-default-vpc
{
    "Vpc": {
        "CidrBlock": "172.31.0.0/16",
        ...
        ],
        "IsDefault": true,
        "Tags": []
    }
}
```

EC2를 프로비저닝하는 기본적인 코드이다. 해당 코드를 실행하면, EC2를 생성할 수 있으며 접속할 public IP를 받을 수 있다.

```go
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_instance" "example" {
  ami                    = "ami-0c9c942bd7bf113a2"
  instance_type          = "t2.micro"
  **vpc_security_group_ids = [aws_security_group.instance.id]**

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, T101 Study" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  tags = {
    Name = "Single-WebSrv"
  }
}

resource "aws_security_group" "instance" {
  name = var.security_group_name

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

variable "security_group_name" {
  description = "The name of the security group"
  type        = string
  default     = "terraform-example-instance"
}

**output** "public_ip" {
  value       = aws_instance.example.public_ip
  description = "The public IP of the Instance"
}
```

```bash
$ tf apply
...
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

public_ip = "54.180.106.217"
```

<aside>
💡 만약 "빈번하게 서버의 포트가 변경되면" 어떻게 해야 될까? 이런 "불편함을 해결"하려면 어떻게 해야 될까?
</aside>

→ `user_data_replace_on_change = false` 옵션을 추가한다!

코드를 아래와 같이 변경한다.

```go
resource "aws_instance" "example" {
  ami                    = "ami-0c9c942bd7bf113a2"
	...

  user_data_replace_on_change = false
  tags = {
    Name = "Single-WebSrv"
  }
}
...
resource "aws_security_group" "instance" {
  name = var.security_group_name
	...
	# lifecycle을 추가하면 다운타임을 줄일 수 있다.
	lifecycle {
    create_before_destroy = true
  }
}
```

변경하기 이전과 이후에 생성과정을 보면 다른 것을 알 수 있다. 이전처럼 파괴하고 재생성이 아닌, 변경된 값을 업데이트한다.

```bash
# 이전
aws_security_group.instance: Destroying... [id=sg-03c90b3d559abb123]
aws_security_group.instance: Destruction complete after 1s
aws_security_group.instance: Creating...
aws_security_group.instance: Creation complete after 1s [id=sg-0de633908986b76ad]
# 라이프사이클 적용 후
...
aws_security_group.instance: Modifying... [id=sg-0de633908986b76ad]
aws_security_group.instance: Modifications complete after 1s [id=sg-0de633908986b76ad]
```

즉, "user_data_replace_on_change = false" 옵션을 통해 인스턴스를 새로 생성하지 않고, 기존 인스턴스를 유지하면서 변경된 값을 업데이트할 수 있다.
이 설정은 **`user_data`**가 변경될 때 EC2 인스턴스를 새로 생성할 것인지, 아니면 기존 인스턴스를 유지할 것인지를 결정한다. `false`는 인스턴스를 유지하는 옵션이며, 인스턴스를 유지한다.

```bash
# 적용 전
aws_instance.example: Destroying... [id=i-0794bd9bb343a948b]
...
aws_instance.example: Creating...

# "user_data_replace_on_change = false" 적용 후
aws_instance.example: Modifying... [id=i-0568341d533d4ca58]
aws_instance.example: Still modifying... [id=i-0568341d533d4ca58, 10s elapsed]
aws_instance.example: Still modifying... [id=i-0568341d533d4ca58, 20s elapsed]
```

## 테라폼 문법 설명

Terraform 블록, 아래와 같이 내용을 잘 정리해주셨다.

- ***오늘 실행하던, 3년 후에 실행하던 동일한 결과를 얻을 수 있어야 한다! (Desired State + Immutable)***

```go
terraform {
  required_version = "~> 1.3.0" # 테라폼 버전

  required_providers { # 프로바이더 버전을 나열
    random = {
      version = ">= 3.0.0, < 3.1.0"
    }
    aws = {
      version = "4.2.0"
    }
  }

  cloud { # Cloud/Enterprise 같은 원격 실행을 위한 정보 [참고: Docs]
    organization = "<MY_ORG_NAME>"
    workspaces {
      name = "my-first-workspace"
    }
  }

  backend "local" { # state를 보관하는 위치를 지정 [참고: Docs, local, remote, s3]
    path = "relative/path/to/terraform.tfstate"
  }
}
```

- 테라폼 0.13 버전 이전에는 provider 블록에 함께 버전을 명시했지만 해당 버전 이후 프로바이더 버전은 terraform 블록에서 `required_providers`에 정의

```go
terraform {
  cloud {
    hostname = "[app.terraform.io](http://app.terraform.io/)"
    organization = "my-org"
    workspades = {
       name = "my-app-prod"
    }
  }
}
```

## Backend

협업을 위해서는 s3 등 원격으로 저장해서 관리함, 기본적으로 lock을 지원한다. 로컬에서 간단하게 테스트할 수 있는 데, 로컬에서 `apply` 명령어를 실행하고, 승인을 기다릴 때  ls -al 명령어로 작업 디렉터리의 파일을 확인하면 `.terraform.tfstate.lock.info` 파일이 생성된 것을 확인할 수 있다. 

```bash
cat .terraform.tfstate.lock.info | jq .
{
  "ID": "b4dbfee6-a28f-04da-d235-5591414dbcbc",
  "Operation": "OperationTypeApply",
  "Info": "",
  "Who": "kane@kanes-MacBook-Pro.local",
  "Version": "1.5.6",
  "Created": "2023-08-27T14:07:14.110318Z",
  "Path": "terraform.tfstate"
}
```

- 추가 옵션1 (**이전 구성 유지**) : **`-migrate-state`**는 terraform.tfstate의 이전 구성에서 최신의 state 스냅샷을 읽고 기록된 정보를 새 구성으로 전환한다.
- 추가 옵션2 (**새로 초기화**) : `-reconfigure`는 init을 실행하기 전에 terraform.tfstate 파일을 삭제해 테라폼을 처음 사용할 때처럼 이 작업 공간(디렉터리)을 초기화 하는 동작이다.

## .tfstate

Terraform의 **`.tfstate`** 파일 내의 **`serial`** 값은 상태 파일의 버전을 나타내며, 동시성 제어와 데이터 무결성 확인에도 중요한 역할을 합니다. 이 값은 Terraform 명령이 실행될 때마다 자동으로 증가하여, 상태 파일의 최신성과 일관성을 유지합니다. 그렇기에 backup 파일이 현재 state 파일보다 serial 번호가 낮다.

## 도전과제1 (**EC2 웹 서버 배포*)***

위의 EC2 실습에서 user_data 부분만 변경했다. 

```go
user_data = <<-EOF
              #!/bin/bash
              echo "T101 Study Kane" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF
```

![](https://velog.velcdn.com/images/han-0315/post/c5f02ee5-8f33-4217-ae07-5fa3d7b7fbc1/image.png)


## 도전과제2 (Backend)

**AWS S3/DynamoDB 백엔드** 
Backend는 Terraform의 상태파일을 원격저장소에 저장하는 것이다. 이를 통해 팀 단위의 협업이 가능하다. 만약, 로컬에 상태파일을 저장하면 팀원들이 변경할 때마다 상태파일을 주고 받아야한다. 그렇지 않으면, 상태파일과 실제 인프라의 상태가 달라 문제가 발생한다.


> Backend는 다른 편에서 더 자세하게 다룬다.
> 

아래는 관련 코드이다.

```go
provider "aws" {
  profile = "eks"
  region  = "ap-northeast-2"
}

resource "aws_s3_bucket" "mys3bucket" {
  bucket = "kane-t101study-tfstate"
}

# Enable versioning so you can see the full revision history of your state files
resource "aws_s3_bucket_versioning" "mys3bucket_versioning" {
  bucket = aws_s3_bucket.mys3bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "mydynamodbtable" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

output "s3_bucket_arn" {
  value       = aws_s3_bucket.mys3bucket.arn
  description = "The ARN of the S3 bucket"
}

output "dynamodb_table_name" {
  value       = aws_dynamodb_table.mydynamodbtable.name
  description = "The name of the DynamoDB table"
}

```

EC2를 배포하는 코드에 아래와 같은 원격 backend를 설정한다.

```go
terraform {
  backend "s3" {
    bucket = "kane-t101study-tfstate"
    key    = "dev/terraform.tfstate"
    region = "ap-northeast-2"
    dynamodb_table = "terraform-locks"
    # encrypt        = true
  }
}
```

배포한 후, AWS 콘솔에 접속하면 아래와 같이 table을 확인할 수 있다.

![](https://velog.velcdn.com/images/han-0315/post/21b1bc61-2539-4c5e-9a64-02486d9d4355/image.png)




이제, s3를 모니터링하며 Terraform을 배포하여 state 파일이 정상적으로 변경되는 지 확인한다.

```bash
while true; do aws s3 ls s3://$NICKNAME-t101study-tfstate --recursive --human-readable --summarize ; echo "------------------------------"; date; sleep 1; done

Total Objects: 0
   Total Size: 0 Bytes
------------------------------
Mon Aug 28 00:50:18 KST 2023

Total Objects: 0
   Total Size: 0 Bytes
------------------------------
...
# 리소스 생성
------------------------------
Mon Aug 28 00:53:16 KST 2023
2023-08-28 00:50:56   21.1 KiB dev/terraform.tfstate

Total Objects: 1
   Total Size: 21.1 KiB
------------------------------
Mon Aug 28 00:53:17 KST 2023
2023-08-28 00:53:18   22.4 KiB dev/terraform.tfstate
------------------------------
...
# 리소스 업데이트
------------------------------
Total Objects: 1
   Total Size: 22.4 KiB
------------------------------
# 리소스 삭제
...
------------------------------
Mon Aug 28 00:56:04 KST 2023
2023-08-28 00:56:03  180 Bytes dev/terraform.tfstate

Total Objects: 1
   Total Size: 180 Bytes
------------------------------
Mon Aug 28 00:56:05 KST 2023
```

아래와 같이 콘솔에서도 확인할 수 있다. 생성 순서는 아래에서부터 위 방향이다.

![](https://velog.velcdn.com/images/han-0315/post/3c80559c-3c40-4be1-8600-ab45e84f5281/image.png)
