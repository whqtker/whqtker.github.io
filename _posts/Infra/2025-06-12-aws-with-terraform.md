---
title : "[Infra] AWS 인프라 구축 w/ Terraform"
date : 2025-06-12 19:05:00 +0900
categories : [Infra]
tags : [aws, terraform]
---

## 📌 개요

백엔드 엔지니어라면 한 번 쯤 서비스를 배포한 경험이 있을 것이다. 배포에 필요한 AWS EC2와 RDS 등을 생성하기 위해 AWS 콘솔에 접속하여 관리하는 것은 정말 귀찮다. 리소스를 사용하지 않는다면 반드시 해제해야 과금이 발생하지 않는데, 자꾸 까먹어서 날린 돈만 얼마인지 모르겠다. 콘솔에 매번 접속해서 리소스를 관리하지 않고 AWS 리소스를 간편하게 생성 및 해제할 수 있는 도구가 있는데, 바로 `Terraform` 이다.

## 📌 Terraform이란?

Terraform은 HashiCorp에서 개발한 `IaC(Infrasctucture as Code)` 도구이다. 즉. 코드를 작성하여 인프라를 관리 및 감독할 수 있게 하는 도구인 것이다. 

### 설치

MacOS를 사용한다면 아래 명령을 통해 Terraform을 설치할 수 있다.

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

Windows 환경은 [Terraform 설치 링크](https://developer.hashicorp.com/terraform/install)에 접속하여 설치 파일을 다운로드하면 된다.

![image.png](assets/img/aws-terraform/1.png)

다운로드한 압축 파일 내부의 `terraform.exe` 파일을 `C:\lib\terraform` 으로 이동시킨다. 그 후 시스템 환경 변수 `Path` 에 `C:\lib\terraform` 을 추가한다.

![image.png](assets/img/aws-terraform/2.png)

## 📌 AWS CLI

### IAM 권한 설정

AWS 리소스를 만들기 위한 IAM 사용자를 생성하고 권한을 부여한다.

먼저 AWS root 계정으로 로그인한 후, IAM 메뉴에 접속한다.

![image.png](assets/img/aws-terraform/3.png)

사용자 - 사용자 생성 버튼을 클릭한다.

![image.png](assets/img/aws-terraform/4.png)

사용자 이름을 입력하고 ‘AWS Management Console에 대한 사용자 액세스 권한 제공’에 체크한 후, ‘IAM 사용자를 생성하고 싶음’를 선택한다.

![image.png](assets/img/aws-terraform/5.png)

생성할 사용자에 권한을 연결한다. `AdministratorAccess` 권한을 연결해주었다.

![image.png](assets/img/aws-terraform/6.png)

사용자가 성공적으로 생성되었다.

![image.png](assets/img/aws-terraform/7.png)

생성한 IAM 사용자로 로그인을 진행한다.

![image.png](assets/img/aws-terraform/8.png)

### AWS CLI 설치

[AWS CLI 설치 페이지](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/getting-started-install.html)에 접속하여 본인의 운영환경에 따라 AWS CLI를 설치한다.

설치 후 `aws --version` 명령을 입력했을 때 AWS CLI의 버전이 잘 출력되면 성공적으로 설치된 것이다.

![image.png](assets/img/aws-terraform/9.png)

생성된 IAM 사용자 대시보드에 들어가서 보안 자격 증명 - 액세스 키 만들기 버튼을 클릭한다.

![image.png](assets/img/aws-terraform/10.png)

‘Command Line Interface(CLI)’ 를 선택한다.

![image.png](assets/img/aws-terraform/11.png)

생성된 액세스 키와 비밀 액세스 키를 잘 보관한다.

![image.png](assets/img/aws-terraform/12.png)

`aws configure list` 를 통해 현재 등록된 액세스 키를 확인할 수 있다.

![image.png](assets/img/aws-terraform/13.png)

`aws configure` 를 통해 엑세스 키를 등록한다. 등록 후 `aws configure list` 를 통해 잘 등록되었는지 확인한다.

![image.png](assets/img/aws-terraform/14.png)

혹여나 액세스 키를 분실한 경우 액세스 키를 비활성화한 다음 새로 생성해야 한다. 사용자 대시보드에서 보안 자격 증명 - 액세스 키 - 작업 - 비활성화 버튼을 클릭한다.

![image.png](assets/img/aws-terraform/15.png)

> AWS CLI에 등록된 액세스 키 정보를 삭제하는 명령은 `rmdir /s /q "%USERPROFILE%/.aws"` 이다. (Windows)
> 

## 📌 AWS 리소스 생성

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_vpc" "test" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "test"
  }
}
```

VPC를 하나 생성하는 `main.tf` 이다.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

`terraform` 블록에서 Terraform을 설정한다. `required_providers`는 Terraform 구성에 사용할 프로바이더를 정의한다. 프로바이더는 특정 클라우드 서비스나 API와 상호작용할 수 있게 하는 라이브러리이다. 위 예제에서는 AWS 프로바이더를 사용하고 있다. `source`는 프로바이더의 출처이며, `version`은 프로바이더의 버전이다.

```hcl
provider "aws" {
  region = var.region
}
```

AWS와 상호작용하기 위한 프로바이더를 설정하는 부분이다. `region` 으로 AWS 리전을 설정한다.

```hcl
resource "aws_vpc" "test" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "test"
  }
}
```

VPC를 생성하는 리소스 블록이다. `cidr_block` 은 VPC가 사용할 IP 주소 범위이다.

---

`terraform init` 으로 작업 환경을 초기화한다.

![image.png](assets/img/aws-terraform/16.png)

`terraform plan` 으로 리소스를 적용하기 전 어떠한 변경이 있는지 확인할 수 있다. 즉, Execution Plan(실행 계획)을 확인할 수 있다.

![image.png](assets/img/aws-terraform/17.png)

`terraform apply` 로 실제 인프라 리소스를 생성(또는 수정 및 삭제)할 수 있으며, `terraform destroy` 를 통해 인프라 리소스를 삭제할 수 있다.

![image.png](assets/img/aws-terraform/18.png)

AWS 콘솔에서 VPC 대시보드에 접근하면 생성된 VPC를 확인할 수 있다.

![image.png](assets/img/aws-terraform/19.png)