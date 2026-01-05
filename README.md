# Ansible VM Health Monitoring Project

This project demonstrates how to use Ansible with AWS Dynamic Inventory to manage and monitor EC2 virtual machines automatically.
It dynamically discovers EC2 instances based on tags, connects via SSH, and runs playbooks for health checks or automation tasks.

## Architecture Diagram:

<img width="1132" height="1070" alt="image" src="https://github.com/user-attachments/assets/e35db4fb-a04d-440e-93a4-a5cf68ef770c" />


## Project Features

Dynamic AWS EC2 inventory using Ansible

Automatic EC2 discovery based on tags

Secure SSH access using key-based authentication

Virtual environment for dependency isolation

Scalable and production-ready setup

## Prerequisites

Ubuntu Linux (20.04 or later recommended)

AWS Account with EC2 instances running

IAM user with EC2 read/tag permissions

Python 3.8+

SSH key pair for EC2 access

## Step 1: Update the System
```
sudo apt update && sudo apt upgrade -y
```
## Step 2: Install Ansible
Add Ansible Official PPA
```
sudo add-apt-repository --yes --update ppa:ansible/ansible
```
## Install Ansible
```
sudo apt install ansible -y
```

## Verify installation:
```
ansible --version
```
## Step 3: Install AWS CLI
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
```

## Configure AWS credentials:
```
aws configure
```
## Step 4: EC2 Auto-Tagging Script

This script tags all running EC2 instances with env=dev and assigns sequential names.
```
#!/bin/bash

instance_ids=$(aws ec2 describe-instances \
  --filters "Name=tag:env,Values=dev" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text)

sorted_ids=($(echo "$instance_ids" | tr '\t' '\n' | sort))

counter=1
for id in "${sorted_ids[@]}"; do
  name="web-$(printf "%02d" $counter)"
  echo "Tagging $id as $name"
  aws ec2 create-tags --resources "$id" \
    --tags Key=Name,Value="$name"
  ((counter++))
done
```

## Make executable:
```
chmod +x tag_instances.sh
./tag_instances.sh
```
## Step 5: Ansible Configuration

Edit ansible.cfg:
```
[defaults]
inventory = inventory/aws_ec2.yml
remote_user = ubuntu
private_key_file = /home/ubuntu/devops-key.pem # add your key path
host_key_checking = False
timeout = 30

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
```
## Step 6: AWS Dynamic Inventory Configuration

Create file: inventory/aws_ec2.yml
```
---
plugin: amazon.aws.aws_ec2

regions:
  - eu-north-1

filters:
  "tag:env": dev
  instance-state-name: running

compose:
  ansible_host: public_ip_address

keyed_groups:
  - key: tags.Name
    prefix: name
  - key: tags.env
    prefix: env
```
## Step 7: Python Virtual Environment Setup

Install virtual environment support:
```
sudo apt install python3-venv -y
```

## Create and activate virtual environment:
```
python3 -m venv ansible-env
source ansible-env/bin/activate

```
## Install required Python packages:
```
pip install boto3 botocore docker
```
## Step 8: Install Required Ansible Collection
```
ansible-galaxy collection install amazon.aws

```
## Verify dynamic inventory:
```
ansible-inventory -i inventory/aws_ec2.yml --graph
```
## Step 9: Copy SSH Public Key to EC2 Instances

This script injects your local public SSH key into all discovered EC2 instances.
```
#!/bin/bash

PEM_FILE="your-key.pem"
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
USER="ubuntu"
INVENTORY_FILE="inventory/aws_ec2.yml"

HOSTS=$(ansible-inventory -i $INVENTORY_FILE --list | jq -r '._meta.hostvars | keys[]')

for HOST in $HOSTS; do
  echo "Injecting key into $HOST"
  ssh -o StrictHostKeyChecking=no -i $PEM_FILE $USER@$HOST "
    mkdir -p ~/.ssh && \
    echo \"$PUB_KEY\" >> ~/.ssh/authorized_keys && \
    chmod 700 ~/.ssh && \
    chmod 600 ~/.ssh/authorized_keys
  "
done
```
## Step 10: Run the Ansible Playbook
```
ansible-playbook calling_playbook.yaml
```
