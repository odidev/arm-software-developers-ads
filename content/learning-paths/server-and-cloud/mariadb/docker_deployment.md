---
# User change
title: "Deploy MariaDB via Docker"

weight: 3 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---


## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need an [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start). Create an account if needed. 

Three tools are required on the computer you are using. Follow the links to install the required tools.
* [Terraform](/install-tools/terraform)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

Before installing MariaDB docker container via Ansible, [Generate Access Keys](/learning-paths/server-and-cloud/aws/terraform#generate-access-keys-access-key-id-and-secret-access-key), [Generate key-pair using ssh keygen](/learning-paths/server-and-cloud/aws/terraform#generate-key-pairpublic-key-private-key-using-ssh-keygen) and [Deploy EC2 instance via Terraform](/learning-paths/server-and-cloud/mariadb/ec2_deployment#deploy-ec2-instance-via-terraform). After successful deployment of EC2 instance, we need to configure MariaDB docker container on the same.

## Deploy MariaDB container using Ansible
Docker is an open platform for developing, shipping, and running applications. Docker enables you to separate your applications from your infrastructure so you can deliver software quickly.

To run Ansible, we have to create a **.yml** file, which is also known as `Ansible-Playbook`.
In our **.yml** file, we use the **community.docker** collection to deploy the MariaDB container.
We also need to map the container port to the host port, which is `3306`. Below is a **.yml** file named **mariadb_module.yml** that will do this for us.

```console
---
- hosts: all
  remote_user: root
  become: true
  tasks:
    - name: Update the Machine and Install dependencies
      shell: |
             apt-get update -y
             apt-get -y install mariadb-client
             apt-get install docker.io -y
             usermod -aG docker ubuntu
             apt-get -y install python3-pip
             pip3 install PyMySQL
             pip3 install docker
      become: true
    - name: Reset ssh connection for changes to take effect
      meta: "reset_connection"
    - name: Log into DockerHub
      community.docker.docker_login:
        username: {{dockerhub_uname}}
        password: {{dockerhub_pass}}
    - name: Deploy mariadb docker container
      docker_container:
        image: mariadb:latest
        name: mariadb_test
        state: started
        ports:
          - "3306:3306"
        pull: true
        volumes:
         - "db_data:/var/lib/mysql:rw"
         - "mariadb-socket:/var/run/mysqld:rw"
         - "/tmp:/tmp:rw"
        restart: true
        env:
          MARIADB_ROOT_PASSWORD: {{your_mariadb_password}}
          MARIADB_USER: local_us
          MARIADB_PASSWORD: Armtest123
          MARIADB_DATABASE: arm_test
    - name: Copy database dump file
      copy:
       src: /home/ubuntu/table.sql
       dest: /tmp
    - name: Run a simple command to populate table
      community.docker.docker_container_exec:
       container: mariadb_test
       command: mariadb -u root -p{{your_mariadb_password}} -e "source /tmp/table.sql;"
       chdir: /root
      register: result

```
**NOTE:**- Replace **docker_container.env** variables of **Deploy mariadb docker container** task with your own MariaDB user and password. Also, replace **{{dockerhub_uname}}** and **{{dockerhub_pass}}** with your dockerhub credentials.

In the above **mariadb_module.yml** file, we are pre-populating our database with the [table.sql](https://github.com/ossdev07/ARMFAAS/files/10857764/table_dot_sql.txt) script file.

In our case, the inventory file will generate automatically after the `terraform apply` command.

### Ansible Commands
To run a Playbook, we need to use the following `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace **{your_yml_file}**, **{your_inventory_file}** and **{path_to_private_key}** with your values.

![Screenshot (417)](https://user-images.githubusercontent.com/92315883/220873128-4da09207-258f-428b-9f27-604b542d5767.png)


Here is the output after the successful execution of the `ansible-playbook` command.

![Screenshot (418)](https://user-images.githubusercontent.com/92315883/220873176-5cc87b87-11c2-48d4-b46c-37ba3d46132f.png)


## Connect to Database using EC2 instance

To connect to the database, we need the **public-ip** of the instance where MariaDB is deployed, which can be found in **inventory.txt** file. We also need to use the MariaDB Client to interact with the MariaDB database.

```console
apt install mariadb-client
```

```console
mariadb -h {public_ip of instance where MariaDB deployed} -P3306 -u {user_name of database} -p{password of database}
```

**NOTE:-** Replace **{public_ip of instance where MariaDB deployed}**, **{user_name of database}** and **{password of database}** with your values. In our case, **user_name**= **local_us**, which we have created through the **mariadb_module.yml** file. 

![Screenshot (420)](https://user-images.githubusercontent.com/92315883/220873547-bcad5a32-80f6-4a25-82b5-960642a4684b.png)


### Access Database and Tables

To access our database and tables, use the below command:

```console
show databases;
```

```console
use {{your_database}}
```

```console
show tables;
```
![Screenshot (398)](https://user-images.githubusercontent.com/92315883/219525956-73468894-b90a-4bd7-b0b4-fa42a57876a0.png)

To view the content of the table:

```console
select * from {{your_table}};
```
![Screenshot (399)](https://user-images.githubusercontent.com/92315883/219525937-ecb2ad70-127d-4231-9ea8-b5a98c5f4d5b.png)

Above table has been created by **table.sql** file through Ansible-playbook. To create your own table, follow this [document](/learning-paths/server-and-cloud/mariadb/ec2_deployment#access-database-and-create-table).

