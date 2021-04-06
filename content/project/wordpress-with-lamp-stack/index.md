---
title: WordPress with LAMP stack
subtitle: This Playbook will install a WordPress Content Management System (CMS)
  on top of a LAMP environment (Linux, Apache, MySQL, and PHP) on two remote
  servers in a private network.
date: 2021-04-06T07:05:39.631Z
draft: false
featured: false
authors:
  - Chaance
tags:
  - Linux
  - Wordpress
  - Ansible
  - LAMP
categories:
  - CM-Tools
external_link: https://github.com/ctg123/wordpress-ansible/
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
This Playbook will install a WordPress Content Management System (CMS) on top of a LAMP environment (Linux, Apache, MySQL, and PHP) on two remote servers in a private network. The LAMP versioning highlights the following for each layer:

* **Linux** - Ubuntu 18.04 ( 1 Virtual machine designated as the master node and two managed nodes for hosting the WordPress CMS). Vagrant and VirtualBox create these machines.
* **Apache2** - The Apache HTTP server is the most widely-used web server in the world. It provides many powerful features, including dynamically loadable modules, robust media support, and extensive integration with other popular software.
* **MySQL 5.7** - MySQL is the world’s most popular open-source relational database management system.
* **PHP 7.4** - PHP is a popular general-purpose scripting language that is especially suited to web development.

## Create Vagrant private network

Creating a robust sandbox environment for rapid prototyping eliminates the risk of crashing or breaking other functions when using servers or your local machine when purposed for other vital tasks. We’ll create a private network that you can install on any local device (i.e., laptop) as long as VirtualBox is available via Vagrant. The `vagrantfile` seen below can build test virtual machines (VM’s) for our Ansible playbook testing environment. If you prefer using a public cloud such as AWS, Azure, GCP, or Digital Ocean, the logical design is easy to follow.

```ruby
###############
# Vagrantfile #
###############

# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.

Vagrant.configure("2") do |config|
  config.vm.define "ansible-master" do |vm1|
    vm1.vm.box = "bento/ubuntu-18.04"
    vm1.vm.hostname = "ansible-master"
    vm1.vm.network "private_network", ip: "10.23.45.10"

    config.vm.provider "virtualbox" do |vb|
	  vb.gui = false
	  vb.memory = "4096"
      vb.cpus = "2"
	  vb.customize ['modifyvm', :id, '--cableconnected1', 'on']
    end
	config.vm.provision "shell", run: "always", inline: <<-SHELL
		echo "Welcome to the Ubuntu Ansible network."
	SHELL
  end

  config.vm.define "ansible-node1" do |vm2|
    vm2.vm.box = "bento/ubuntu-18.04"
    vm2.vm.hostname = "ansible-node1"
    vm2.vm.network "private_network", ip: "10.23.45.20"

    config.vm.provider "virtualbox" do |vb|
	  vb.gui = false
	  vb.memory = "2048"
      vb.cpus = "2"
	  vb.customize ['modifyvm', :id, '--cableconnected1', 'on']
    end
	config.vm.provision "shell", run: "always", inline: <<-SHELL
		echo "Welcome to the Ubuntu Ansible network."
	SHELL
  end
  
  config.vm.define "ansible-node2" do |vm3|
    vm3.vm.box = "bento/ubuntu-18.04"
    vm3.vm.hostname = "ansible-node2"
    vm3.vm.network "private_network", ip: "10.23.45.30"

    config.vm.provider "virtualbox" do |vb|
	  vb.gui = false
	  vb.memory = "2048"
      vb.cpus = "2"
	  vb.customize ['modifyvm', :id, '--cableconnected1', 'on']
    end
	config.vm.provision "shell", run: "always", inline: <<-SHELL
		echo "Welcome to the Ubuntu Ansible network."
	SHELL
  end
end
```

## Configure the Master node and SSH connection

To begin using Ansible to manage your server infrastructure, you need to install the Ansible software on the machine that will serve as the Ansible master node. First, connect via SSH to the virtual machine.

```powershell
PS C:\..\vagrant\ubuntu_ansible> vagrant ssh ansible-master
```

By default, `Vagrant` will be the user on the machine; however, an `admin` account with sudo privileges creates the specific purpose of using Ansible to talk to the other virtual machines in the network. Use the `adduser` command as the root user first to add a new user to your system:
There will be a prompt to enter a password of your choice. 
```shell
vagrant@ansible-master:~$ sudo -i
root@ansible-master:~# adduser admin
```
Use the `usermod` command to add the user to the sudo group and test if the sudo commands work when logged into the user account.
```shell
root@ansible-master:~# usermod -aG sudo admin
root@ansible-master:~# su - admin

admin@ansible-master:~$ sudo apt-get update
```
Run the following command to include the official project’s PPA (personal package archive) in your system’s list of sources:
```shell
admin@ansible-master:~$ sudo apt-add-repository ppa:ansible/ansible
```
Press `ENTER` when prompted to accept the PPA addition.
Next, refresh your system’s package index to be aware of the packages available in the newly included PPA and then proceed to install.
```shell
admin@ansible-master:~$ sudo apt-get update
admin@ansible-master:~$ sudo apt install ansible
```
### Configure password-less authentication
We will set up password-less authentication for our admin user from master to all the managed nodes by generating a public-private key pair using `ssh-keygen`. We have pre-defined a blank password using `-P “”` This step will create private and public key pair located in the `~/.ssh` directory.
```shell
admin@ansible-master:~$ ls -al ~/.ssh/
total 20
drwx------ 2 admin admin 4096 Apr  3 08:03 .
drwxr-xr-x 6 admin admin 4096 Apr  4 22:57 ..
-rw------- 1 admin admin 1675 Apr  3 07:49 id_rsa
-rw-r--r-- 1 admin admin  402 Apr  3 07:49 id_rsa.pub
-rw-r--r-- 1 admin admin  444 Apr  3 07:48 known_hosts
```
We will use `ssh-copy-id` to copy the keys to the remote managed server and add it to `authorized_keys`.
```shell
admin@ansible-master:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@10.23.45.20
admin@ansible-master:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@10.23.45.20
```
### SSH and Admin user setup with the Setup playbook
The `setup_ubuntu1804` folder in the Github repository runs an independent playbook that will execute an initial server setup for the managed nodes. The options stores in the `vars/default.yml` variable file. We define the following setting below:
- `create_user`: The name of the remote sudo user to create. In our case, it will be admin.
- `copy_local_key`: Path to a local SSH public key that will be copied as an authorized key for the new user. By default, it copies the key from the current system user running Ansible.
- `sys_packages`: An array with a list of fundamental packages to be installed.

Run the playbook with the following commands. 
```shell
admin@ansible-master:~/.../setup_ubuntu1804$ ansible-playbook -i inventory -u admin playbook.yml
```