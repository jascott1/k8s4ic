# K8s for Intel Cloud

## Overview 

This document covers installing Kuberenets on Intel Cloud VMs. At a high level we will need to accomplish the following: 

1. Create VMs - a "deploy" node, a master node and two minion nodes
3. Install pre-requisites on deploy node
4. Run playbooks to pre-configure all the nodes.
5. Install k8s
6. Install k8s add-ons (dns and dashboard)
7. Test

## Create VMs

### Tips

#### Use a naming convention.

*Example: ops-deploy01, ops-km01, ops-kn01, ops-kn02*

#### Flavor size

4x Small-A flavor VMs are sufficient for testing but YMMV depending on intended purpose.

#### Image

Ubuntu 14.04 

### Create Deploy VM

Create a VM thru Intel Cloud and name it <cluster-prefix>-deploy01

*Example: ops-deploy01*

### Create Master and Minion VMs

Create 3 VMs following your naming convention. This example uses km<number> for master and kn<number> for minions.

*Example: ops-km01, ops-kn01, ops-kn02*

## Install pre-requisites on deploy node

### Configure proxy

    sudo /etc/environment

Add the following contents, substituting the correct node IPs:

    http_proxy="http://proxy-us.intel.com:911"
    https_proxy="http://proxy-us.intel.com:911"
    no_proxy=".intel.com,localhost,127.0.0.1,<deploy ip>,<master ip>,<minion1 ip>,<minion2 ip>

### Install ansible and passlib

    # on deploy node
    sudo apt-get update
    sudo apt-get install -y software-properties-common python-passlib
    sudo apt-add-repository ppa:ansible/ansible
    sudo apt-get update
    sudo apt-get install -y ansible
    
### Create k8s user

Ansible roles are currenly wired to create and utilize user "k8s" on all master and minion nodes and expect to find id_rsa.pub in /home/k8s/.ssh on the deploy node.

    # on deploy node
    sudo adduser k8s
    sudo usermod -aG sudo k8s
    sudo su k8s
    cd ~
    ssh-keygen # accept defaults

Add your public key to k8s user

    vi /home/k8s/.ssh/authorized_keys

### Update /etc/hosts

Append your nodes and IP addresses to the hosts file. This makes it easier to ssh to nodes.

    sudo vi /etc/hosts
    
## Use ansible to configure nodes

Before we can install Kubernetes, we must run Ansible playbooks to setup things like proxy, user and docker.

### Clone k8s4ic repo

    git clone https://github.com/jascott1/k8s4ic.git
    cd k8s4ic
    
### Create Ansible inventory hosts

Creat a hosts file in the k8s4ic directory:

    vi hosts
    
Add the following contents using your hostnames and IPs.   
IMPORTANT: Change the ad_ user to the account that created the VMs (or at least has SSH access and sudo)

    [masters]
    ops-km01 ansible_host=10.64.136.100 ansible_user=ad_myidsid
    
    [masters:vars]
    ansible_ssh_private_key_file=/home/k8s/.ssh/id_rsa
    
    [workers]
    ops-kn01 ansible_host=10.64.136.101
    ops-kn02 ansible_host=10.64.143.102
    
    [workers:vars]
    ansible_ssh_private_key_file=/home/k8s/.ssh/id_rsa
    ansible_user=ad_myidsid

### SSH Fingerprint

From deploy node, SSH to each node and accept the fingerprint.

    ssh ops-km01
    ssh ops-kn01
    ssh ops-kn02

IMPORTANT: If you re-create master and minions, be sure to clear ~/.ssh/known_hosts or Ansible will fail to connect.

TODO automate ssh fingerprint
    
### Run master play   

Run the master playbook and enter a password to be used for the new user as well as your ad_ password twice.
    
    ansible-playbook -i hosts playbooks/master.yml --limit=masters --ask-pass --ask-become-pass
    
### Run worker play

Run the worker playbook and enter a password to be used for the new user (same as before) as well as your ad_ password twice.

    ansible-playbook -i hosts playbooks/worker.yml --limit=workers  --ask-pass --ask-become-pass
    
## Deploy Kubernetes    
    
### Clone Kubernets

    git clone https://github.com/kubernetes/kubernetes.git

### Configure Kubernetes

We must add our nodes and proxy to the config.

    cd kubernetes
    vi cluster/ubuntu/config.default.sh

Find the line similar to the one below and change for your user and nodes

    export nodes=${nodes:-"k8s@10.64.136.104 k8s@10.64.136.111 k8s@10.64.143.105"}

Find the line similar to the one belowand change it to have dedicated server and two minions.

    roles=${roles:-"a i i"}
    
Find the line similar to the one below and add the proxy.

IMPORTANT: Add all the nodes to the no_proxy var.

    PROXY_SETTING=${PROXY_SETTING:-"http_proxy=http://proxy-us.intel.com:911 https_proxy=http://proxy-us.intel.com:911 no_proxy=.intel.com,localhost,10.64.143.105,10.64.136.100,10.64.136.101,10.64.142.102"}

#### Patch bug: Missing untar and copy for salt 

The download-release script is missing an untar and copy commmand and kube-up.sh will fail unless the script is modified.

##### add untar

In cluster/ubuntu/download-release.sh add  
'tar xzf kubernetes-salt.tar.gz' as below:

    ...
    tar xzf kubernetes-server-linux-amd64.tar.gz
    tar xzf kubernetes-salt.tar.gz
    popd
    ...

##### add copy

and add 'cp -a kubernetes/server/kubernetes/saltbase ../' as below:

    ...
    cp kubernetes/server/kubernetes/server/bin/kubectl binaries/
    cp -a kubernetes/server/kubernetes/saltbase ../
    echo ${KUBE_VERSION} > binaries/.kubernetes
    ...



### Install Kubernetes

    cd kubernetes/cluster
    KUBERNETES_PROVIDER=ubuntu ./kube-up.sh



## Deploy Addons (DNS and UI)  

    cd ubuntu
    KUBERNETES_PROVIDER=ubuntu ./deployAddons.sh


## Reference

### Ubuntu Bar metal tutorial

http://kubernetes.io/docs/getting-started-guides/ubuntu/



    
