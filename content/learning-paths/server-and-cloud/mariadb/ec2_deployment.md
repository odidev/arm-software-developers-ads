---
# User change
title: "Deploy single instance of MariaDB"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy single instance of MariaDB 

## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need an [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start). Create an account if needed. 

Three tools are required on the computer you are using. Follow the links to install the required tools.
* [Terraform](/install-tools/terraform)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)


Before installing EC2 instance of MariaDB via Ansible, [Generate Access Keys](/learning-paths/server-and-cloud/aws/terraform#generate-access-keys-access-key-id-and-secret-access-key), [Generate key-pair using ssh keygen](/learning-paths/server-and-cloud/aws/terraform#generate-key-pairpublic-key-private-key-using-ssh-keygen).

## Deploy EC2 instance via Terraform

After generating the public and private keys, we have to create an EC2 instance. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22`(ssh) and `3306`(MariaDB). Below is a Terraform file named **main.tf** that will do this for us.
   

```console
provider "aws" {
  region = "us-east-2"
  access_key  = "Axxxxxxxxxxxxxxxxxxxx"
  secret_key   = "Rxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}

resource "aws_instance" "Mariadb_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity.name]
  key_name = "mariadb_key"
  tags = {
    Name = "Mariadb_TEST"
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
    from_port        = 3306
    to_port          = 3306
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

resource "local_file" "inventory" {
    depends_on=[aws_instance.Mariadb_TEST]
    filename = "/home/ubuntu/inventory.txt"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${aws_instance.Mariadb_TEST.public_ip} ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "mariadb_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQClK0s+DjdgSVcOhCy34VF605SuyJOQcv/ZASuXpxh7PqrANXZzGDxLaDkGS4ovxt5t78BDJcohe+SxiCaZCSKwHg0M75e6FTYaC2y0bKfD7FXGAWBJtC5mF927XdwfEDLxmxYtKDRWZT0SMQdGwxL50hnvzVoPmV8SafQAEvKTmF7qB+8ovpOQFbCQDZ8BFg3eNplYwJHcT8a3ErmBiAe6o2qjUDyoYaYGOM+xpbT8HQ/CbuV5qyEHWqAAc4bgtRrJc/waZt8+NqWYmzYc5swr8bg/ILKhxYz4Llm2Mx6HQUUKiCk8ywj26Yt5zBz/c9ErSXKlvxRt2afwJKrt0XWb ubuntu@ip-172-31-93-112"
}

```
**NOTE:-** Replace **public_key**, **access_key**, **secret_key**, and **key_name** with your values.

Now, use the [Terraform commands](/learning-paths/server-and-cloud/aws/terraform#terraform-commands) to deploy **main.tf** file. After successful deployment of EC2 instance we need to configure MariaDB on the same via Ansible


## Configure MariaDB through Ansible
Ansible is a software tool that provides simple but powerful automation for cross-platform computer support.

To deploy MariaDB instance, we have to create a **.yml** file, which is also known as **Ansible-Playbook**. Below is the ansible-playbook named **mariadb_module.yml** .

```console
---
- hosts: all
  remote_user: root
  become: true
  tasks:
    - name: Update the Machine and Install dependencies
      shell: |
             apt-get update -y
             apt-get -y install mariadb-server
             apt -y install python3-pip
             pip3 install PyMySQL
      become: true
    - name: start and enable maridb service
      service:
        name: mariadb
        state: started
        enabled: yes
    - name: Change Root Password
      shell: sudo mariadb -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '{{Your_mariadb_password}}'"
    - name: Create database user with password and all database privileges and 'WITH GRANT OPTION'
      mysql_user:
         login_user: root
         login_password: {{Your_mariadb_password}}
         login_host: localhost
         name: Local_user
         host: '%'
         password: {{Give_any_password}}
         priv: '*.*:ALL,GRANT'
         state: present
    - name: Create a new database with name 'arm_test'
      community.mysql.mysql_db:
        name: arm_test
        login_user: root
        login_password: {{Your_mariadb_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
    - name: MariaDB secure installation
      become: yes
      expect:
        command: mariadb-secure-installation
        responses:
           'Enter current password for root': '{{Your_mariadb_password}}'
           'Set root password': 'n'
           'Remove anonymous users': 'y'
           'Disallow root login remotely': 'n'
           'Remove test database': 'y'
           'Reload privilege tables now': 'y'
        timeout: 1
      register: secure_mariadb
      failed_when: "'... Failed!' in secure_mariadb.stdout_lines"
    - name: Enable remote login by changing bind-address
      lineinfile:
         path: /etc/mysql/mariadb.conf.d/50-server.cnf
         regexp: '^bind-address'
         line: 'bind-address = 0.0.0.0'
         backup: yes
      notify:
         - Restart mariadb
  handlers:
    - name: Restart mariadb
      service:
        name: mariadb
        state: restarted

```
**NOTE:-** Replace **{{Your_mariadb_password}}** and **{{Give_any_password}}** with your password.

In the above **mariadb_module.yml** file, we are creating a user (**Local_user**) with all privileges granted and setting the password for the **root** user.
We are also enabling remote login by changing the **bind address** to **0.0.0.0** in the **/mariadb.conf.d/50-server.cnf** file.

In our case, the inventory file will generate automatically after the `terraform apply` command.

### Ansible Commands
To run a Playbook, we need to use the following `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace **{your_yml_file}**, **{your_inventory_file}** and **{path_to_private_key}** with your values.

![Screenshot (408)](https://user-images.githubusercontent.com/92315883/220863383-5e277651-2fda-49b3-b2e2-26e320d9f729.png)

Here is the output after the successful execution of the `ansible-playbook` command.

![Screenshot (409)](https://user-images.githubusercontent.com/92315883/220863364-bd944936-a7dc-4dde-89dc-c478ffe7a385.png)


## Connect to Database using EC2 instance

To connect to the database, we need the **public-ip** of the instance where MariaDB is deployed, which can be found in **inventory.txt** file. We also need to use the MariaDB Client to interact with the MariaDB database.

```console
apt install mariadb-client
```

```console
mariadb -h {public_ip of instance where MariaDB deployed} -P3306 -u {user_name of database} -p{password of database}
```

**NOTE:-** Replace **{public_ip of instance where MariaDB deployed}**, **{user_name of database}** and **{password of database}** with your values. In our case **user_name = Local_user**, which we have created through the **mariadb_module.yml** file. 

![Screenshot (410)](https://user-images.githubusercontent.com/92315883/220863315-06803b90-877d-4a26-a038-2667301fbfce.png)


### Access Database and Create Table

We can access our database by using the below commands.

```console
show databases;
```
```console
use {your_database};
```
![Screenshot (390)](https://user-images.githubusercontent.com/92315883/218905040-05fbbb32-fda0-4022-8323-68b8278cd818.png)

Use the below commands to create a table and insert values into it.

```console
create table book(name char(10),id varchar(10));
```
```console
insert into book(name,id) values ('Abook','10'),('Bbook','20'),('Cbook','20'),('Dbook','30'),('Ebook','45'),('Fbook','40'),('Gbook
','69');
```
```console
describe book;
```

![Screenshot (391)](https://user-images.githubusercontent.com/92315883/218905081-e8e8b1e7-cf17-4b00-9302-f5c9728db1e9.png)

Use the below command to access the content of the table.

```console
select * from {{your_table_name}};
```

![Screenshot (392)](https://user-images.githubusercontent.com/92315883/218905127-ee0dfd54-9068-4bd1-9fdb-4f7a714e6019.png)


