---
# User change
title: "Install Redis with docker container on a single node"

weight: 3 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Install Redis with docker container on a single node 

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


## Deploy EC2 instance via Terraform

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

  connection {
    type        = "ssh"
    host        = self.public_ip
    user        = "ubuntu"
    private_key = file("/home/ubuntu/aws/aws_key")
    timeout     = "4m"
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
}

resource "local_file" "inventory" {
    depends_on=[aws_instance.redis-deployment]
    filename = "/home/ubuntu/inventory.txt"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${aws_instance.redis-deployment.public_ip} ansible_user=ubuntu
                EOF
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

![image](https://user-images.githubusercontent.com/90673309/213456956-9e5e897a-1cf9-4609-8a56-a0af0f99b3a8.png)


## Install Redis manually on EC2 instance via Ansible
Ansible is a software tool that provides simple but powerful automation for cross-platform computer support.
Ansible allows you to configure not just one computer, but potentially a whole network of computers at once.
To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. The following playbook contains a collection of tasks which install redis using docker.

### Here is the complete YML file of Ansible-Playbook
```console
---
- hosts: all
  become: true
  become_user: root
  remote_user: ubuntu

  tasks:
    - name: Update the Machine
      shell: apt update
    - name: Install docker dependencies
      shell: apt install -y ca-certificates curl gnupg lsb-release
    - name: Create directory
      shell: mkdir /etc/apt/keyrings
    - name: Download docker gpg key
      shell: curl -fsSL 'https://download.docker.com/linux/ubuntu/gpg' | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    - name: Add docker gpg key
      shell: echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    - name: Update the apt sources
      shell: apt update
    - name: Install docker
      shell: apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    - name: Start and enable docker service
      service:
        name: docker
        state: started
        enabled: yes
    - name: Start redis container
      shell: docker run --name redis-container -p 6000:6379 -d redis --protected-mode no
    - name: Connect to redis server using redis client
      shell: docker exec -it redis-container redis-cli set name test
```

To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}` and `{path_to_private_key}` with orignal values.



Here is the output after the successful execution of the `ansible-playbook` command.

![image](https://user-images.githubusercontent.com/90673309/214467310-d79837b6-f612-4dbd-9d60-d63334a9326f.png)

## Connecting to remote Redis server from local machine

Install `redis-cli` on local machine using the commands below.

```console
curl -fsSL "https://packages.redis.io/gpg" | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt update

sudo apt install -y redis-tools redis
```

Redis is binded to localhost (`127.0.0.1`) IP and runs at port `6379` by default. This default configuration makes connecting to redis outside of EC2 impossible. 

To connect with redis installed on remote server we need to run `redis-server` with:
- `--port` option specifying the port number 
- `--protected-mode` option set to `no` to allow connection from redis client (`redis-cli`)
- `--daemonize` option set to `yes` to run redis in background. 

We can select any random available port to start redis server other than `6379` because as soon as redis is installed it runs locally with the localhost (`127.0.0.1`) IP and `6379` port.

We can connect to remote Redis server from local machine using redis-cli using the command below. If we want to enter into redis shell we can keep the `{command}` argument blank.

```console
redis-cli -h {public_dns} -p {port} {command}
```

![image](https://user-images.githubusercontent.com/90673309/214235167-f971cd1d-210c-4e5b-8da5-242594cd895c.png)
