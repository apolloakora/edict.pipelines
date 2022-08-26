
# Concourse CI/CD in Kubernetes on Ubuntu 22.04

This blog starts with a number of freshly installed machines running Ubuntu. You could use Cloud VM, Vagrant VMs, Intel NUCs or even (arm64) **[Raspberry Pi](https://www.devopswiki.co.uk/wiki/kubernetes/pi/raspberry-pi-kubernetes-cluster)** nodes.


---


## 1. Initial Basic Machine Setup

When the VM awakens login with username - **`ubuntu`** and your chosen password. Then proceed with

1. **`lsb_release -a`** - check the linux distroy and version
1. **`sudo apt update`** - update the list of package repositories
1. **`sudo dpkg --configure -a`** - get the package manager ready
1. **`sudo apt upgrade --assume-yes`** - upgrade the system packages
1. **`sudo apt install emacs --assume-yes`** - install emacs editor
1. **`ip a`** - record the IP address of this pi

Now SSH in from your "pet" machine to continue the setup. If you have connected to this machine before you may need to delete a line from known_hosts and maybe change the IP address in **`~/.ssh/config`** and **`/etc/hosts`**


---


## 2. Set Up Passwordless Sudo

After these commands you won't need to type the password every time you run a command with sudo. To achieve this we modify the user and then assume root rather than just using sudo.

```bash
sudo usermod -a -G sudo $USER
sudo su root
cd;
sudo echo "$SUDO_USER ALL=NOPASSWD: ALL" >> /etc/sudoers
```

In Ubuntu 22.04 you no longer need to exit and pull up a new terminal.


---


## 3. (Optional) Change Machine Hostname

Skip this section if your machines already have distinct hostnames. **A network with 8 machines called `ubuntu` is disconcerting!** Set the hostname to uniquely identify the rack and the position in the rack where **1 is high** and **4 is low**.

<blockquote>
pi-r1d1 is the hostname of the top machine in rack one<br/>
pi-r2d8 the eighth (bottom) machine in the second rack
</blockquote>

1. **`hostnamectl`** - report the current hostname and machine ID
1. **`sudo hostnamectl set-hostname pi-r1d1`** - change the hostname
1. **`sudo emacs /etc/hosts`** - edit the `/etc/hosts` file
1. add this **`127.0.0.1 pi-r1d1`** as the second line

### Edit the cloud configuration

Now edit the cloud configuration file and shutdown the machine

1. **`sudo emacs /etc/cloud/cloud.cfg`** - edit the cloud config
1. `preserve_hostname: true` - change flag from false to true
1. **`sudo shutdown -r now`** - reboot the cluster node

Now on your laptop (pet) add a mapping in **`/etc/hosts`** linking the pi's IP address and its new hostname. This method doesn't scale - if you run more than 5 or 6 machines you should consider setting up a dedicated DNS server on one of your cluster nodes.

Test connectivity with **`ping -c 4 pi-r1d1`**


---


## 4. Add /etc/hosts mappings for each machine

**Note that you may need to return to this configuration when you have the ip address/hostnames of the other raspberry pis.**

<blockquote>
As well as the 127.0.0.1 mapping add the IP address/hostname mappings for all the other Raspberry Pi machines.
Every machine needs to talk to every other machine.
</blockquote>

1. **`sudo emacs /etc/hosts`** - edit the /etc/hosts file
1. Add the Address Hostname mappings for other cluster nodes
3. Save and Exit

```
127.0.0.1 localhost
127.0.0.1 pi-r1d1

# Address Hostname mappings for K8s Cluster Nodes
192.168.0.59 pi-r1d2
192.168.0.61 pi-r1d3
192.168.0.62 pi-r1d4
```

Check that your new node can talk to all the other nodes.

```
ping pi-r1d2
ping pi-r1d3
ping pi-r1d4
```


---


## 5. Setup SSH Public/Private Keypair

**Ansible** the configuration management tool will be used to setup our Kubernetes cluster. Ansible requires a private key and needs our Raspberry Pi machines to have the corresponding authorized public key.

Use **`ssh-keygen`** or **`safe keygen`** to produce your keypair.

1. **`ssh ubuntu@pi-r1d1`** - ssh in with password
1. **`echo "<public key text>" >> ~/.ssh/authorized_keys`** - add public key
1. **`cat ~/.ssh/authorized_keys`** - assert public key in authorized_keys
1. **`exit`** - exit the ssh session

Finally we place the private key equivalent inside the .ssh folder and connect **securely without a password**.

1. **`safe write private.key --folder=~/.ssh`** - write the private key into **`~/.ssh`**
1. **`ssh ubuntu@pi-r1d1 -i ~/.ssh/services.cluster.pi-r1d1.pem`** - ssh in securely


---


## 6. Setup Simple Secure SSH Access

For humans and Ansible to **simply ssh into each cluster node** you need the following sections within the ssh config file found at **`~/.ssh/config`** on your pet Linux/Mac laptop that Ansible will run from.

```
## ################################### ##
## Kubernetes Cluster Nodes SSH Config ##
## ----------------------------------- ##
## ################################### ##

Host master
  HostName pi-r1d1
  User ubuntu
  IdentityFile ~/.ssh/services.cluster.pi-r1d1.pem
  StrictHostKeyChecking no

Host worker1
  HostName pi-r1d2
  User ubuntu
  IdentityFile ~/.ssh/services.cluster.pi-r1d2.pem
  StrictHostKeyChecking no

Host worker2
  HostName pi-r1d3
  User ubuntu
  IdentityFile ~/.ssh/services.cluster.pi-r1d3.pem
  StrictHostKeyChecking no

Host worker3
  HostName pi-r1d4
  User ubuntu
  IdentityFile ~/.ssh/services.cluster.pi-r1d4.pem
  StrictHostKeyChecking no
```

In each block of the **`~/.ssh/config`** file you are pinpointing

- the name (**`master`**, **`worker1`**) that refers to the host
- the actual hostname of the node (set with **`hostnamectl`**)
- the username (**`ubuntu`** is the default for Ubuntu server 20.04)
- the path to the private key placed in the **`~/.ssh`** folder

Test host and ssh connectivity like this.

```
ssh master
ssh worker1
ssh worker2
ssh worker3
```

When debugging SSH connectivity failures remember

- to delete the hostname line in authorized_keys on your **(pet)** laptop
- to delete the host line in known_hosts on the **(cattle)** node
- that if using DHCP your router may assign different IP addresses
- check /etc/hosts has correct IP Address hostname mapping
- check your pet's ssh config has the correct username, hostname and key file


---


## 7. Install Ansible on your (Pet) Laptop

Ansible is built on an agentless paradigm so you only need to install it in one place then point at the nodes you want it to configure.

### Install Ansible on Mac

If you are using a MacBook it is simple to install Ansible.

```
brew install ansible
ansible --version
```

### Install Ansible on Linux (Ubuntu)

If you have an Ubuntu VM (Raspberry Pi, or laptop, or built by Vagrant) you can install Ansible with these commands.

```
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible
ansible --version
```


---


## 8. Configure `hosts.ini` - Ansislbe's Inventory

If you work hard and configure SSH **Ansible rewards your efforts** and automates the entire Kubernetes cluster build creation and management. In this repository the hosts.ini file looks like this.

```
[master]
node1

[worker]
node2
node3
node4
```

It's really simple. The names **`master`**, **`worker1`** and so on reflect the names you put into your laptop's **`~/.ssh/config`** file.


---


## 9. ansible pre-flight checks

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


## 10. create kubernetes cluster with ansible

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


## 11. setup k9s kubernetes explorer

Learning kubectl and kubernetes manifests is important - but k9s for navigating and managing clusters makes you highly productive.

So to setup k9s you
- **`cd ~/.kube; ls -lah`**               # change dir to the .kube folder
- **`mv config $(date '+%s-%F')-config`** # backup the present config file
- **`scp <k8s_host>:~/.kube/config .`**   # overwrite with new kube config
- [install k9s on Mac Windows and Linux](https://www.devopswiki.co.uk/k9s/k9s)


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

## 13. Install Concourse Using Helm

*To install concourse we have to begin by installing helm*

- **`curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null`**
- **`sudo apt-get install apt-transport-https --yes`**
- **`echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list`**
- **`sudo apt-get update`**
- **`sudo apt-get install helm`**

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
