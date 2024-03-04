# How to Install RKE Cluster

## First you should Install kubectl and RKE (Rancher Kubernetes Engine) and Ansible

#### kubectl:
- curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
- chmod +x ./kubectl
- sudo mv ./kubectl /usr/local/bin/kubectl
- kubectl version --client

#### RKE:
- curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
- chmod +x rke_linux-amd64
- sudo mv rke_linux-amd64 /usr/local/bin/rke
- rke --version

#### Ansible:
- sudo apt install python3-pip python3-venv
- mkdir .venv   ( in home directory , not in project folder )
- python3 -m venv ~/.venv/ansible ( or any other name )   ( if we use python3 -m venv --copies ~/.venv/ansible , the python will not be symbolic link to python of system , it will copy all the env )
- source the env ( what is the difference between run a bash script in a regular way or source it ? if we source a bash script , it will run in the bash that it exists , we can source with . filename.sh ) ==we source the env with source ~/.venv/ansible/bin/activate==
- pip install ansible

## Update your Linux System
- sudo apt update -y
- sudo apt upgrade -y

#### and its better to reboot the system after Updating

## Enable required Kernel modules:
#### Create a playbook with below contents and run it against your RKE servers inventory.



```YAML
---
- name: Load RKE kernel modules
  hosts: rke-hosts
  remote_user: root
  vars:
    kernel_modules:
      - br_netfilter
      - ip6_udp_tunnel
      - ip_set
      - ip_set_hash_ip
      - ip_set_hash_net
      - iptable_filter
      - iptable_nat
      - iptable_mangle
      - iptable_raw
      - nf_conntrack_netlink
      - nf_conntrack
      - nf_conntrack_ipv4
      - nf_defrag_ipv4
      - nf_nat
      - nf_nat_ipv4
      - nf_nat_masquerade_ipv4
      - nfnetlink
      - udp_tunnel
      - veth
      - vxlan
      - x_tables
      - xt_addrtype
      - xt_conntrack
      - xt_comment
      - xt_mark
      - xt_multiport
      - xt_nat
      - xt_recent
      - xt_set
      - xt_statistic
      - xt_tcpudp

  tasks:
    - name: Load kernel modules for RKE
      modprobe:
        name: "{{ item }}"
        state: present
      with_items: "{{ kernel_modules }}"
 
```
#### The manual way
````bash
for module in br_netfilter ip6_udp_tunnel ip_set ip_set_hash_ip ip_set_hash_net iptable_filter iptable_nat iptable_mangle iptable_raw nf_conntrack_netlink nf_conntrack nf_conntrack_ipv4   nf_defrag_ipv4 nf_nat nf_nat_ipv4 nf_nat_masquerade_ipv4 nfnetlink udp_tunnel veth vxlan x_tables xt_addrtype xt_conntrack xt_comment xt_mark xt_multiport xt_nat xt_recent xt_set  xt_statistic xt_tcpudp;
     do
       if ! lsmod | grep -q $module; then
         echo "module $module is not present";
         sudo modprobe $module
       fi;
done
````

## Disable swap and Modify sysctl entries
#### The recommendation of Kubernetes is to disable swap and add some sysctl values.

```YAML
---
- name: Disable swap and load kernel modules
  hosts: rke-hosts
  remote_user: root
  tasks:
    - name: Disable SWAP since kubernetes can't work with swap enabled (1/2)
      shell: |
        swapoff -a
     
    - name: Disable SWAP in fstab since kubernetes can't work with swap enabled (2/2)
      replace:
        path: /etc/fstab
        regexp: '^([^#].*?\sswap\s+.*)$'
        replace: '# \1'
    - name: Modify sysctl entries
      sysctl:
        name: '{{ item.key }}'
        value: '{{ item.value }}'
        sysctl_set: yes
        state: present
        reload: yes
      with_items:
        - {key: net.bridge.bridge-nf-call-ip6tables, value: 1}
        - {key: net.bridge.bridge-nf-call-iptables,  value: 1}
        - {key: net.ipv4.ip_forward,  value: 1}
```

#### Manually
##### Swap:
````bash
$ $ sudo vim /etc/fstab
# Add comment to swap line

$ sudo swapoff -a
````
##### Sysctl:
````bash
$ sudo tee -a /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

$ sudo sysctl --system
````


## Install Docker
#### Installing Docker due to [Official Docker Installation](https://docs.docker.com/engine/install/ubuntu/)

## Allow SSH TCP Forwarding
#### You need to enable your SSH server system-wide TCP forwarding.
- Open ssh configuration file located at /etc/ssh/sshd_config:
```bash
$ sudo vi /etc/ssh/sshd_config
AllowTcpForwarding yes
```
#### Restart ssh service after making the change.
- sudo systemctl restart ssh

## Generate RKE cluster configuration file.

#### RKE uses a cluster configuration file, referred to as cluster.yml to determine what nodes will be in the cluster and how to deploy Kubernetes.
#### There are many configuration options that can be set in the cluster.yml. This file can be created from  minimal example  templates or generated with the rke config command.

#### Run rke config command to create a new cluster.yml in your current directory :
```bash
rke config --name cluster.yml
```
#### This command will prompt you for all the information needed to build a cluster.

#### If you want to create an empty template cluster.yml file instead, specify the --empty flag.
```bash
rke config --empty --name cluster.yml
```

## Example of a RKE Cluster Configuration (3 Master Nodes – etcd and control plane ( 3 for HA) and 2 Worker nodes)
## just use it as reference to create your own configuration.
````yaml
nodes:
- address: 10.10.1.10
  internal_address:
  hostname_override: rke-master-01
  role: [controlplane, etcd]
  user: rke
- address: 10.10.1.11
  internal_address:
  hostname_override: rke-master-02
  role: [controlplane, etcd]
  user: rke
- address: 10.10.1.12
  internal_address:
  hostname_override: rke-master-03
  role: [controlplane, etcd]
  user: rke
- address: 10.10.1.13
  internal_address:
  hostname_override: rke-worker-01
  role: [worker]
  user: rke
- address: 10.10.1.114
  internal_address:
  hostname_override: rke-worker-02
  role: [worker]
  user: rke

# using a local ssh agent 
# Using SSH private key with a passphrase - eval `ssh-agent -s` && ssh-add
ssh_agent_auth: true

#  SSH key that access all hosts in your cluster
ssh_key_path: ~/.ssh/id_rsa

# By default, the name of your cluster will be local
# Set different Cluster name
cluster_name: rke

# Fail for Docker version not supported by Kubernetes
ignore_docker_version: false

# prefix_path: /opt/custom_path

# Set kubernetes version to install: https://rancher.com/docs/rke/latest/en/upgrades/#listing-supported-kubernetes-versions
# Check with -> rke config --list-version --all
kubernetes_version:
# Etcd snapshots
services:
  etcd:
    backup_config:
      interval_hours: 12
      retention: 6
    snapshot: true
    creation: 6h
    retention: 24h

kube-api:
  # IP range for any services created on Kubernetes
  #  This must match the service_cluster_ip_range in kube-controller
  service_cluster_ip_range: 10.43.0.0/16
  # Expose a different port range for NodePort services
  service_node_port_range: 30000-32767
  pod_security_policy: false


kube-controller:
  # CIDR pool used to assign IP addresses to pods in the cluster
  cluster_cidr: 10.42.0.0/16
  # IP range for any services created on Kubernetes
  # # This must match the service_cluster_ip_range in kube-api
  service_cluster_ip_range: 10.43.0.0/16
  
kubelet:
  # Base domain for the cluster
  cluster_domain: cluster.local
  # IP address for the DNS service endpoint
  cluster_dns_server: 10.43.0.10
  # Fail if swap is on
  fail_swap_on: false
  # Set max pods to 150 instead of default 110
  extra_args:
    max-pods: 150

# Currently only nginx ingress provider is supported.
# To disable ingress controller, set `provider: none`
# `node_selector` controls ingress placement and is optional
ingress:
  provider: nginx
  options:
     use-forwarded-headers: "true"
````

# Deploy Kubernetes Cluster with RKE
#### Once you’ve created the cluster.yml file, you can deploy your cluster with a simple command.
```bash
rke up
```
#### This command assumes the cluster.yml file is in the same directory as where you are running the command. If using a different filename, specify it like below.
```bash
rke up --config ./rancher_cluster.yml
```
## Accessing your Kubernetes cluster
#### As part of the Kubernetes creation process, a kubeconfig file has been created and written at kube_config_cluster.yml.
```bash
export KUBECONFIG=./kube_config_cluster.yml
```

### Check list of nodes in the cluster.
```bash
kubectl get pod
```