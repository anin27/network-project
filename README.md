## What I Built

This is my CA project for the Network Systems and Administration module. The idea was to take a simple web app and deploy it on AWS automatically — without manually logging into any server or clicking around the AWS console.

I set up the whole thing using Terraform, Ansible, Docker and GitHub Actions. Each tool has its own job in the pipeline and together they make the deployment fully automated.

Everything was built and tested on Ubuntu using WSL (Windows Subsystem for Linux) on Windows.

---

## How It Works

1. **Terraform** creates the AWS EC2 server and configures the security group (ports 22 and 80)
2. **Ansible** connects to the server over SSH, installs Docker, and deploys the container
3. **Docker** runs the nginx web app inside a container on port 80
4. **GitHub Actions** automatically validates and redeploys everything whenever I push to main



## Prerequisites

- AWS account with credentials configured (aws configure)
- SSH key pair called nw-key created in AWS eu-west-1 region
- nw-key.pem downloaded and placed inside the terraform/ folder
- Terraform installed
- Ansible installed
- Fix key permissions:


## Steps to Deploy

**Step 1 — Provision the server with Terraform**

cd terraform
terraform init
terraform plan
terraform apply


After apply finishes it will print the public IP of your EC2 instance. Copy that IP and update it in ansible/inventory.

**Step 2 — Configure server and deploy app with Ansible**


ansible-playbook -i ansible/inventory ansible/nginx-playbook.yml \
  --private-key terraform/nw-key.pem \
  --user ec2-user


**Step 3 — Push to GitHub for automatic CI/CD**

Any push to main triggers GitHub Actions which validates all configs and redeploys the container automatically. Add these two secrets in GitHub Settings → Secrets → Actions:

- EC2_KEY — full contents of nw-key.pem file
- EC2_HOST — EC2 public IP address


## Terraform Configuration (terraform/main.tf)


provider "aws" {
  region = "eu-west-1"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_security_group" "web_sg" {
  name        = "web-security-group"
  description = "Allow SSH and HTTP"

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
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

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  key_name               = var.key_name
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  tags = {
    Name = "project-nw"
  }
}


## Terraform Variables (terraform/variable.tf)


variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "key_name" {
  type    = string
  default = "nw-key"
}


## Terraform Output (terraform/output.tf)


output "instance_public_ip" {
  value = aws_instance.web.public_ip
}


## Ansible Inventory (ansible/inventory)


[web]
54.78.229.16 ansible_user=ec2-user ansible_python_interpreter=/usr/bin/python3


## Ansible Playbook (ansible/nginx-playbook.yml)

- name: Configure EC2 and Deploy Docker App
  hosts: web
  become: yes
  gather_facts: yes

  tasks:

    - name: Install Docker (Amazon Linux 2)
      yum:
        name: docker
        state: present

    - name: Start and enable Docker
      service:
        name: docker
        state: started
        enabled: yes

    - name: Add ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes

    - name: Create project directory
      file:
        path: /home/ec2-user/network-project/app
        state: directory
        owner: ec2-user
        group: ec2-user
        mode: '0755'
        recurse: yes

    - name: Copy app files to EC2
      copy:
        src: ../app/
        dest: /home/ec2-user/network-project/app/
        owner: ec2-user
        group: ec2-user
        mode: '0755'

    - name: Stop existing container if running
      command: docker stop network-container
      ignore_errors: yes

    - name: Remove existing container if exists
      command: docker rm network-container
      ignore_errors: yes

    - name: Remove old Docker image
      command: docker rmi network-app
      ignore_errors: yes

    - name: Build Docker image
      command: docker build --no-cache -t network-app .
      args:
        chdir: /home/ec2-user/network-project/app

    - name: Run Docker container
      command: docker run -d -p 80:80 --name network-container network-app

## Dockerfile (app/Dockerfile)

FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

## GitHub Actions Pipeline (.github/workflows/main.yml)


name: DevOps CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:

  build-and-validate:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
        
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.6

      - name: Terraform Init
        working-directory: ./terraform
        run: terraform init -backend=false

      - name: Terraform Validate
        working-directory: ./terraform
        run: terraform validate

      - name: Terraform Format Check
        working-directory: ./terraform
        run: terraform fmt -check

      - name: Install Ansible
        run: |
          sudo apt update
          sudo apt install -y ansible

      - name: Ansible Syntax Check
        run: |
          ansible-playbook ansible/nginx-playbook.yml --syntax-check

      - name: Validate Dockerfile
        run: |
          if [ -f app/Dockerfile ]; then
            echo "Dockerfile exists"
          else
            echo "Dockerfile missing"
            exit 1
          fi

  deploy:
    needs: build-and-validate
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Setup SSH Key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.EC2_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa

      - name: Deploy via SSH
        run: |
          ssh -o StrictHostKeyChecking=no ec2-user@${{ secrets.EC2_HOST }} << 'EOF'
            sudo yum install -y git
            sudo yum install -y docker
            sudo systemctl start docker
            sudo systemctl enable docker
            sudo usermod -aG docker ec2-user
            mkdir -p /home/ec2-user/network-project
            cd /home/ec2-user/network-project
            if [ ! -d .git ]; then
              git clone https://github.com/anin27/network-project.git .
            else
              git pull origin main
            fi
            cd app
            sudo docker stop network-container || true
            sudo docker rm network-container || true
            sudo docker build --no-cache -t network-app .
            sudo docker run -d -p 80:80 --name network-container network-app
          EOF

## Live Website

http://54.78.229.16
