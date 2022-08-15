Tired of paying through the nose for a cloud kubernetes cluster. Fancy mining bitcoin with your own server rack. How about creating a **[Raspberry Pi Kubernetes Cluster](https://www.devopswiki.co.uk/wiki/kubernetes/pi/raspberry-pi-kubernetes-cluster)**

# Use Ansible to Install Kubernetes on a Raspberry Pi Cluster or Vagrant VMs

This Ansible configuration manager will create a Kubernetes cluster on cloud VMs, vagrant VMs or even a Raspberry Pi cluster.


---


## step 1 - install ansible and setup nodes

We need to prepare our nodes, install Ansible and configure **`/etc/hosts`** entries so that machines can talk to one another.

1. build your Raspberry Pi rack and install Ubuntu Server 20.04 on each node
1. or install Vagrant and use it to create 3 or 4 VMs running Ubuntu Server
1. install Ansible on a Linux laptop (or VM)
1. ensure every node can contact every other node by name with **`/etc/hosts`** entries
1. you need **`/etc/hosts`** entries on your Linux laptop/VM too mapping IP addresses


---


## step 2 - setup ssh from ansible host

Ansible needs to be able to SSH into each node to configure it with whatever is needed for that node to play the role of a Kubernetes master or worker. To do this

1. create SSH private / public key pairs on your Ansible laptop
1. put the private keys in the laptop's .ssh folder (eg **`~/.ssh/cluster.pi-r1d4.pem`**)
1. use this command put the corresponding public keys into the node
1. **`echo "<public key text>" >> ~/.ssh/authorized_keys`**
1. create a section in your **`~/.ssh/config`** file like below

### The SSH Config File

Put this section inside the **`~/.ssh/config**` file to help Ansible configure the kubernetes cluster machines.

```
## ###################################### ##
## Rack 1 Raspberry Pi Kubernetes Cluster ##
## -------------------------------------- ##
## ###################################### ##

Host master
  HostName pi-r1d1
  User ubuntu
  IdentityFile ~/.ssh/cluster.pi-r1d1.pem
  StrictHostKeyChecking no

Host worker1
  HostName pi-r1d2
  User ubuntu
  IdentityFile ~/.ssh/cluster.pi-r1d2.pem
  StrictHostKeyChecking no

Host worker2
  HostName pi-r1d3
  User ubuntu
  IdentityFile ~/.ssh/cluster.pi-r1d3.pem
  StrictHostKeyChecking no

Host worker3
  HostName pi-r1d4
  User ubuntu
  IdentityFile ~/.ssh/cluster.pi-r1d4.pem
  StrictHostKeyChecking no
```

In each block of the **`~/.ssh/config`** file you are pinpointing
- the name (**`master`**, **`worker1`**) that refers to the host
- the actual hostname of the node (set with **`hostnamectl`**)
- the username (**`ubuntu`** is the default for Ubuntu server 20.04)
- the path to the private key placed in the **`~/.ssh`** folder


---


## step 3 - validate the ssh configuration

If you are setting up a 4 node cluster (on Raspberry Pi or Vagrant VMs) you should be able to **ssh from** your linux laptop (with Ansible installed) **into each node** like this

- **`ssh master`**
- **`ssh worker1`**
- **`ssh worker2`**
- **`ssh worker3`**

These names match the entries in the **`~/.ssh/config`** file. 


---


## step 4 - the `hosts.ini` ansislbe inventory

If you work hard and configure SSH **Ansible rewards your efforts** and automates the entire Kubernetes cluster configuration. In this repository the hosts.ini file looks like this.

```
master

[worker]
worker1
worker2
worker3
```

It's really simple. The names **`master`**, **`worker1`** and so on reflect the names you put into your laptop's **`~/.ssh/config`** file.


---


## step 5 - ansible pre-flight checks

Run through this checklist before you let Ansible take-off and create your kubernetes cluster.

- **`ansible -m ping all -i hosts.ini`** # can Ansible talk to each node
- **`ansible-playbook -i hosts.ini --syntax-check playbook-base-install.yml`**
- **`ansible-playbook -i hosts.ini --list-hosts playbook-base-install.yml`**
- **`ansible-playbook -i hosts.ini --list-tasks playbook-base-install.yml`**

If you change any playbook you can syntax check it and look at the hosts that will be engaged with the **`--list-hosts`** flag.

The output of the command with **`--list-hosts`** shows that the **hosts pattern**

- **`all`** - will execute on all 4 machines
- **`master`** - will execute on the master machine
- **`worker`** - will execute on the 3 workers


---


## step 6 - create your kubernetes cluster

Creating the kubernetes cluster is about running three commands. There is **no need to change** the **`hosts.ini`** file because the playbooks know which things to do to all nodes, or the msater, or the worker or a combination.

- **`ansible-playbook -i hosts.ini playbook-base-install.yml`**
- **`ansible-playbook -i hosts.ini playbook-master-setup.yml`**
- **`ansible-playbook -i hosts.ini playbook-worker-joins.yml`**

### base install playbook

This playbook prepares every node to be part of a kubernetes cluster no matter whether they are masters or workers. It is idempotent can be ran multiple times without a revert or reset playbook.

### master setup playbook

The master setup playbook **is also a laptop setup** playbook. It is responsible for

- running **`kubeadm init`** on the node that will be master
- creating kubectl configuration on the master node
- creating kubectl configuration on your Linux laptop (running Ansible)
- installing Calico pod networking necessary for pod communication

Once the playbook completes you can examine your master from **both** the Linux laptop and when you ssh into the master node.

- **`kubectl get nodes -o wide`**
- **`kubectl get pods --all-namespaces`**

This playbook leaves the kubeadm output logs in a file called **`kubeadm-init-output.log`** and for high availability setups it includes a command you can use to join multiple masters (control plane) to the cluster.

### worker joins playbook

This playbook picks up a join command from the master and applies it to each worker. Use the **`kubectl get nodes -o wide`** command to verify the worker has joined.

You can also ssh into each node and run **`watch docker ps -a`** to see what container workloads are executing on that node.


---


## step 7 - deploy and scale `nginx`

You can use kubectl from your linux laptop (or VM) to interact with your cluster. Let's deploy nginx and then scale it up to run many pods on each machine.

- **`kubectl create deployment nginx --image=nginx`**
- **`kubectl get pods -o wide`**
- **`kubectl create service nodeport nginx --tcp=80:80`**
- **`kubectl get svc -o wide`**

```
NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
nginx-6799fc88d8-wsnj4   1/1     Running   0          49s   172.16.148.129   pi-r1d2   <none>           <none>
```


```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE     SELECTOR
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        3h59m   <none>
nginx        NodePort    10.97.23.211   <none>        80:32107/TCP   11s     app=nginx
```

For quick validation we bound the service to a port on the node. In this case the node is **`pi-r1d2`** and the port is **`32107`** so this is the url.

```
http://pi-r1d2:32107/
```

There it is. The ubiquitous **`Welcome to nginx!`** splash page. Don't forget to scale up the number of nginx pods.

- **`kubectl scale deployments/nginx --replicas=4`**
- **`kubectl get pods -o wide`**


---


## Summary

**Now your `kubernetes cluster` is ready for lots of learning, validating, software development, bitcoin mining, problem solving, model training and more besides.**


## Appendix A | Use Ansible to update /etc/hosts database

**Kubernetes nodes all need to be able to talk to each other, not just with the master.**

Ansible can add **`/etc/hosts`** entries for every host and if outdated entries already exist it can ammend them. Even better it does not add duplicate entries.

```
---
- name: Add IP address of all hosts to all hosts
  lineinfile:
    dest: /etc/hosts
    regexp: '.*{{ item }}$'
    line: "{{ hostvars[item].ansible_host }} {{item}}"
    state: present
  when: hostvars[item].ansible_host is defined
  with_items: "{{ groups.all }}"
```

If you are not running a DNS server but have a number of hosts to manage alongside a **fickle DHCP allocator**, the above Ansible code can be a godsend.


---


## Appendix B | Finding Docker Versions

Sometimes Kubernetes (during **`kubeadm init`**) declares incompatibility with the **docker version** installed on the nodes.

At the time of writing Kubernetes refused to use **`docker 20.10`** so we needed to specify the docker version like so **`docker-ce=5:19.03.15~3-0~ubuntu-focal`** for **Ubuntu 20.04** Focal Fossa. The steps are to

- **`sudo apt-cache showpkg docker-ce`** - discover available versions
- replace the version tag so that Ansible installs the correct one

But now you have to remove the already present docker and kubernetes packages.

- **`docker system prune --all --force`** - remove all containers and images
- **`sudo apt-get remove docker-ce --assume-yes`** - remove docker-ce
- **`sudo apt autoremove docker-ce --assume-yes`** - and again
- **`sudo apt-get remove docker-ce-cli --assume-yes`** - remove docker-ce-cli
- **`sudo apt autoremove docker-ce-cli --assume-yes`** - and again

It may also pay to backpedal and remove the kubernetes packages. This is how.

- **`sudo apt-get remove kubectl --assume-yes`**
- **`sudo apt autoremove kubectl --assume-yes`**
- **`sudo apt-get remove kubelet --assume-yes`**
- **`sudo apt autoremove kubelet --assume-yes`**
- **`sudo apt-get remove kubeadm --assume-yes`**
- **`sudo apt autoremove kubeadm --assume-yes`**

Note that there is no need to install **`containerd.io`** separately as it is already included within docker.
