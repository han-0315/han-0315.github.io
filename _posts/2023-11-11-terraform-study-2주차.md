---
layout: post
title: 'Terraform Study #2'
date: 2023-11-11 15:17 +0900
description: 'Terraform Study 2주차'
category: [Study, Terraform]
tags: [Terraform, IaC, Study]
image:
  path: /assets/img/logo/Terraform/logo.png
  alt: Terraform Logo
pin: false
math: true
mermaid: true
---
‘테라폼으로 시작하는 IaC’ 책으로 진행하는 Terraform 스터디[T101] 2주차 정리내용입니다.
<!--more-->
# 2주차
## 데이터 소스

데이터 소스(`data`)는 외부의 리소스 혹은 저장된 정보를 내부로 가져올 때 사용한다.

기본 사용법은 2기 스터디원 Ssoon님이 [블로그](https://kschoi728.tistory.com/124)에 잘 정리해주셨다. 

아래와 같이 AMI나 AZ를 조회할 때 유용하다.

- ubuntu AMI 조회

```go
data "aws_ami" "ubuntu" {
  most_recent = true

  owners = ["099720109477"] 

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

- AZ 검색

```go
data "aws_availability_zones" "available" {
    group_names = [
        "ap-northeast-2",
    ]
    id          = "ap-northeast-2"
    names       = [
        "ap-northeast-2a",
        "ap-northeast-2b",
        "ap-northeast-2c",
        "ap-northeast-2d",
    ]
    state       = "available"
    zone_ids    = [
        "apne2-az1",
        "apne2-az2",
        "apne2-az3",
        "apne2-az4",
    ]
}
```

## 입력 변수

변수는 Terrraform 코드를 동적으로 구성할 수 있게 한다. 테라폼에서는 이것을 **입력 변수** Input Variables 로 정의한다.

- 선언 예시
    
    ```go
    variable "<이름>" {
     <인수> = <값>
    }
    
    variable "image_id" {
     type = string
    }
    ```
    

위와 같이 변수를 정의할 때 다양한 메타인수를 넣을 수 있다. 관련 정보는 아래와 같다.

<aside>
💡 변수 정의 시 사용 가능한 메타인수

- **default** : 변수 값을 전달하는 여러 가지 방법을 지정하지 않으면 기본값이 전달됨, 기본값이 없으면 대화식으로 사용자에게 변수에 대한 정보를 물어봄
- **type** : 변수에 허용되는 값 유형 정의, string number bool list map set object tuple 와 유형을 지정하지 않으면 any 유형으로 간주
- **description** : 입력 변수의 설명
- **validation** : 변수 선언의 제약조건을 추가해 유효성 검사 규칙을 정의 - [링크](https://honglab.tistory.com/217)
- **sensitive** : 민감한 변수 값임을 알리고 테라폼의 출력문에서 값 노출을 제한 (암호 등 민감 데이터의 경우) - [링크](https://daaa0555.tistory.com/371)
- **nullable** : 변수에 값이 없어도 됨을 지정
</aside>

- 우선순위

1번 부터 변수를 대입하며, 후 순위가 전 순위를 덮어쓰기 합니다. 결론적으로 아래에 있는 옵션이 우선순위가 높습니다. 

| Order | Option                                 |
| ----- | -------------------------------------- |
| 1     | Environment Variables                  |
| 2     | terraform.tfvars                       |
| 3     | terraform.tfvars.json                  |
| 4     | *.auto.tfvars (alphabetical order)     |
| 5     | -var or –var-file (command-line flags) |

## Local

local은 외부에서 입력되지 않고, 코드 내에서만 가공되어 동작하는 값이다. 외부에서 입력되진 않지만 Local 선언 자체에 일반 변수를 넣을 수 있다. (아래의 예시 참고)

local은 회사내의 클라우드 서비스를 이용할 때, 리소스에 태그를 걸어야한다. ex) Owner, Purpose 등

이 때 Local 변수를 사용하면 아래와 같이 편하게 리소스에 태그를 걸 수 있다.

```go
locals {
  additional_tags = {
    Purpose     = var.purpose
    Owner       = var.owner
  }
}
...
resource "aws_instance" "app" {
...
  tags = merge(
    {
      Name = "web-app"
    },
    local.additional_tags
  )

}
```

## 실습

`도전과제2` : 위 3개 코드 파일 내용에 **리소스**의 이름(**myvpc**, **mysubnet1** 등)을 반드시! 꼭! 자신의 **닉네임**으로 변경해서 배포 실습해보세요!

- VPC DNS 옵션 활성화

```go
resource "aws_vpc" "myvpc" {
  cidr_block           = "10.10.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "t101-study"
  }
}
```

![](https://velog.velcdn.com/images/han-0315/post/9687844c-e387-4839-81ad-bc66cbf4fb1c/image.png)


- `[도전과제1]` 리전 내에서 사용 가능한 **가용영역 목록 가져오기**를 사용한 VPC 리소스 생성 실습 진행
- 아래와 같이, data 소스를 이용하여 AZ를 가져온다.

```go
resource "aws_subnet" "mysubnet1" {
  vpc_id     = aws_vpc.myvpc.id
  cidr_block = "10.10.1.0/24"

  availability_zone = data.aws_availability_zones.available.names[2]

  tags = {
    Name = "t101-subnet1"
  }
}

resource "aws_subnet" "mysubnet2" {
  vpc_id     = aws_vpc.myvpc.id
  cidr_block = "10.10.2.0/24"

  availability_zone = "ap-northeast-2c"

  tags = {
    Name = "t101-subnet2"
  }
}
```
![](https://velog.velcdn.com/images/han-0315/post/c7c5bd11-e370-48df-af0a-f6869fdcdfd8/image.png)



- ec2 생성 콘솔에서 확인

![](https://velog.velcdn.com/images/han-0315/post/7875e838-0973-4bcf-a3c4-fef6620a761b/image.png)



- Graph

Vscode 에서 추출한 그림인데, 리소스가 많아 보기 조금 불편하다.

![](https://velog.velcdn.com/images/han-0315/post/f1fdc61e-ee24-4db2-adfb-a0e47e06dcc3/image.png)



- EC2 접속하기

```bash
$ MYIP=$(terraform output -raw kane_ec2_public_ip)
$ echo $MYIP                   
3.35.173.67
$ while true; do curl --connect-timeout 1  http://$MYIP/ ; echo "------------------------------"; date; sleep 1; done
<h1>RegionAz(apne2-az1) : Instance ID(i-0ca40805a20604dbe) : Private IP(10.10.1.34) : Web Server</h1>
------------------------------
Mon Sep  4 00:50:56 KST 2023
<h1>RegionAz(apne2-az1) : Instance ID(i-0ca40805a20604dbe) : Private IP(10.10.1.34) : Web Server</h1>
------------------------------
Mon Sep  4 00:50:57 KST 2023
<h1>RegionAz(apne2-az1) : Instance ID(i-0ca40805a20604dbe) : Private IP(10.10.1.34) : Web Server</h1>
------------------------------
Mon Sep  4 00:50:58 KST 2023
```

## Output

terraform apply 이후 파일에 적힌 출력값을 콘솔에 출력해준다. 주로 Ec2의 퍼블릭 ip같이 꼭 확인해야 하는 것들을 주로 출력한다. 생성 후의 정보를 출력하기에 당연한 이야기지만 **오로지, apply를 적용할 때만 출력한다. 또한 이런 값들은 추후 파이프라인 구성, shell script 혹은 `ansible` 에 사용할 수도 있다.**

**기본 예시**

```go
output "instance_ip_addr" {
  value       = aws_instance.server.private_ip
  description = "The private IP address of the main server instance."
}
```

**조건 검사 진행**

```go
output "api_base_url" {
  value = "https://${aws_instance.example.private_dns}:8433/"

  # The EC2 instance must have an encrypted root volume.
  precondition {
    condition     = data.aws_ebs_volume.example.encrypted
    error_message = "The server's root volume is not encrypted."
  }
}
```

- Option
    - `sensitive` : CLI 에서 출력되지 않게 할 수 있다.
    - ****`depends_on` : 선수관계를 정할 수 있다.(먼저, 출력되는 것을 결정할 수 있다.)**

```go
output "instance_ip_addr" {
  value       = aws_instance.server.private_ip
  description = "The private IP address of the main server instance."

  depends_on = [
    # Security group rule must be created before this IP address could
    # actually be used, otherwise the services will be unreachable.
    aws_security_group_rule.local_access,
  ]
}
```

## 반복문

- **count :** 반복문, 정수 값만큼 리소스나 모듈을 생성함. 인스턴스가 거의 동일한 경우 Count가 적절(For each 보다), `count, count.index` 로 접근
    
    ```go
    variable "subnet_ids" {
      type = list(string)
    }
    
    resource "aws_instance" "server" {
      # Create one instance for each subnet
      count = length(var.subnet_ids)
    	...
      subnet_id     = var.subnet_ids[count.index]
    
      tags = {
        Name = "Server ${count.index}"
      }
    }
    ```
    
- **for_each :** 반복문, 선언된 key 값 개수만큼 리소스를 생성
    
    ```go
    resource "aws_instance" "example" {
      # One VPC for each element of var.vpcs
      for_each = var.instances
    
      # each.value here is a value from var.vpcs
      name = each.key
    	ami = each.value.ami
    }
    ```
    
- **for**
    
    만약 [ ]으로 되어있으면 tuple 형식으로 컨테이너를 반환하고, {}이면 오브젝트로 반환하는 반복문이다.
    
    또한 `for` 뒤에 `If` 를 통해 필터링 기능도 가능하다.(if 인 값만 사용)
    
    ```go
    [for s in var.list : upper(s) if s != ""]
    [for i, v in var.list : "${i} is ${v}"]
    # object 형식일때
    [for k, v in var.map : length(k) + length(v)]
    ```
    

- **Dynamic Block**

특수한 목적의 Dynamic Block을 통해 동적으로 만들어지는 변수에 대해 반복 가능한 블럭을 만들 수 있다. 기존의 for_each, count 등 반복문은 리소스 block 등 자신의 바깥 블럭을 반복해서 찍어내는 것에 비해 dynamic block은 block자체를 정의하며 반복적으로 찍어낸다. (resource와 같은 단일블락이 아닌 내부 블락으로만 사용된다.) 사용방법은 Argument를 확인하면 된다.

- 찾아보니, 다음과 같은 안내사항도 있었다.
    - 과도한 사용을 피한다. (동적 블록을 과도하게 사용하면 구성을 읽고 유지하기 어려울 수 있다.)
    - 재사용 가능한 모듈을 위한 깨끗한 사용자 인터페이스를 구축하기 위해 세부 정보를 숨겨야 할 때 사용합니다
    - 가능한 경우 항상 중첩된 블록을 문자 그대로 써라.

```go
resource "aws_security_group" "backend-sg" {
  name        = "backend-sg"
  vpc_id      = aws_vpc.backend-vpc.id
	dynamic "ingress" {
		for_each = var.ingress_ports
		content {
	      from_port = ingress.value
				to_port = ingress.value
				protocol = "tcp"
				cidr_blocks = ["0.0.0.0/0"]
		}
	}
}
# 아래와 같이 하기 싫어서 위처럼 진행
resource "aws_security_group" "backend-sg" {
  name        = "backend-sg"
  vpc_id      = aws_vpc.backend-vpc.id
	ingress {
	      from_port = 22
				to_port = 22
				protocol = "tcp"
				cidr_blocks = ["0.0.0.0/0"]
	}
	ingress {
	      from_port = 8080
				to_port = 8080
				protocol = "tcp"
				cidr_blocks = ["0.0.0.0/0"]
	}
}
		
		
```

## 도전과제3

> `도전과제3` : **입력변수**를 활용해서 리소스(어떤 리소스든지 상관없음)를 배포해보고, 해당 코드를 정리해주세요!
> 

위에서 진행한 EC2 배포 코드를 이용한다. 변수를 통해 인스턴스의 타입을 동적으로 구성한다.

- EC2 구성

```go
resource "aws_instance" "kane_ec2" {

  depends_on = [
    aws_internet_gateway.kane_igw
  ]

  ami                         = data.aws_ami.amazonlinux2.id
  associate_public_ip_address = true
	// 아래의 내용을 수정!
  instance_type               = var.ec2_instance_type
  vpc_security_group_ids      = ["${aws_security_group.kane_sg.id}"]
  subnet_id                   = aws_subnet.kane_subnet1.id
...
```

- variable.tf 파일을 생성한 뒤, 아래의 내용 추가

```go
variable "ec2_instance_type" {
  type        = string
  description = "The type of EC2 instance to launch"
}
```

- terraform.tfvars 파일을 생성한 뒤 아래의 내용을 추가한다.
    - 해당 파일이 존재하면, 테라폼은 자동으로 변수의 값을 가져간다. 우선순위에 따라 덮어써질 수 있긴 하다. 하지만 여기선 변수 입력을 해당 파일로만 하니 상관없다.

```go
ec2_instance_type = "t2.small"
```

이제 `Terraform apply` 명령어를 통해 인프라를 구축한다.

기존과는 다르게 `t2.micro` 가 아닌 `t2.small` 이 생성된 것을 확인할 수 있다. 

![](https://velog.velcdn.com/images/han-0315/post/81a554a6-f603-49ca-b473-95b46c532e15/image.png)

## 도전과제4

> `도전과제4` : **local**를 활용해서 리소스(어떤 리소스든지 상관없음)를 배포해보고, 해당 코드를 정리해주세요!
> 

local을 통해, EC2에 태깅 작업을 진행한다.

- local 선언

```go
locals {
  additional_tags = {
    Environment = "Dev"
    Purpose     = "Test"
    Owner       = "Kane"
  }
}
```

- EC2에 추가

```go
resource "aws_instance" "kane_ec2" {

  depends_on = [
    aws_internet_gateway.kane_igw
  ]

	...

  tags = merge({
    Name = "t101-kane_ec2"
    }
  , local.additional_tags)
}
```

`terraform apply`를 실행한다.

이제 AWS 콘솔에 들어가 **EC2 > Tags** 페이지를 확인하면 다음과 같이 태깅이 올바르게 된 것을 확인할 수 있다.


![](https://velog.velcdn.com/images/han-0315/post/c4ee119f-ed52-4d97-ba77-70bdd10e8bb4/image.png)

