# B9IS121 Network Systems — Automated Container Deployment

Student: Anin Thomas | ID: 20076600

## Overview
This project automates the provisioning and deployment of a Dockerised web
application on AWS EC2 using Terraform, Ansible, and GitHub Actions CI/CD.

## Architecture
- Terraform — provisions EC2 instance, security group, resolves AMI
- Ansible — installs Docker, copies app files, builds and runs the container
- GitHub Actions — validates code and deploys on every push to main

## Prerequisites
- AWS credentials configured (aws configure)
- SSH key pair named nw-key created in AWS eu-west-1
- nw-key.pem downloaded and placed in terraform/
- Ansible installed locally

## Usage

### 1. Provision Infrastructure
cd terraform
terraform init
terraform plan
terraform apply

### 2. Deploy Application
cd ansible
source ansible-env/bin/activate
ansible-playbook -i inventory nginx-playbook.yml \
  --private-key ../terraform/nw-key.pem \
  --user ec2-user

### 3. CI/CD
Push to main branch triggers GitHub Actions which validates and deploys automatically.

## Live Deployment
http://54.78.229.16

## Known Limitations
- EC2 IP hardcoded in ansible/inventory — update if instance is reprovisioned
- Uses AWS default VPC
# Updated Sun Mar  1 11:45:18 GMT 2026
