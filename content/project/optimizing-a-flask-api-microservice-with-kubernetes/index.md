---
title: Optimizing a Flask API microservice with Kubernetes
subtitle: 👇🏾 Click on the button to access the code repository
date: 2021-08-23T05:31:47.430Z
summary: An end-to-end project that deploys a Flask API microservice to an AWS
  Kubernetes cluster with kOps and Ansible.
draft: false
featured: false
authors:
  - ctgadget
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
    icon_pack: fas
    name: Github
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
## **Background of the problem statement:**

A popular payment application, EasyPay where users add money to their wallet accounts, faces an issue in its payment success rate. The timeout that occurs with the connectivity of the database has been the reason for the issue.

While troubleshooting, it is found that the database server has several downtime instances at irregular intervals. This situation compels the company to create its own infrastructure that runs in high-availability mode. 

Given that online shopping experiences continue to evolve as per customer expectations, the developers are driven to make their app more reliable, fast, and secure for improving the performance of the current system.

## **Create Ansible host Virtual Machine**

We will create our development environment inside a local virtual machine. The logical design is easy to follow if you prefer using a public cloud such as AWS, Azure, GCP, or Digital Ocean. The `vagrantfile` seen below uses an Ubuntu 20.04 base image provided by the **[Bento project](https://app.vagrantup.com/bento/boxes/ubuntu-20.04)**. We'll create a single VM on the local device (i.e., laptop) in VirtualBox via Vagrant. You can set the network as either "private" or "public" if the ports are forwarded when testing Docker images locally.

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
* [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/) - kubectl is a command-line tool for controlling Kubernetes clusters.
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
* [External-DNS](https://github.com/kubernetes-sigs/external-dns) - Configure external DNS servers (AWS Route53, Google CloudDNS, and others) for Kubernetes Ingresses and Services.
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

### Generate the SSH Keys and deploy the cluster

- - -

When the environment and the pre-requisites configures, run the following playbooks with Ansible. The `generate-ssh-key.yml` will create the SSH key pair to access Kubernetes API, which you can use to log in to the master node with.
Once generated, make sure the path matches the variable specified in the `group_vars` directory. You're now ready to run the `deploy-cluster.yml` playbook!

```shell
$ ansible-playbook generate-ssh-key.yml
$ ansible-playbook deploy-cluster.yml --ask-become-pass
```

It should take approximately 8 - 10 minutes to complete.

### Install Kubernetes Dashboard

A critical feature for any Kubernetes cluster is efficient monitoring of all resources with an accessible UI. The Kubernetes dashboard enables the ability to deploy containerized applications, troubleshoot pods, and manage other cluster resources such as scaling a deployment, initiating rolling updates, resetting pods instead of using the kubectl command.

Run the `deploy_dashboard.sh` to pull from the latest version of the Kubernetes dashboard stored in the official Github repo. You can check at this **[link](https://github.com/kubernetes/dashboard/releases)** to see which version is the latest release and update the script accordingly. 

Once complete, the default service type configures as a ClusterIP. We will change this to LoadBalancer to access it externally. You can find the service type and edit it with the following command:

```shell
$ kubectl -n kubernetes-dashboard edit svc kubernetes-dashboard
```

Make sure the service type changed to **LoadBalancer** successfully. You should get an AWS ELB address as an output.

```shell
$ kubectl -n kubernetes-dashboard get svc
NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP      100.67.72.147   <none>                                                                    8000/TCP        5h20m
kubernetes-dashboard        LoadBalancer   100.69.145.80   a6c1db00d3d9d42659150be7771c2ba5-1256148891.us-east-1.elb.amazonaws.com   443:31463/TCP   5h20m
```

An output of the script produced a security token needed to log in. Copy the token and enter it in the dashboard. You will then sign into the Kubernetes dashboard. You can retrieve the token when needed with this command:

```shell
$ kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

## Deploy Flask API + MongoDB app on Kubernetes

We will develop a simple Python Flask API application, which will communicate to a MongoDB database, containerize it using Docker, and deploy it to the Kubernetes cluster.

### Prerequisites for development on a local machine

Install the following python libraries using pip located in the `requirements.txt` file in the `payment-app` directory. You will have Flask running locally on your machine prior to deploying the Docker image to Kubernetes.

```shell
$ cd ansible-kops/payment-app
$ python3 -m pip install -r requirements.txt --user
$ pip list
```

### Creating the Flask Payment application

We'll produce a simple RESTful API to create, read, update, and delete (CRUD) payment entries. The app will store the data in a MongoDB database, an open-source database that stores flexible JSON-like documents that is Non-relational (often called NoSQL databases).

By default, when a MongoDB Server instance starts on a machine, it listens to port `27017`. The Flask-PyMongo module helps us to bridge Flask and MongoDB and provides some convenience helpers. An objectId module is a tool for working with MongoDB ObjectId, the default value of _id field of each document, generated during the creation of any document.

The `app.py` which can run on any host (python app.py), can be accessed at `http://localhost:5000/` inside it. 

```python
from flask import Flask, request, jsonify
from flask_pymongo import PyMongo
from bson.objectid import ObjectId
from flask_cors import CORS
import socket

# Configuration
DEBUG = True

# Instantiate the app
app = Flask(__name__)
app.config["MONGO_URI"] = "mongodb://mongo:27017/dev"
app.config['JSONIFY_PRETTYPRINT_REGULAR'] = True
mongo = PyMongo(app)
db = mongo.db

# enable CORS
CORS(app, resources={r'/*': {'origins': '*'}})

# UI message to show which pods the Payment API container is running
@app.route("/")
def index():
    hostname = socket.gethostname()
    return jsonify(
        message="Welcome to the EasyPay app. I am running inside the {} pod!".format(hostname)
    )

@app.route("/payments")
def get_all_payments():
    payments = db.payment.find()
    data = []
    for payment in payments:
        item = {
            "id": str(payment["_id"]),
            "payment": payment["payment"]
        }
        data.append(item)
    return jsonify(
        data=data
    )

# POST Method to collect a user's payment
@app.route("/payments", methods=["POST"])
def add_payment():
    data = request.get_json(force=True)
    db.payment.insert_one({"payment": data["payment"]})
    return jsonify(
        message="Payment saved successfully to your account!"
    )

# PUT Method to update a user's payment
@app.route("/payments/<id>", methods=["PUT"])
def update_payment(id):
    data = request.get_json(force=True)["payment"]
    response = db.payment.update_one({"_id": ObjectId(id)}, {"$set": {"payment": data}})
    if response.matched_count:
        message = "Payment updated successfully!"
    else:
        message = "No Payments were found!"
    return jsonify(
        message=message
    )

# DELETE Method to delete a user's payment
@app.route("/payments/<id>", methods=["DELETE"])
def delete_payment(id):
    response = db.payment.delete_one({"_id": ObjectId(id)})
    if response.deleted_count:
        message = "Payment deleted successfully!"
    else:
        message = "No Payments were found!"
    return jsonify(
        message=message
    )

# POST Method to delet all payment data
@app.route("/payments/delete", methods=["POST"])
def delete_all_payments():
    db.payment.remove()
    return jsonify(
        message="All Payments deleted!"
    )

# The app server will be able to run locally at port 5000
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

We first import all the required modules and create instances of the Flask class (the app) and the PyMongo class (the database). Note that the hostname in the `MONGO_URI` Flask configuration variable defines the mongo instead of localhost. Mongo will be the name of our database container, and containers in the same Docker network can talk to each other by their names.

Our app consists of six functions which are assigned URLs by @app.route() Python decorator. At first glance, it is easy to understand that the decorator is telling our app that executes the underlying function whenever a user visits our @app domain at the given route().

* `index()` - displays a welcome message for the app. It Also displays the hostname of the machine where our app is running. This is useful to understand that we will be hitting a random pod each time we try to access our app on Kubernetes.
* `get_all_payments()` - displays all the payments that are available in the database as a list of dictionaries.
* `add_payment()` - adds a new payment that is stored in the database with a unique ID.
* `update_payment(id)` - modifies any existing payment entry. If no payment data is found with the queried ID, the appropriate message is returned.
* `delete_payment(id)` - removes that entry of the task having the queried ID from the database. Returns appropriate message if no task with the specified ID is found.
* `delete_all_payments()` - removes all the payment data and returns an empty list.

In the final section, where we run the app, we define the host parameter as **'0.0.0.0'** to make the server publicly available, running on the machine's IP address, which will be inside a unique container.

### Containerizing the application

Once you have Docker installed locally, we will store our images to Docker Hub. Use the `docker login` command to authorize Docker to connect to your `Docker Hub` account.

Let's build a Docker image of the app to push to the Docker Hub registry. In the directory `payment-app`, a `Dockerfile` with the following contents to create the image:

```dockerfile
######################################
# ~~ DOCKERFILE for Flask API app ~~ #
######################################

FROM python:alpine3.9
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
ENV PORT 5000
EXPOSE 5000
ENTRYPOINT [ "python" ]
CMD [ "app.py" ]
```

We are using the official Python3.9 image, based on the Alpine Linux project, as the base image and copying our working directory's contents to a new directory on the image. We are instructing the image to expose the port `5000` when run as a container, on which we can access our app. Finally, our app container configures to run `python app.py` automatically when deployed to a pod.

Here, we build our image with the tag `<username>/<image-name>:<version>` format using the below command:

```shell
$ docker build -t ctgraves16/paymentapp-python:1.0.0 .
```

and then push it to the Docker Hub registry. It will be publicly available where anyone in the world can download and run it:

```shell
$ docker push ctgraves16/paymentapp-python:1.0.0
```

> 👉🏾 *NOTE: Ensure to replace ctgraves16 with your Docker Hub username.*

Now that we containerized the app, what about the database? How can we containerize that? We don't have to worry about it as we can easily use the official `mongo` Docker image and run it on the same network as the app container.

Run the below commands to test the image locally where it can be accessible at <http://localhost:5000/>

```shell
$ docker network create payment-app-net
$ docker run --name=mongo --rm -d --network=payment-app-net mongo
$ docker run --name=paymentapp-python --rm -p 5000:5000 -d --network=payment-app-net ctgraves16/paymentapp-python:1.0.0
```