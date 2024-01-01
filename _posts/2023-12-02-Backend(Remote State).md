---
layout: post
title: Terraform Backend(Remote State)
date: 2023-12-01 11:00 +0900
description: Terraform Backend 정리
category: [IaC, Terraform]
tags: [IaC, Terraform, Basic, Backend] 
pin: false
math: true
mermaid: true
---
Terraform Backend
<!--more-->


## Backend


원격으로 State 파일을 관리하기 위해선, Backend Block이 필요하다. Terraform Backend Block만 구성하고, 세부적인 내용을 설정하지 않으면 Terraform은 자동으로 Local Backend로 설정한다. **아래와 같이 원격으로 상태파일을 관리하면, state locking 기능덕분에 협업할 때 안전한 작업이 가능하며** 민감한 데이터에 대한 정보를 암호화할 수 있다고도 한다.


**지원목록은 아래와 같다.**

- local
- remote : Terraform Cloud
- azurerm
- consul
- cos
- gcs
- http
- Kubernetes
- oss
- pg
- s3

### Remote State


만약, 팀간의 협업이 필요할 때 state 파일을 로컬에 저장하면 다른 팀원들은 알맞은 상태를 공유받지 못하기에 문제가 발생한다. 이를 공유해야 하며, 만약 동시에 작업할 경우 충돌이 발생한다. 


팀 단위로 작업할 때는 팀원들이 배포 상태를 알고, 누가 먼저 적용할 지 순서도 알아야 한다. 그렇지 않으면 서로가 서로의 변경 사항을 덮어쓰게 된다. 그리고 원격으로 저장을 한다면 파이프라인에 의해 자동으로 최신 상태(state:latest)를 기반으로 배포할 수 있다.

- 아래의 예시는 S3 버킷을 통해 state를 원격으로 관리하는 모습이다.

```go
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.24.0"
    }
  }
  backend "s3" {
    bucket         = "env0-acme-tfstate"
    dynamodb_table = "env0-acme-tfstate-lock"
    key            = "acme-demo-s3"
    region         = "us-west-2"
    role_arn       = "arn:aws:iam::..." 
    # external_id    = "value" 
  }
}
provider "aws" {
  region = "us-west-2"
}
```


### State locking


원격 백엔드의 경우 상태 장금을 사용해 동일한 상태에 대해 Terraform이 실행되는 것을 방지할 수 있다. 


OS에서 동기화를 배웠을 때와 같이 locking을 통해 terraform은 작업을 큐에 대기시켜, 동시작업을 방지한다. 방법은 아래와 같다. 


먼저 DynamoDB에 테이블을 만든다. 이 테이블은 tfstate를 S3에 관리하면서 동시에 작업이 일어나지 않도록 하는 Lock 테이블이다. Lock 테이블은 선택사항이나 Lock 테이블이 있다면 `plan`이나 `apply`를 할 때 먼저 잠그고 작업이 끝나면 잠금을 해제하게 된다.

- terrafrom state 파일 관리 +  Lock 테이블용 **Dynamodb 리소스**

```go
resource "aws_dynamodb_table" "terraform_state_lock" {
  name = "TerraformStateLock"
  read_capacity = 5
  write_capacity = 5
  hash_key = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

- Terraform 설정

```go
terraform {
  required_version = ">= 0.9.5"
  backend "s3" {
    bucket = "kr.ne.outsider.terraform.state"
    key = "test/terraform.tfstate"
    region = "ap-northeast-1"
    encrypt = true
    lock_table = "TerraformStateLock"
    acl = "bucket-owner-full-control"
  }
}
```


이러고 만약 plan 이나, apply를 적용한다면 다음과 같이 Lock을 관리한다.


```bash
$ terraform plan
...

Releasing state lock. This may take a few moments...
```


## 실습


아래 실습은 T101에서 진행한 코드이다. S3 + dynamodb_table을 이용한다. lock 구현 및 확인하는 코드이다. 동시에 작업하면 `locking state` 에러 메시지 발생해야하지만, 생각보다 코드가 빨리 실행되어 여기서는 lock state 메세지를 보기 힘들다. 기업이나 큰 프로젝트의 인프라인 경우 유용하게 쓰일 듯하다.


### **AWS S3/DynamoDB 백엔드** 


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


![Untitled.png](/assets/img/post/Backend(Remote%20State)/1.png)


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


![Untitled.png](/assets/img/post/Backend(Remote%20State)/2.png)

