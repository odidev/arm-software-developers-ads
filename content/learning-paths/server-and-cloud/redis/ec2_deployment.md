---
# User change
title: "Install Redis manually on a single node"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Install Redis manually on a single node 

## Prerequisites

* An [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/install-tools/terraform.md)

## Generate Access keys (access key ID and secret access key)

The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS. For authentication, generate access keys (access key ID and secret access key). These access keys are used by Terraform for making programmatic calls to AWS via the AWS CLI.
  
### Go to My Security Credentials
   
![image](https://user-images.githubusercontent.com/87687468/190137370-87b8ca2a-0b38-4732-80fc-3ea70c72e431.png)

### On Your Security Credentials page click on create access keys (access key ID and secret access key)
   
![image](https://user-images.githubusercontent.com/87687468/190137925-c725359a-cdab-468f-8195-8cce9c1be0ae.png)
   
### Copy the Access Key ID and Secret Access Key 

![image](https://user-images.githubusercontent.com/87687468/190138349-7cc0007c-def1-48b7-ad1e-4ee5b97f4b90.png)

## Generate key-pair(public key, private key) using ssh keygen

### Generate the public key and private key

Before using Terraform, first generate the key-pair (public key, private key) using ssh-keygen. Then associate both public and private keys with AWS EC2 instances.

Generate the key-pair using the following command:

```console
ssh-keygen -t rsa -b 2048
```
       
By default, the above command will generate the public as well as private key at location **$HOME/.ssh**. You can override the end destination with a custom path.

Output when a key pair is generated:

![image](https://user-images.githubusercontent.com/90673309/213357856-0aa283e9-2d67-4297-91cc-cfae92bee3c6.png)

      
**Note:** Use the public key aws_key.pub inside the Terraform file to provision/start the instance and private key aws_key to connect to the instance.


## Install Redis manually on EC2 instance via Terraform

After generating the public and private keys, we have to create an EC2 instance. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22`(ssh) and `6000`(Redis). We will also install Redis on remote server using `remote-exec` provisioner. Below is a Terraform file called `main.tf` which will do this for us.

    

```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AAXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}

resource "aws_instance" "redis-deployment" {
  ami = "ami-0888c389af05d881a"
  instance_type = "t4g.small"
  key_name= "aws_key"
  vpc_security_group_ids = [aws_security_group.main.id]

  provisioner "remote-exec" {
    inline = [
      "curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg",
      "echo 'deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main' | sudo tee /etc/apt/sources.list.d/redis.list",
      "sudo apt update",
      "sudo apt install -y redis",
      "redis-server --port 6000 --protected-mode no --daemonize yes",
      "redis-cli -h ${self.public_dns} -p 6000 set name test",
    ]
  }

  connection {
    type        = "ssh"
    host        = self.public_ip
    user        = "ubuntu"
    private_key = file("/home/ubuntu/aws/aws_key")
  }

}

resource "aws_security_group" "main" {
  name        = "main"
  description = "Allow TLS inbound traffic"

  ingress {
    description      = "Open redis connection port"
    from_port        = 6000
    to_port          = 6000
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
  ingress {
    description      = "Allow ssh to instance"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "main"
  }
}

resource "aws_key_pair" "deployer" {
        key_name   = "aws_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCUZXm6T6JTQBuxw7aFaH6gmxDnjSOnHbrI59nf+YCHPqIHMlGaxWw0/xlaJiJynjOt67Zjeu1wNPifh2tzdN3UUD7eUFSGcLQaCFBDorDzfZpz4wLDguRuOngnXw+2Z3Iihy2rCH+5CIP2nCBZ+LuZuZ0oUd9rbGy6pb2gLmF89GYzs2RGG+bFaRR/3n3zR5ehgCYzJjFGzI8HrvyBlFFDgLqvI2KwcHwU2iHjjhAt54XzJ1oqevRGBiET/8RVsLNu+6UCHW6HE9r+T5yQZH50nYkSl/QKlxBj0tGHXAahhOBpk0ukwUlfbGcK6SVXmqtZaOuMNlNvssbocdg1KwOH ubuntu@ip-172-31-XXXX-XXXX"
 }
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with actual values.

Now, use the below Terraform commands to deploy the `main.tf` file.


### Terraform Commands

#### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command is responsible for downloading all dependencies which are required for the AWS provider.

```console
terraform init
```

![image](https://user-images.githubusercontent.com/90673309/213456956-9e5e897a-1cf9-4609-8a56-a0af0f99b3a8.png)

#### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```

**NOTE:** The **terraform plan** command is optional. You can directly run **terraform apply** command. But it is always better to check the resources about to be created.

#### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan to your cloud infrastructure. The below command creates all required infrastructure.

```console
terraform apply
```      

![image](https://user-images.githubusercontent.com/90673309/213504132-f8cf403e-6fd7-4844-9fe1-a54a97cbc185.png)

![image](https://user-images.githubusercontent.com/90673309/213504440-4a14de31-0e74-4081-a08c-1fe6d96befba.png)

## Install Redis manually on EC2 instance via Ansible
Ansible is a software tool that provides simple but powerful automation for cross-platform computer support.
Ansible allows you to configure not just one computer, but potentially a whole network of computers at once.
To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. The following playbook contains a collection of tasks which install redis manually.

### Here is the complete YML file of Ansible-Playbook
```console
---
- hosts: all
  remote_user: root
  become: true

  tasks:
    - name: Update the Machine
      shell: apt update -y
    - name: Set environment variable
      shell: HOST=$(curl -s http://169.254.169.254/latest/meta-data/public-hostname)
    - name: Download redis gpg key
      shell: curl -fsSL https://packages.redis.io/gpg | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
    - name: Add redis gpg key
      shell: echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
    - name: Update the apt sources
      shell: apt update
    - name: Install redis
      shell: apt install -y redis
    - name: Start redis server
      shell: redis-server --port 6000 --protected-mode no --daemonize yes
    - name: Connect to redis server using redis client
      shell: redis-cli -h ${HOST} -p 6000 set name test
```

To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}` and `{path_to_private_key}` with orignal values.



Here is the output after the successful execution of the `ansible-playbook` command.



## Connecting to remote Redis server from local machine

Install redis-cli on local machine using the commands below.

```console
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt update

sudo apt install -y redis
```

We can connect to remote Redis server from local machine using redis-cli using the command below. If we want to enter into redis shell we can keep the `{command}` argument blank.

```console
redis-cli -h {public_dns} -p {port} {command}
```

![image](https://user-images.githubusercontent.com/90673309/213519785-eb5297d6-b207-45db-968a-54883d7031d1.png)
