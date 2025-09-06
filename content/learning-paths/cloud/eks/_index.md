---
title: "Deploy WordPress with MySQL on Elastic Kubernetes Service"

description: >
    Provisioning of EKS cluster on Arm-based instance and then deployment of Wordpress(with Mysql) on Elastic Kubernetes Service.

minutes_to_complete: 80

who_is_this_for: >
    Developers new to AWS EC2 and Kubernetes

learning_objectives: 
    - Provision EKS cluster on Arm-based instance
    - Deploy Wordpress(with Mysql) on Elastic Kubernetes Service

prerequisites:
    - An Amazon Web Services (AWS) account
    - Terraform
    - The AWS CLI
    - The Kubernetes CLI also known as `kubectl`

author_primary: Jason Andrews

### Tags
skilllevels: Introductory
subjects: Containers and Virtualization
armips:
    - Neoverse
tools:
    - Terraform
softwares:
operatingsystems:
    - Linux

#       FIXED, DO NOT MODIFY
# ================================================================================
weight: 1                       # _index.md always has weight of 1 to order correctly
layout: "learningpathall"       # All files under learning paths have this same wrapper
learning_path_main_page: "yes"  # Indicates this should be surfaced when looking for related content. Only set for _index.md of learning path content.
---
