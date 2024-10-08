# 백엔드 CI/CD 구축하기

---

## 해야할 일

---

- CI
    - Terraform 을 활용해 환경 구성하기 ( EC2, VPC, 서브넷 등 )
    - DB 및 BE 도커파일 구성
    - jenkins 연결
    - github webhook 설치
    - jenkins를 통하여 ECR에 도커 이미지 올리기
- CD
    - ECR에 올려진 이미지 pull로 가져와서 실행
    - 이전에 실행했던 컨테이너 삭제

### CI - Terraform을 활용해 환경 구성하기

---

1. 기존에 있던 [main.tf](http://main.tf) 파일에 변수를 변경해서 진행함
    - terraform 스펙
        - 지역 : ap-northeast-2 (서울)
        - 인스턴스 유형 : t3.medium → 추후 축소 예정
        - 인스턴스 개수 : private 1개, public 1개
        - VPC : 1개
        - 서브넷 : private 1개, public 1개
    - main.tf
        
        ```jsx
        provider "aws" {
        	region = var.region
        }
        
        resource "aws_vpc" "terraform_vpc" {
        	cidr_block = var.vpc_cidr
        
        	tags = {
        	    Name = "terraform-vpc"
        	}
        }
        
        resource "aws_internet_gateway" "terraform_igw" {
        	vpc_id = aws_vpc.terraform_vpc.id
        
        	tags = {
        	    Name = "terraform-igw"
        	}
        }
        
        resource "aws_subnet" "terraform_public_subnet" {
        	vpc_id = aws_vpc.terraform_vpc.id
        	cidr_block = var.public_subnet.cidr
        	availability_zone = var.public_subnet.az
        	map_public_ip_on_launch = true
        
        	tags = {
        	   Name = var.public_subnet.name
        	}	
        }
        
        resource "aws_subnet" "terraform_private_subnet" {
        	vpc_id = aws_vpc.terraform_vpc.id
        	cidr_block = var.private_subnet.cidr
        	availability_zone = var.private_subnet.az
        	map_public_ip_on_launch = false
        
        	tags = {
        	  Name = var.private_subnet.name
        	}
        }
        
        resource "aws_route_table" "terraform_public_rt"{
        	vpc_id = aws_vpc.terraform_vpc.id
        	
        	route {
        	   cidr_block = "0.0.0.0/0"
        	   gateway_id = aws_internet_gateway.terraform_igw.id
        	}
        
        	tags = {
        	   Name = "terraform-public-rt"
        	}
        }
        
        resource "aws_eip" "nat_eip" {
        	domain = "vpc"
        	lifecycle {
        	  create_before_destroy = true
        	}
        }
        
        resource "aws_nat_gateway" "nat_gw" {
            allocation_id = aws_eip.nat_eip.id
            subnet_id     = aws_subnet.terraform_public_subnet.id
        
            tags = {
                Name = "terraform-nat-gateway"
            }
        }
        
        resource "aws_route_table" "terraform_private_rt"{
        	vpc_id = aws_vpc.terraform_vpc.id
        
        	route {
        	  cidr_block = "0.0.0.0/0"
        	  nat_gateway_id = aws_nat_gateway.nat_gw.id
        	}
        
        	tags = {
        	  Name = "terraform-private-rt"
        	}
        }
        
        resource "aws_route_table_association" "terraform_public_rta" {
        	subnet_id = aws_subnet.terraform_public_subnet.id
        	route_table_id = aws_route_table.terraform_public_rt.id
        }
        
        resource "aws_route_table_association" "terraform_private_rta" {
                subnet_id = aws_subnet.terraform_private_subnet.id
                route_table_id = aws_route_table.terraform_private_rt.id
        }
        
        resource "aws_security_group" "terraform_sg" {
            vpc_id = aws_vpc.terraform_vpc.id
        
            ingress {
                from_port   = 22
                to_port     = 22
                protocol    = "tcp"
                cidr_blocks = ["0.0.0.0/0"]
            }
        
            ingress {
        	from_port = 3000
        	to_port   = 3000
        	protocol  = "tcp"
        	cidr_blocks = ["0.0.0.0/0"]
            }
        
            egress {
                from_port   = 0
                to_port     = 0
                protocol    = "-1"
                cidr_blocks = ["0.0.0.0/0"]
            }
        
            tags = {
                Name = "terraform-sg"
            }
        }
        
        resource "aws_instance" "public_instance" {
        	count		= var.instance_public_count
        	ami		= "ami-07f15eb4844514508"
        	instance_type	= var.instance_type
        	key_name	= "ktb_test_key"
        	subnet_id       = aws_subnet.terraform_public_subnet.id
        	associate_public_ip_address = true
        	vpc_security_group_ids = [aws_security_group.terraform_sg.id]
        
        	tags = {
        	   Name = "TodayFin be instance"
        	}
        }
        
        resource "aws_instance" "private_instance" {
        	count		= var.instance_private_count
        	ami		= "ami-07f15eb4844514508"
        	instance_type   = var.instance_type
        	key_name	= "ktb_test_key"
        	subnet_id	= aws_subnet.terraform_private_subnet.id
        	associate_public_ip_address = false
        	 vpc_security_group_ids = [aws_security_group.terraform_sg.id]
        
                tags = {
                   Name = "TodayFin Db instance"
                }
        }
        
        ```
        
    - terraform.tfvars
        
        ```jsx
        region = "ap-northeast-2"
        
        instance_public_count = 1
        
        instance_type = "t3.medium"
        
        vpc_cidr = "192.168.0.0/22"
        
        public_subnet = {
        	name = "terraform-public-subnet"
        	cidr = "192.168.0.0/24"
        	az   = "ap-northeast-2a"
        }
        
        private_subnet = {
        	name = "terraform-private-subnet"
        	cidr = "192.168.1.0/24"
        	az   = "ap-northeast-2a"
        }
        ```
        
    - variable.tf
        
        ```jsx
        variable "region" {
        	description = "배포지역"
        	type = string
        	default = "ap-northeast-2"
        }
        
        variable "instance_public_count" {
        	description = "공개 인스턴스 개수"
        	type = number
        	default = 1
        }
        
        variable "instance_private_count" {
        	description = "비공개 인스턴스 개수"
        	type = number
        	default = 1
        }
        
        variable "instance_type" {
        	description = "인스턴스 유형"
        	type = string
        	default  = "t3-medium"
        }
        
        variable "vpc_cidr" {
        	description = "주소 범위"
        	type = string
        	default = "192.168.0.0/16"
        }
        
        variable "public_subnet" {
        	description = "퍼블릭 서브넷 설정"
        	type = map(string)
        	default = {
        		name = "terraform-public-subnet"
        		cidr = "192.168.0.0/24"
        		az   = "ap-northeast-2c"
        	}
        }
        
        variable "private_subnet" {
        	description = "프라이빗 서브넷 설정"
        	type = map(string)
        	default = {
        		name = "terraform-private-subnet"
        		cidr = "192.168.1.0/24"
        		az   = "ap-northeast-2d"
        	}
        }
        
        ```
        

### CI - DB 및 BE 도커파일 구성

---

- BE dockerfile
    - 도커 파일 작성
        
        ```jsx
        // WEB dockerfile
        
        FROM node:22.5-alpine
        
        WORKDIR /app
        
        COPY package*.json ./.
        
        RUN npm install -g pnpm
        
        RUN pnpm install
        
        COPY . .
        
        EXPOSE 5000
        
        CMD ["pnpm","start"]
        ```
        
    - 로컬에서 테스트 후 alpine이미지로 최소화하여 git에 merge
- DB dockerfile & docker-compose
    - 도커파일 작성
        
        ```jsx
        // DB dockerfile
        
        FROM mariadb:10.11.8
        
        COPY init.sql /docker-entrypoint-initdb.d/
        
        EXPOSE 3306
        ```
        
    - 추후 DB 재부팅 및 편한 관리를 위한 docker-compose 파일 작성
        
        ```jsx
        
        services:
          mariadb:
            build: ./
            container_name: mariadb-server
            ports:
              - "3306:3306"
            env_file:
              - .env
            volumes:
              - db_data:/var/lib/mysql
            networks:
              - mynetwork
        
        volumes:
          db_data:
        
        networks:
          mynetwork:
        ```
        

### CI - jenkins 서버 구축

---

- jenkins, java, docker 설치 ( 공식 문서 참고) https://pkg.jenkins.io/redhat-stable/

```jsx
  // jenkins 패키지 파일 repo 추가
  sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
  sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
  
  // jenkins 사용을 위한 jdk 및 jenkins 설치
  yum install fontconfig java-17-openjdk
  yum install jenkins
  
  // docker 설치
  yum install [docker.io](http://docker.io/) -y
	sudo chown jenkins:jenkins /var/run/docker.sock
	
  systemctl enable docker
  systemctl start docker
```

- jenkins 접속
    - <Instance public ip>:8080
    - password 입력, 초기 비밀번호는 아래 명령어 참고
    
    ```jsx
    cat /var/lib/jenkins/secrets/initialAdminPassword
    ```
    
    - Jenkins관리 - 플러그인 - avaliable plugins  설치하기
        - GitHub Integration, Docker Pipeline, Amazon ECR, AWS Credentials 설치 후 재시작 체크

### CI - github webhook 연결

---

- <repository> Settings - Webhooks - Add webhook
    - Payload URL: http://<JENKINS_SERVER_IP>:8080/githubwebhook/
    - Content type: application/json
    - Secret: 빈칸
    - Just the push event 선택
- 상단 프로필 클릭 - settings 선택 - Developer Settings - Personal access tokens - Tokens (classic)
    - Note : 아무거나
    - Expiration : 7 days
    - Select scopes repo 전체 선택, adminrepo_hook 전체 선택
- Generate Token - ghp_XXX 토큰 복사
Jenkins 대시보드 - Jenkins 관리 - System - Github Server 검색 - +Add - Credential 생성 - Secret에 ghp_XXX 토큰 붙여넣기

### CI - jenkins ecr credential 연결 ( ecr을 위한 IAM Access key 등록 )

---

- Jenkins 관리 - Credentials - (global) 클릭 - Kind : AWS Credentials 선택
- ID: ecr_credentials_id
- ACCESS_KEY_ID 입력, SECRET_ACCESS_KEY 입력

### CI - jenkins pipeline 구축

---

- Jenkins 메인페이지에서 좌측 새로운 Item 클릭 - Pipeline 선택 후 OK
- GitHub project - Project url : github.com/userid/reponame
- GitHub hook trigger for GITScm polling 체크
- jenkins pipeline 코드
    
    ```jsx
    pipeline {
        agent any
    
        environment {
            REPO = 'KTB-LuckyVicky/todayfin-be'
            ECR_REPO = '851725447172.dkr.ecr.ap-northeast-2.amazonaws.com/todayfin'
            ECR_CREDENTIALS_ID = 'ecr:ap-northeast-2:ecr_credentials_id'
        }
    
        stages {
            stage('Checkout') {
                steps {
                    git branch: 'dev', url: "https://github.com/${REPO}.git"
                }
            }
            
            stage('Build docker images') {
                steps {
                    script {
                        dockerImage = docker.build("${ECR_REPO}:latest")
                    }
                }
            }
            
            stage('Push to ECR') {
                steps {
                    script {
                        docker.withRegistry("https://${ECR_REPO}", "${ECR_CREDENTIALS_ID}") {
                            dockerImage.push('latest')
                        }
                    }
                }
            }
    ```
    

### CD - 웹 api 연결을 위한 환경변수 구축

---

- BE에서 .env를 통해 DB들의 정보를 입력받지만, 해당방식으로 빌드할경우 run 할때, .env가 없으므로 따로 환경변수를 입력해줘야함
- jenkins 관리 - System - Global properties - Environment variables

```jsx
DB_NAME = {}
DB_URI = {}

MARIADB_DATABASE = {}
MARIADB_HOST = {}
MARIADB_USER = {}
MARIADB_PASSWORD = {}
MARIADB_PORT = {}
```

- 이름:값 정보 입력후 저장

### CD - 빌드 후 배포를 위한 Script 추가

---

- ECR에 빌드된 이미지를 pull로 내려 받아 docker run 실행

```jsx
pipeline {
    agent any

    environment {
        REPO = 'KTB-LuckyVicky/todayfin-be'
        ECR_REPO = '851725447172.dkr.ecr.ap-northeast-2.amazonaws.com/todayfin'
        ECR_CREDENTIALS_ID = 'ecr:ap-northeast-2:ecr_credentials_id'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'dev', url: "https://github.com/${REPO}.git"
            }
        }
        
        stage('Build docker images') {
            steps {
                script {
                    dockerImage = docker.build("${ECR_REPO}:latest")
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_REPO}", "${ECR_CREDENTIALS_ID}") {
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy Docker') {
            steps {
                script {
		                //  Jenkins 파이프라인 내에서 Docker 레지스트리와의 인증을 처리
                    docker.withRegistry("https://${ECR_REPO}", "${ECR_CREDENTIALS_ID}") {
                        
                        // 기존 container 삭제
                        sh "docker rm -f todayfin-be || true"
                        
                        // 최근 docker image 가져오기
                        sh "docker pull ${ECR_REPO}:latest"
                        
                        // 변수들을 설정하여 docker container 실행
                        sh """
                        docker run -d -p 5000:5000 \
                            -e MARIADB_HOST=${MARIADB_HOST} \
                            -e MARIADB_PASSWORD=${MARIADB_PASSWORD} \
                            -e MARIADB_USER=${MARIADB_USER} \
                            -e MARIADB_PORT=${MARIADB_PORT} \
                            -e MARIADB_DATABASE=${MARIADB_DATABASE} \
                            -e DB_URI='${DB_URI}' \
                            -e DB_NAME=${DB_NAME} \
                            --name todayfin-be \
                            ${ECR_REPO}:latest
                        """
                    }
                }
            }
        }
    }
}
```

- 기존 포트가 겹칠 경우 충돌이 일어나기 때문에, 컨테이너 삭제 후 실행
- docker run 명령어에 필요한 환경 변수들은 Jenkins 환경변수에서 설정된 값들을 가져와서 사용

### CD - Pipeline script from SCM 변경

---

- Definition을 Pipeline script from SCM으로 변경
- Git에 Jenkinefile 생성 (대소문자 주의하기)
- SCM을 Git으로 설정
- Repository URL에 git clone시 사용 되는 Repo URL 설정
- Credentials = 비워두됌
- Branches to build 설정 → jenkinsfile을 찾을 브랜치 설정 ex) */dev
- 장점
    - 타 개발자와 협업하여 관리 할 수 있음
    - 버전관리에 용의함
    - 누가 언제 뭘 변경했는지 쉽게 확인 가능

### 추가적 해야할 일

---

- ec2 db서버 프라이빗으로 분리
- https 로 연결하는법 확인해보기
-
