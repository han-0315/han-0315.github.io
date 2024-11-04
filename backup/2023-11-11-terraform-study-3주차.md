---
layout: post
title: 'Terraform Study #3'
date: 2023-11-11 15:17 +0900
description: 'Terraform Study 3주차'
category: [Study, Terraform]
tags: [Terraform, IaC, Study]
image:
  path: /assets/img/logo/Terraform/logo.png
  alt: Terraform Logo
pin: false
math: true
mermaid: true
---
‘테라폼으로 시작하는 IaC’ 책으로 진행하는 Terraform 스터디[T101] 3주차 정리내용입니다.
<!--more-->
# 3주차
이번시간은 테라폼 기본사용 마지막 단계(3/3)이다. 이번주차에서는 조건문, 함수, 프로비저너, data block에 대해 배운 뒤 프로바이더를 경험해보고 마무리된다. 개인적으로 null_resource에 대해 잘몰랐다. 많이 사용한 플러그인 중 하나라고 한다. 이번 기회에 잘 배워두면 좋을 것 같다. (지금은 terraform_data가 같은 기능을 수행한다.)

## Conditional(조건문)

조건 문의 경우 C언어의 삼항연산자와 유사하다. 그 외에는 지원하지 않는 모양이다.

형식:  `condition ? true_val : false_val` 

**실습**

- main.tf

```go
variable "enable_file" {
  default = true
}

resource "local_file" "foo" {
  count    = var.enable_file ? 1 : 0
  content  = "foo!"
  filename = "${path.module}/foo.bar"
}

output "content" {
  value = var.enable_file ? local_file.foo[0].content : ""
}
```

위의 코드의 내용은 var.enable_file의 값을 입력하지 않거나, true로 설정하면 foo.bar라는 local file을 생성한다. 반대의 경우라면 리소스를 생성하지 않는다.

- `false`를 지정한 경우

```bash
$ export TF_VAR_enable_file=false
$ export | grep TF_VAR_enable_file
TF_VAR_enable_file=false
$ terraform init && terraform plan && terraform apply -auto-approve
...
Changes to Outputs:
  + content = ""

You can apply this plan to save these new output values to the Terraform state, without changing any real
infrastructure.

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

content = ""
```

- `true`를 지정한경우(=아무것도 입력하지 않음)

```bash
# 아무것도 입력하지 않았을 떄
$ terraform init && terraform plan && terraform apply -auto-approve
...
+ content = "foo!"
local_file.foo[0]: Creating...
local_file.foo[0]: Creation complete after 0s [id=4bf3e335199107182c6f7638efaad377acc7f452]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:
content = "foo!"

$ terraform state list
local_file.foo[0]

$ echo "local_file.foo[0].content" | terraform console
╷
│ Warning: Value for undeclared variable
│ 
│ The root module does not declare a variable named "ec2_instance_type" but a value was found in file
│ "terraform.tfvars". If you meant to use this value, add a "variable" block to the configuration.
│ 
│ To silence these warnings, use TF_VAR_... environment variables to provide certain "global" settings to
│ all configurations in your organization. To reduce the verbosity of these warnings, use the
│ -compact-warnings option.
╵
"foo!"

```

위의 예시처럼, 조건문이 아주 잘 적용된다.

## 함수

함수는 내장함수만 사용가능하다. 사용자 정의함수와 같이 직접 만들 수 없다. [공식문서](https://developer.hashicorp.com/terraform/language/functions)에서 확인하면서 확인한 함수를 간단하게 정리해봤다.

- `toset` : 해당 함수는 집합과 같이 중복된 원소를 제거하고, 정렬시킨다.
    - `toset(["b", "a","b"]) =[”a”, “b”]`
- `Slice` :  목록 내에서 일부 연속 요소(elements)를 추출합니다. 시작 인덱스(`startindex`
)는 포함되지만 끝 인덱스(`endindex`)는 제외
- `[**length](https://developer.hashicorp.com/terraform/language/functions/length)` :** list, map 또는 string의 길이를 계산합니다. 리스트 또는 맵이면 컬렉션(collection)의 요소 수, 문자열이면 문자 수를 반환합니다.

- 숫자 관련 함수
    - `min, max, ceil, floor` 함수도 존재한다.
    - min(-1,2,var.temp) = -1, ceil(10.1) = 11
- 문자열 관련 함수
    - split(",", "ami-xyz,AMI-ABC,ami-efg") = `[ "ami-xyz","AMI-ABC","ami-efg" ]`
    - lower, upper : lower(var.ami)= `[ "ami-xyz","ami-abc","ami-efg" ]`
    - substr(var.ami.0,7) = ami-xyz
    - join(”,” , [ "ami-xyz","AMI-ABC","ami-efg" ]) : `"ami-xyz,AMI-ABC,ami-efg"`
- Collection 함수
    - length(var.ami) = 3
    - index(var.ami, “AMI-ABC”) = 1
    - element(var.ami,2) = ami-efg
    - contains(var.ami, “AMI-ABC”) = true (요소가 있는 지 없는 지)
- MAP 관련 함수 → map 함수는 지원하지 않고, `tomap` 함수를 지원
    
```go
variable "ami" {
  type = map
  default = { 
    "us-east-1" = "ami-xyz",
    "ca-central-1" = "ami-efg",
    "ap-south-1" = "ami-ABC"
  }
  description = "A map of AMI ID's for specific regions" 
}
```
`lookup (var.ami, "us-east-1")` 명령어를 실행하면 `ami-xyz`를 반환한다.

## 프로비저너

프로비저너는 프로바이더로 실행되지 않는 커맨드와 파일 복사 같은 역할을 수행한다. 

> **프로비저너로 실행된 결과는 테라폼의 상태 파일과 동기화되지 않으므로 프로비저닝에 대한 결과가 항상 같다고 보장할 수 없다** ⇒ **선언적 보장 안됨**
> 

그렇기에, 프로비저너보단 userdata 등을 사용하는 것이 좋다.

프로비저너는 생성할 때만 실행되고 추후 작업은 없다. 그래서 `provisioner`가 실패하면 리소스가 잘못되었다고 판단하고 다음 `terraform apply` 할 때 제거하거나 다시 생성한다. 

`provisioner`에서 `when = "destroy"`를 지정하면 해당 프로비저너는 리소스를 제거하기 전에 실행되고 프로비저너가 실패한다면 다음 `terraform apply` 할 때 다시 실행하게 된다. 문서에 따르면 이 때문에 제거 프로비저너는 여러 번 실행해도 괜찮도록 작성해야 한다고 한다. 

---
**참고 자료**

앤서블과 연동해서 쓸거면, 아래의 링크로 진행하면 된다.

[https://github.com/ansible/terraform-provider-ansible/tree/main/examples](https://github.com/ansible/terraform-provider-ansible/tree/main/examples)

---

아래와 같이 원격에 내용을 전달할 수 있지만, 가급적 user_data를 사용하는 것이 좋다.

`user_data = base64encode(templatefile("${path.module}/ubuntu_docker.tftpl", {}))` 

**connection**: remote-exec와 file 프로비저너를 사용하려면, 원격에 연결할 정보를 명시해야 한다. 주로 SSH/WinRM만 존재한다.

```go
resource "aws_instance" "web" {
	...
  connection {
    type     = "ssh"
    user     = "root"
    password = var.root_password
    host     = self.public_ip
  }

  provisioner "file" {
    source      = "script.sh"
    destination = "/tmp/script.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/script.sh",
      "/tmp/script.sh args",
    ]
  }
}
```

## null resource

아무작업도 수행하지 않는 리소스이다.

> 이런 리소스가 필요한 이유는 테라폼 프로비저닝 동작을 설계하면서 사용자가 의도적으로 프로비저닝하는 동작을 조율해야 하는 상황이 발생하여, 프로바이더가 제공하는 리소스 수명주기 관리만으로는 이를 해결하기 어렵기 때문이다.
> 

**주로 사용되는 시나리오**

- 프로비저닝 수행 과정에서 명령어 실행
- 프로비저너와 함께 사용
- 모듈, 반복문, 데이터 소스, 로컬 변수와 함께 사용
- 출력을 위한 데이터 가공

**예시 상황**

EC2의 인스턴스로 웹서비스를 실행한다. 웹서비스 설정에 고정된 IP(EIP)가 필요하다. 

아래와 같이 순환참조 에러가 발생하는 상황에서, null_resource를 추가해 해결할 수 있음

```go
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_security_group" "instance" {
  name = "t101sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

}

resource "aws_instance" "example" {
  ami                    = "ami-0c9c942bd7bf113a2"
  instance_type          = "t2.micro"
  subnet_id              = "subnet-dbc571b0" 
  private_ip             = "172.31.1.100"
  vpc_security_group_ids = [aws_security_group.instance.id]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, T101 Study" > index.html
              nohup busybox httpd -f -p 80 &
              EOF

  tags = {
    Name = "Single-WebSrv"
  }
	# (1) 여기에서 eip 리소스에 대한 접근을 하면 순환참조로 에러가 발생함. 
  provisioner "remote-exec" {
    inline = [
      "echo ${aws_eip.myeip.public_ip}"
     ]
  }
}
# (1)번의 내용을 대체할 수 있는 Null_resource
resource "null_resource" "echomyeip" {
  provisioner "remote-exec" {
    connection {
      host = aws_eip.myeip.public_ip
      type = "ssh"
      user = "ubuntu"
      private_key =  file("/home/kaje/kp-kaje.pem") # 각자 자신의 EC2 SSH Keypair 파일 위치 지정
      #password = "qwe123"
    }
    inline = [
      "echo ${aws_eip.myeip.public_ip}"
      ]
  }
}

resource "aws_eip" "myeip" {
  #vpc = true
  instance = aws_instance.example.id
  associate_with_private_ip = "172.31.1.100"
}

output "public_ip" {
  value       = aws_instance.example.public_ip
  description = "The public IP of the Instance"
}
```

## terraform_data

이 리소스 또한, null_resource와 동일한 역할을 하나, 테라폼 자체에 포함된 **기본 수명주기 관리자**가 제공된다.

**triggers_replace:** 인스턴스의 상태를 저장하며, 상태가 변경되면 아래의 명령어를 수행한다.

아래의 예시를 통해 확인하면, 

```go
resource "terraform_data" "foo" {
  triggers_replace = [
    local_file.foo
  ]
  provisioner "local-exec" {
    command = "echo 'terraform_data test'"
  }
}
output "terraform_data_output" {
  value = terraform_data.foo.output # 출력 결과는 "world"
}

variable "enable_file" {
  default = true
}

resource "local_file" "foo" {
  count    = var.enable_file ? 1 : 0
  content  = "foo!"
  filename = "${path.module}/foo.bar"
}

output "content" {
  value = var.enable_file ? local_file.foo[0].content : ""
}
```

1. terraform apply 이후, foo.bar의 내용을 수정했을 때

```go
$ terraform apply -auto-approve
local_file.foo[0]: Refreshing state... [id=4bf3e335199107182c6f7638efaad377acc7f452]
terraform_data.foo: Refreshing state... [id=bdfbfd37-fccc-4f02-6542-08f1bbb3d2a1]
...
terraform_data.foo: Destroying... [id=bdfbfd37-fccc-4f02-6542-08f1bbb3d2a1]
terraform_data.foo: Destruction complete after 0s
local_file.foo[0]: Creating...
local_file.foo[0]: Creation complete after 0s [id=4bf3e335199107182c6f7638efaad377acc7f452]
terraform_data.foo: Creating...
terraform_data.foo: Provisioning with 'local-exec'...
terraform_data.foo (local-exec): Executing: ["/bin/sh" "-c" "echo 'terraform_data test'"]
terraform_data.foo (local-exec): terraform_data test
terraform_data.foo: Creation complete after 0s [id=4301eef8-bbc6-58f0-6948-a6f6ac176b6c]

Apply complete! Resources: 2 added, 0 changed, 1 destroyed.

Outputs:

content = "foo!"
```

1. (triggers_replace를 주석으로 제거한 뒤)terraform apply 이후, foo.bar의 내용을 수정했을 때

```bash
$ terraform apply
terraform_data.foo: Refreshing state... [id=3396b6e8-ffca-dac3-f0d5-c41455864c42]
local_file.foo[0]: Refreshing state... [id=4bf3e335199107182c6f7638efaad377acc7f452]

Terraform used the selected providers to generate the following execution plan. Resource actions are
indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # local_file.foo[0] will be created
  + resource "local_file" "foo" {
      + content              = "foo!"
      + content_base64sha256 = (known after apply)
      + content_base64sha512 = (known after apply)
      + content_md5          = (known after apply)
      + content_sha1         = (known after apply)
      + content_sha256       = (known after apply)
      + content_sha512       = (known after apply)
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "./foo.bar"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

local_file.foo[0]: Creating...
local_file.foo[0]: Creation complete after 0s [id=4bf3e335199107182c6f7638efaad377acc7f452]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

content = "foo!"
```

## moved

state에 기록되는 리소스의 이름이 변경되면 기존 리소스 삭제 후 재생성한다. 이름은 변경하지만, 인프라를 유지하고 싶을 때 moved block을 사용한다.

- 원본

```bash
resource "local_file" "a" {
  content  = "foo!"
  filename = "${path.module}/foo.bar"
}

output "file_content" {
  value = local_file.a.content
}
```

- 이름 수정 후

```bash
resource "local_file" "b" {
  content  = "foo!"
  filename = "${path.module}/foo.bar"
}

moved {
  from = local_file.a
  to   = local_file.b
}

output "file_content" {
  value = local_file.b.content
}
```

이와 같이 moved block을 사용하면, 인프라를 유지할 수 있다.

## 프로바이더

프로바이더란 인프라 리소스를 제공하는 업체라고 생각하면 된다. Terraform은 플러그인을 사용하여 프로바이더라고 불리는 클라우드, SaaS, 다른 API와 상호작용한다.

Terraform 이 어떤 공급자와 사용할 지 표현하기 위해, `provider.tf` 에 별도로 정의한다.

프로바이더는 `terraform init` 명령어를 통해, 필요한 플러그인을 검색 및 다운로드하며 lock.hcl 파일에 프로바이더를 명시하여 앞으로의 코드 수행에서 사용되는 플러그인을 제한한다. (예상하지 못한, 동작을 방지하는 역할을 한다.) `terraform init` 명령어는 백엔드 설정 혹은 프로젝트 시작시 수행하기에 여러 작업이 일어난다. 프로바이더만 업그레이드하고 싶으면, `terraform init -upgrade` 를 수행한다. 

스터디를 미리 진행해주신 악분님이 한눈에 이해할 수 있는 그림을 그려주셨다.

![](https://velog.velcdn.com/images/han-0315/post/1bfc8411-bfc9-490a-91e5-91305e17397b/image.png)
<p align="center">
	출처:[악분님 티스토리](https://malwareanalysis.tistory.com/619)
</p>

아래와 같이, 파트너사 혹은 플러그인을 제공하는 업체라면 테라폼을 통해 리소스를 정의할 수 있다.
Terraform과 파트너 목록은 아래의 이미지 참고
![](https://velog.velcdn.com/images/han-0315/post/3c9477df-6c67-4fa7-9ada-e4ba7d4c39ae/image.png)


- kubernetes 환경 인프라 구축하기
    - `provider`
        ```bash
        terraform {
          required_providers {
            kubernetes = {
              source = "hashicorp/kubernetes"
            }
          }
        }
        
        provider "kubernetes" {
          config_path    = "~/.kube/config"
        }
        ```
        
    - `kubernetes.tf`
        
        ```bash
        resource "kubernetes_deployment" "nginx" {
          metadata {
            name = "nginx-example"
            labels = {
              App = "t101-nginx"
            }
          }
          spec {
            replicas = 2
            selector {
              match_labels = {
                App = "t101-nginx"
              }
            }
            template {
              metadata {
                labels = {
                  App = "t101-nginx"
                }
              }
              spec {
                container {
                  image = "nginx:1.7.8"
                  name  = "example"
        
                  port {
                    container_port = 80
                  }
                }
              }
            }
          }
        }
        
        resource "kubernetes_service" "nginx" {
          metadata {
            name = "nginx-example"
          }
          spec {
            selector = {
              App = kubernetes_deployment.nginx.spec.0.template.0.metadata[0].labels.App
            }
            port {
              node_port   = 30080
              port        = 80
              target_port = 80
            }
        
            type = "NodePort"
          }
        }
        ```
        
- 실행결과(미니큐브로 테스트)

```bash
$ terraform init && terraform plan && terraform apply -auto-approve
...
Plan: 2 to add, 0 to change, 0 to destroy.
kubernetes_deployment.nginx: Creating...
kubernetes_deployment.nginx: Still creating... [10s elapsed]
kubernetes_deployment.nginx: Creation complete after 16s [id=default/nginx-example]
kubernetes_service.nginx: Creating...
kubernetes_service.nginx: Creation complete after 0s [id=default/nginx-example]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
kubernetes_deployment.nginx
kubernetes_service.nginx
```

```bash
Every 1.0s: kubectl get pods,svc                                 MacBook-Pro.local: Wed Sep 13 21:30:54 2023

NAME                                 READY   STATUS    RESTARTS   AGE
pod/nginx-example-868fbd6dcc-8r9bv   1/1     Running   0          89s
pod/nginx-example-868fbd6dcc-xp4rg   1/1     Running   0          89s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        114d
service/nginx-example   NodePort    10.103.116.226   <none>        80:30080/TCP   74s
```

이처럼 정상적으로 테스트된 것을 확인할 수 있다.