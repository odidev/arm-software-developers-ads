---
# User change
title: "Deploy a single instance of PostgreSQL"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy a single instance of PostgreSQL 

## Before you begin
Any computer which has the required tools installed can be used for this section.

You will need an [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start). Create an account if needed.

Below tools are required on the computer you are using. Follow the links to install the required tools.
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/install-tools/terraform)

## Generate Access keys (Access key ID and Secret access key)

The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS. For authentication, generate access keys (Access key ID and Secret access key). These access keys are used by Terraform for making programmatic calls to AWS via AWS CLI. To generate an Access key and Secret key, follow this [documentation](/learning-paths/server-and-cloud/aws/terraform#generate-access-keys-access-key-id-and-secret-access-key)


## Generate key-pair(public key, private key) using ssh keygen

Before using Terraform, first generate the key-pair (public key, private key) using `ssh-keygen`. Then associate both public and private keys with AWS EC2 instances. To generate the key-pair, follow this [documentation](/learning-paths/server-and-cloud/aws/terraform#generate-key-pairpublic-key-private-key-using-ssh-keygen).

## Deploy EC2 instance via Terraform

After generating the public and private keys, we have to create an EC2 instance. Then we will push our public key to the **authorized_keys** folder in **~/.ssh**. We will also create a security group that opens inbound ports **22**(ssh) and **5432**(PSQL). Below is a Terraform file called **main.tf**.


```console

// instance creation
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXXX"
  secret_key  = "AXXXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "PSQL_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity.name]
  key_name = "task2-key"
 
  tags = {
    Name = "PSQL_TEST"
  }
}
resource "aws_default_vpc" "main" {
  tags = {
    Name = "main"
  }
}
resource "aws_security_group" "Terraformsecurity" {
  name        = "Terraformsecurity"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_default_vpc.main.id

ingress {
    description      = "TLS from VPC"
    from_port        = 5432
    to_port          = 5432
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
 ingress {
    description      = "TLS from VPC"
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
    Name = "Terraformsecurity"
  }
 }
output "Master_public_IP" {
  value = [aws_instance.PSQL_TEST.public_ip]
}
 resource "aws_key_pair" "deployer" {
         key_name   = "task2-key"
         public_key = "ssh-rsaxxxxxxxxxxxxxx"
  }
// Generate inventory file
resource "local_file" "inventory" {
    depends_on= [aws_instance.PSQL_TEST]
    filename = "(your_current_directory)/hosts"
    content = <<EOF
          [db_master]
          ${aws_instance.PSQL_TEST.public_ip}         
          [all:vars]
          ansible_connection=ssh
          ansible_user=ubuntu
          EOF
}
```
**NOTE:-** Replace **public_key**, **access_key**, **secret_key**, **key_name** and **filename** with respective values. You can check your current directory using `pwd` command.

Now, use the  Terraform commands below to deploy the **main.tf** file.
## Terraform Commands

### Initialize Terraform and Create a Terraform execution plan

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/learning-paths/server-and-cloud/aws/terraform#initialize-terraform) to initialize and create a Terraform execution plan for main.tf file.

### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan to your cloud infrastructure. The below command creates all required infrastructure.

```console
terraform apply
```      
![image](https://user-images.githubusercontent.com/92078754/217154441-c292c420-8a51-41da-a9dc-49714704cf1c.png)



## Configure PostgreSQL through Ansible

Here we need to install the PostgreSQL database itself along with the python3-psycopg2 Python library which will allow us to use the ansible PostgreSQL modules. Modify the pg_hba.conf file to allow the user to connect with a connection string. Create a database, user for the database and grant access for the user to the created database. Finally, we need to create a table in the database. We will add some dummy data to it by creating an SQL file.

Here is the complete YML file of Ansible-Playbook
```console
---
- hosts: all
  become: yes
  become_method: sudo

  vars_files:
    - vars.yml   
  pre_tasks:
    - name: Update the Machine & Install PostgreSQL
      shell: |
             sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
             wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
             sudo apt-get install postgresql -y
             sudo systemctl start postgresql
             sudo systemctl status postgresql
      become: true
    - name: Update apt repo and cache on all Debian/Ubuntu boxes
      apt:  upgrade=yes update_cache=yes force_apt_get=yes cache_valid_time=3600
      become: true
    - name: Install Python pip & Python package
      apt: name={{ item }} update_cache=true state=present force_apt_get=yes
      with_items:
      - python3-pip
      become: true
  tasks:
    - name: "Find out if PostgreSQL is initialized"
      ansible.builtin.stat:
        path: "/var/lib/pgsql/main/pg_hba.conf"
      register: init_status

    - name: "Start and enable services"
      service: "name={{ item }} state=started enabled=yes"
      with_items:
        - postgresql
    - name: "Create app database"
      postgresql_db:
        state: present
        name: "{{ db_name }}"
      become: yes
      become_user: postgres

    - name: "Create db user"
      postgresql_user:
        state: present
        name: "{{ db_user }}"
        password: "{{ db_password }}"
      become: yes
      become_user: postgres

    - name: "Allow md5 connection for the db user"
      postgresql_pg_hba:
        dest: "~/14/main/pg_hba.conf"
        contype: host
        databases: all
        method: md5
        users: "{{ db_user }}"
        create: true
      become: yes
      become_user: postgres
      notify: restart postgres
    - name: Copy database dump file
      copy:
       src: /tmp/dump.sql
       dest: /tmp
    - name: "Add some dummy data to our database"
      become: true
      become_user: postgres
      shell: psql "{{ db_name }}" < /tmp/dump.sql
  handlers:
    - name: restart postgres
      service: name=postgresql state=restarted

```
**NOTE:** Replace **db_name**, **db_user** and **db_password** with your database name, user and password respectively or you can add all these variables in the [vars.yml](https://github.com/puppetlabs/pdk-docker/files/10739641/vars.txt) file. 

In our case, the hosts(inventory) file is generated automatically after the terraform apply command. 
We are using [dump.sql](https://github.com/puppetlabs/pdk-docker/files/10728905/dump.txt) file to create a table and insert values into the database. 

### Ansible Commands

To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_hosts_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace **{{ your_yml_file }}**, **{your_hosts_file}** and **{path_to_private_key}** with respective values.

![image](https://user-images.githubusercontent.com/92078754/221488191-5047021c-e14c-4196-aa38-21c7bdee0cc0.png)

Here is the output after successful execution of the **ansible-playbook** command.

![image](https://user-images.githubusercontent.com/92078754/221488280-d1d281b0-6ebd-4001-9131-4784b92e1fb0.png)


## Connect to Database 

To connect to the database, we need the **host(public-ip of the node)** where PostgreSQL is deployed. 

```console
ssh -i ~/.ssh/private_key username@host
```
**NOTE:-** Replace **{private_key}**, **{host}** and **username** with respective values.

![image](https://user-images.githubusercontent.com/92078754/221488467-cb4ec054-e351-45a0-a408-70d6c9c922f9.png)

Next, log into the postgres by using the below commands.
```console
cd ~postgres/
sudo su postgres -c psql
```
![image](https://user-images.githubusercontent.com/92078754/221488904-46ec8a8d-5800-4f63-a4f4-9081ccb0609d.png)

Use the below command to show databases and tables.

```console
 \l;
```
![image](https://user-images.githubusercontent.com/92078754/218688208-abaa4da6-ef1a-45bd-a00c-e2e1502f80a3.png)

Use the below command to use existing databases.
```console
 \c testdb;
```
![image](https://user-images.githubusercontent.com/92078754/218688529-8bdbb62f-3ac8-49b1-a3d0-7647b2bff50a.png)

Use the below command to show the tables.

```console
 \dt;
```
![image](https://user-images.githubusercontent.com/92078754/218688649-679ace9f-1711-4181-a0b7-aa25e8a9ae8e.png)

Use the below command to access the content of the table.

```console
select * from teachers;
```
![image](https://user-images.githubusercontent.com/92078754/218688811-e9294095-ebe5-4c0f-b74f-770dcee777f5.png)


