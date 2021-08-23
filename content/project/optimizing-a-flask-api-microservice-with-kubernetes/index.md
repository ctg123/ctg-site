---
title: Optimizing a Flask API microservice with Kubernetes
subtitle: An end to end project that deploys a Flask API microservice to an AWS
  Kubernetes cluster with kOps and Ansible.
date: 2021-08-23T05:31:47.430Z
draft: false
featured: false
tags:
  - Kubernetes
  - Ansible
  - DevOps
  - Python
  - AWS
categories:
  - Cloud-Native
  - CM-Tools
  - Python
external_link: https://github.com/ctg123/ansible-kops
links:
  - url: https://github.com/ctg123/ansible-kops
    name: Github
    icon_pack: fas
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
### **Background of the problem statement:**

A popular payment application, EasyPay where users add money to their wallet accounts, faces an issue in its payment success rate. The timeout that occurs with the connectivity of the database has been the reason for the issue.

While troubleshooting, it is found that the database server has several downtime instances at irregular intervals. This situation compels the company to create their own infrastructure that runs in high-availability mode. 

Given that online shopping experiences continue to evolve as per customer expectations, the developers are driven to make their app more reliable, fast, and secure for improvin

![]()

g the performance of the current system.

## **Create Ansible host Virtual Machine**

We will create our development environment inside a local virtual machine. The logical design is easy to follow if you prefer using a public cloud such as AWS, Azure, GCP, or Digital Ocean. The `vagrantfile` seen below uses an Ubuntu 20.04 base image provided by the Bento project. We'll create a single VM on the local device (i.e., laptop) in VirtualBox via Vagrant. You can set the network as either "private" or "public" if the ports are forwarded when testing Docker images locally.

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "bento/ubuntu-20.04"
  config.vm.hostname = "ansible-controller"
  config.vm.network "public_network", ip: "192.168.1.100"
  config.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
  config.vm.provider "virtualbox" do |vb|
  vb.customize ['modifyvm', :id, '--cableconnected1', 'on']
  end
```

## Install the pre-requisites from the Ansible-KOPS repository

The purpose of this repository is to provide a Kubernetes cluster in a Public Cloud. The deployment of the cluster is fully automated and managed by multiple tools such as Ansible or Kops. It follows the Best Practices of Docker, Kubernetes, AWS, and Ansible as much as possible.

### Required Tools

- - -

* [kOps](https://kops.sigs.k8s.io/) - kOps is an official Kubernetes project for managing production-grade Kubernetes clusters to Amazon Web Services.
* [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/) - kubectl is a command line tool for controlling Kubernetes clusters.
* [Ansible](https://docs.ansible.com/ansible/latest/index.html) - Ansible is a radically simple IT automation platform that makes your applications and systems easier to deploy.
* [Docker](https://docs.docker.com/) - Docker is a set of platform as a service (PaaS) products that use OS-level virtualization to deliver software in packages called containers.
* [Helm](https://helm.sh/) - Helm is a tool for managing Charts. Charts are packages of pre-configured Kubernetes resources.

### Prerequisites

- - -

* An AWS account
* AWS CLI v2
* Ansible
* Boto3 library
* Docker
* Docker Compose
* A registered domain
* Certbot

### Kubernetes add-ons

- - -

* [Kube2IAM](https://github.com/jtblin/kube2iam) - kube2iam provides different AWS IAM roles for pods running on Kubernetes
* [External-DNS](https://github.com/kubernetes-sigs/external-dns) - Configure external DNS servers (AWS Route53, Google CloudDNS and others) for Kubernetes Ingresses and Services.
* [Ingress NGINX](https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx) - Ingress-nginx is an Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer.
* [Cert-manager](https://github.com/jetstack/cert-manager) - Automatically provision and manage TLS certificates in Kubernetes.

```shell
$ vagrant up && vagrant ssh
```

Once inside the virtual machine, clone down the `ansible-kops` repo and set up the environment with the `install_prereqs.sh`.

```shell
$ git clone https://github.com/ctg123/ansible-kops.git
$ cd ansible-kops
$ chmod +x install_prereqs.sh
$ ./install_prereqs.sh
```

You may need to reboot the machine for all the commands to appear. Once finished, check the following commands to verify that AWS CLI, Docker, and Ansible are correctly installed.

```shell
$ aws --version

aws-cli/2.2.29 Python/3.8.8 Linux/5.4.0-58-generic exe/x86_64.ubuntu.20 prompt/off

$ docker version

Client: Docker Engine - Community
 Version:           20.10.8
 API version:       1.41
 Go version:        go1.16.6
 Git commit:        3967b7d
 Built:             Fri Jul 30 19:54:27 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.8
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.6
  Git commit:       75249d8
  Built:            Fri Jul 30 19:52:33 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.9
  GitCommit:        e25210fe30a0a703442421b0f60afac609f950a3
 runc:
  Version:          1.0.1
  GitCommit:        v1.0.1-0-g4144b63
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

$ docker-compose version

docker-compose version 1.27.4, build 40524192
docker-py version: 4.3.1
CPython version: 3.7.7
OpenSSL version: OpenSSL 1.1.0l  10 Sep 2019

$ ansible --version

ansible [core 2.11.3] 
  config file = None
  configured module search path = ['/home/vagrant/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/vagrant/.local/lib/python3.8/site-packages/ansible
  ansible collection location = /home/vagrant/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/vagrant/.local/bin/ansible
  python version = 3.8.10 (default, Jun  2 2021, 10:49:15) [GCC 9.4.0]
  jinja version = 3.0.1
  libyaml = True
```

## Environment Setup

The ansible packages for Kops & Kubectl are compatible with Linux-AMD64 and Darwin-AMD64 architectures.

### AWS account

For the Kubernetes to be fully operational, you need to create an IAM user for your AWS account with the following permissions:

* AmazonEC2FullAccess
* AmazonRouteS3FullAccess
* AmazonS3FullAccess
* IAMFullAccess

Once you have all the permissions, run the following command. Enter your AWS user credentials and select the region you will use in your environment.

```shell
$ aws configure

# Optionally you can specify the AWS Profile
$ aws configure --profile <profile_name>

# You will be prompted for your access keys
AWS Access Key ID [None]: AKIA************
AWS Secret Access Key [None]: kCcT****************
Default region name [None]: us-east-1
Default output format [None]: json
```

### Register your Domain

You must own a registered domain to complete the Kubernetes cluster deployment by using either method(s) seen below

1. New Domain: [Register a new domain](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-register.html) using AWS Route53.
2. Existing Domain: Create a subdomain without migrating the parent domain.

   * Create a hosted zone for your subdomain([example.mydomain.com](http://example.mydomain.com/).)
   * Take note of your NS record.
   * Log into your domain registrar account.
   * Create the corresponding NS record to let your domain know that your subdomain is hosted on AWS.

```shell
DOMAIN                   TTL       TYPE                     TARGET
example.mydomain.com.      0         NS      ns-xxxx.awsdns-xx.org
example.mydomain.com.      0         NS      ns-yyy.awsdns-yy.org
...
```

### Certbot

We'll use the Route53 DNS plugin for Certbot. This plugin automates completing a DNS-01 challenge (DNS01) by creating and subsequently removing TXT records using the Amazon Web Services Route 53 API. My example is for my registered domain. To initiate a DNS challenge, please execute the following command:

```shell
# Set up your domain
# Initiate a dns01 challenge using certbot-dns-route53
# This task is optional if you've already done this before
$ certbot certonly --dns-route53 -d ctgkube.com \
--config-dir ~/.config/letsencrypt \
--logs-dir /tmp/letsencrypt \
--work-dir /tmp/letsencrypt \
--dns-route53-propagation-seconds 30
```

### Environment variables

- - -

You will find all of the environment variables in the `group_vars` directory.

* `group_vars/all.yml` contains the global environment variables.

```yaml
#####################
# ~~ Domain name ~~ #
cluster_name: ctgkube.com

##########################
# ~~ Kops state store ~~ #
bucket_name: ctgadget-kops-store

###############################################
# ~~ SSH public key to access Kubernetes API ~~#
# Enter the full path with the name of the SSH public key created by the generate-ssh-key.yml file.
ssh_pub_key: ~/.ssh/ctgkube.pub

########################################
# ~~ AWS environment for kubernetes ~~ #

# Match with your AWS profile in case of multi-account environment
aws_profile: default
aws_region: us-east-1

# Be careful, master_zones must match maser_node_count
# Example: can't have 1 master in 2 AWS availability zones
master_zones: us-east-1a
aws_zones: us-east-1a,us-east-1b,us-east-1c

# EC2 host sizing
# (Ubuntu 20.04 LTS)
base_image: ami-019212a8baeffb0fa

# Kubernetes master nodes
master_instance_type: t3.medium
master_node_count: 1

# Kubernetes worker nodes
worker_instance_type: t3.medium
worker_node_count: 3

############################################
# ~~ Let's encrypt domain's email owner ~~ #
email_owner: chaance.graves@ctginnovations.io
```