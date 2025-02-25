# k8s-kvm

Upstream Kubernetes via KVM Vagrant and Ansible

# Project purpose

Create a "real" Kubernetes cluster on commodity hardware you might already have
at home or in the lab.  A lot of folks need something more substantial than than Minikube
ie distinct hosts for the control plane and worker nodes, but don't want to pay
for the cloud all day long when using the cluster a few times a month.

**Please note**: you must use wired connection (i.e. Ethernet) for the virtual network bridge. Wireless conenction (i.e. WiFi) will not work!

## About

KVM is highly efficient allowing for a "real" kubernetes cluster to run on a single machine given a decent bit of CPU and RAM

This installation is targeted for a bare metal install on a machine hooked into a lab or home network.  
The master and worker nodes will have ip addresses on your network. This will allow you to interact with it like its a "real" cluster.

The cluster can be destroyed and created again easily as the networking configuration, Vagrantfile, and Ansible playbook only need to be configured once.  

# Pre-install requirements

## Pick some hardware

A bare metal machine (the kind that will hurt you if you kick it barefoot).  Do yourself a favor and plug it
into a physical network too (using a cord).  

An old workstation with multiple sockets is great (I used an seven year old Z600).
A NUC with 32GB RAM will do as well.  

Make sure you have at least 4 cores (not threads) and preferably more than 16GB RAM.
(16 will work, but just run two worker nodes).  Used hardware is great for this project.

Lastly do a **fresh** OS install.  Tested with Ubuntu 18.04.

## Identify a block of contiguous free ip addresses

Kubernetes is IP address hungry, so make sure you have some on your network.

One is needed for the master and must end in 0.  Then one for each worker node starting with 1.  In the example I used 192.168.11.120 through 192.168.11.123 for a total of four addresses.

## Install dependencies

```bash
sudo apt install -qy qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker python3 python3-pip vagrant ansible
```

## Install Python virtualenv and requirements

For Ubuntu 24.04 and newer

```bash
sudo apt install -qy python3-virtualenv
```

For older Ubuntu versions

```bash
python3 -m pip install virtualenv
```

Then activate the virtualenv and install dependencies

```bash
python3 -m virtualenv .venv
source .venv/bin/activate
pip install -r requirments
```

## Create a bridge

**IMPORTANT**: Bridge will only work on wired connections. Bridge will NOT work on wireless network.

This will depend on the ubuntu distro and how you chose to set up networking.  
Call the bridge 'br0' if you can.  An example netplan configuration is

Open /etc/netplan/50-cloud-init.yaml with your favorite editor

```yml
network:
    ethernets:
        enp1s0:
            dhcp4: true
    bridges:
        br0:
            dhcp4: true
            interfaces:
                - enp1s0
    version: 2
 ```

 Now run

 ```bash
 netplan apply
 ```

 *Note this will likely result in the machine getting a new ip address.*

# Installation

KVM needs to run as root. Use sudo for vagrant commands, otherwise the VMs won't boot up.

## Change the network prefix to the free address block you chose

 Open Vagrantfile with your favorite text editor and edit the following line.  Make sure you choose a network block that's good on your network.

 ```ruby
 NETWORK_PREFIX="192.168.11.12"
 
 ```

## Edit alloted resources if desired

 Also in Vagrantfile.

 The number of worker nodes can be changed by editing the following line.  Total number of nodes will be the number of workers
 selected plus a master.  Two or three is recommended.

 ```ruby
 NUM_NODES = 3
 ```

You change the allotted number of CPU cores and memory.  Be sure not to overallocate.

```ruby
   config.vm.provider :libvirt do |libvirt|
    libvirt.cpus = 2
    libvirt.memory = 4096
```

You can also change the disk size. By default it is set to 15GB.

## Run Vagrant

```bash
sudo vagrant up
```

If you get errors regaring the image, use `download-ubuntu.sh` script to download the image manually and modify the `IMAGE_NAME` in `Vagrantfile`.

Your virtual machines should be created.  It may take several minutes to download the image the first time,
but it'll cache it in case you wish to rebuild the cluster.

*If* You get any screwy messages about image pool conflicts, try running:

```bash
virsh pool-destroy images
virsh pool-delete images
virsh pool-undefine images
```

You can stop virtual machines without destroying them with `sudo vagrant halt`.

## Configure hosts for ansible

Edit hosts file
Make sure it has the correct ip address for the master starting with the ip address block you chose

Each node is an incrment of 1 from the master.  Add or delete worker nodes as needed depending on how many you chose.

```conf
[kube-masters]
master1.kube.local ansible_host=192.168.11.120

[kube-nodes]
worker1.kube.local ansible_host=192.168.11.121
worker2.kube.local ansible_host=192.168.11.122
worker3.kube.local ansible_host=192.168.11.123

[ubuntu:children]
kube-masters
kube-nodes
```

## Run ansible

This step takes several minutes even on a powerful host.  Initializing a kubernetes cluster is a non-trivial task.

You can run the tasks seperately **recommended for first time**:

```bash
ansible-playbook -i hosts bootstrap.yml
ansible-playbook -i hosts kube-masters.yml
ansible-playbook -i hosts kube-workers.yml
```

Or run them all together

```bash
ansible-playbook -i hosts kubernetes.yml
```

**NOTE: you will have to type "yes" several times as prompted to accept the rsa fingerprint for ssh! Do it quick or its easy to get lost in the console.**

**NOTE: reboot step can sometimes get stuck. You should try running the same ansible runbook again.**

Output should look something like:

```bash
PLAY RECAP ***********************************************************************************************************************************************
master1.kube.local         : ok=24   changed=19   unreachable=0    failed=0
worker1.kube.local         : ok=19   changed=16   unreachable=0    failed=0
worker2.kube.local         : ok=19   changed=16   unreachable=0    failed=0
worker3.kube.local         : ok=19   changed=16   unreachable=0    failed=0
```

## SSH to the master node

```ssh kube@192.168.11.120```

Change the ip address to whatever you made the master.  The default username / password is kube / kubernetes

## Add kubernetes network plugin

 ```bash
 kubectl get nodes
 kubectl get all -A
 ```

 The nodes will show not ready until you install a network plugin.

 ```bash
 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.11.0/Documentation/kube-flannel.yml
 ```

## Smoke testing cluster

 ```bash
 kubectl get nodes
 kubectl create deployment nginx --image=nginx
 kubectl create service nodeport nginx --tcp=80:80
 kubectl describe service nginx
```

# Rebooting the cluster

Building the VM's with KVM and using a host bridge allows the cluster to be torn
down and brought back up quite easily.  This is great when you want to start fresh.

## Kill the old cluster

Destroy VM's through Vagrant
```sudo vagrant destroy```

Kill the host ssh key fingerprint cache
```rm ~/.ssh/known_hosts```

## Make the cluster again

Run the vagrant up command

Run ansible again

Install Kubernetes networking

# Housekeeping items

## Setup kubectl on your machine

Grab the config and keys from the .kube/config file in the kube user's home directory
on the master.  Add them to the .kube/config file on your machine with kubectl.

### Check the status of your VM's

```bash
root@z600:~/k8s-kvm# virsh list --all
 Id    Name                           State
----------------------------------------------------
 1     k8s-kvm_worker-3               running
 2     k8s-kvm_worker-1               running
 3     k8s-kvm_worker-2               running
 4     k8s-kvm_master1                running

```

# Next Steps

You can start with [kubernetes basics tutorial course](https://kubernetes.io/docs/tutorials/kubernetes-basics/). You can safely ignore minikube commands, otherwise the tutorial should work just fine on this cluster.

# Improvement Ideas

1. `apt upgrade` step takes a long time and a lot of network traffic. Use packer to pre-create vagrant image.
1. Add a script to run all commands (python setup, install dependensies, run vagrant, run the runbooks)
1. Port to Ubuntu 24.04. Please note that generic/ubuntu2204 is the last official Ubuntu vagrant box
