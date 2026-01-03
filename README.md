# Ansible-VM-Health-Monitoring-Project

## Pre-requesites:

-Update the System

```
sudo apt update && sudo apt upgrade -y

```
-Add the Ansible PPA

Ansible provides an official maintained PPA (for latest versions):

```
sudo add-apt-repository --yes --update ppa:ansible/ansible
```

-Install Ansible
 
``` 
sudo apt install ansible -y
```

## Install AWS CLI

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure

```

## Tagging Script:
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
## ansible.cfg file modifications:
```
[defaults]
inventory = inventory/aws_ec2.yml
remote_user = ubuntu
private_key_file = /home/ubuntu/devops-key.pem
host_key_checking = False
timeout = 30

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null

```
## Dynamic Inventory:

-inventory/aws_ec2.yml 
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


## Install venv module if not already present
```
sudo apt install python3-venv -y
```
## Create a virtual environment
```
python3 -m venv ansible-env
```
## Activate it
```
source ansible-env/bin/activate
```
## Install required Python packages
```
pip install boto3 botocore docker
```
## check this point:
```
ansible-galaxy collection install amazon.aws
```
```
ansible-inventory -i inventory/aws_ec2.yaml --graph
```

## Copy Pub Key to access VM via ssh:
```
#!/bin/bash

# Define vars
PEM_FILE="your-key.pem" # add your key
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
USER="ubuntu"  # or ec2-user
INVENTORY_FILE="inventory/aws_ec2.yaml"

# Extract hostnames/IPs from dynamic inventory
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
## Create the Project & Run below Command to execute
```
ansible-playbook calling_playbook.yaml 

```
