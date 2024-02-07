# Kubespray-Installation
# Kubespray Installation Documentation

## Date: 22/01

### K8s Cluster Deployment on APC Site using Kubespray

#### Step 1: Virtual Machines Setup on Proxmox APC

Create 7 VMs on Proxmox APC using the template named '150 (k8s-lab-template)' with the following IP configurations:

| VM Name               | MGMT             | DMZ              | Cluster           |
|-----------------------|------------------|------------------|-------------------|
| k8s-bastion-lab-apc   | 10.102.14.127    | 10.102.45.120    | 10.106.192.7/24   |
| k8s-master01-lab-APC  | 10.102.14.121    | 10.102.45.121    | 10.106.192.1/24   |
| k8s-master02-lab-APC  | 10.102.14.122    | 10.102.45.122    | 10.106.192.2/24   |
| k8s-master03-lab-APC  | 10.102.14.123    | 10.102.45.124    | 10.106.192.3/24   |
| k8s-worker01-lab-APC  | 10.102.14.124    | 10.102.45.125    | 10.106.192.4/24   |
| k8s-worker02-lab-APC  | 10.102.14.125    | 10.102.45.118    | 10.106.192.5/24   |
| k8s-worker03-lab-APC  | 10.102.14.126    | 10.102.45.119    | 10.106.192.6/24   |

#### Step 2: Network Configuration with "netplan"

Configure the network for each VM using "netplan." Set the hostname and add all entries to the "/etc/hosts" file.

#### Step 3: Installing Kubespray on Bastion

Follow the provided links to install Kubespray on the Bastion:

- [Kubespray GitHub](https://github.com/kubernetes-sigs/kubespray)
- [Red Hat Sysadmin Kubespray Deployment](https://www.redhat.com/sysadmin/kubespray-deploy-kubernetes)

Run the following commands on the Bastion:

```bash
vim /etc/hosts
10.102.14.127 k8s-bastion-lab-apc
10.102.14.121 k8s-master01-lab-apc
10.102.14.122 k8s-master02-lab-apc
10.102.14.123 k8s-master03-lab-apc
10.102.14.124 k8s-worker01-lab-apc
10.102.14.125 k8s-worker02-lab-apc
10.102.14.126 k8s-worker03-lab-apc

10.105.255.254 cluster.local Kubernetes
10.105.255.249 cluster.local Kubernetes dashboard.Kubernetes
10.101.206.20 cluster.local kubernetes dashboard.kubernetes
```

Ensure Bastion has access to all nodes using `ssh-copy-id`.

#### Step 4: Kubespray Installation

Follow the links provided to install Kubespray from the Git repository. Install all Python dependencies.
 https://www.redhat.com/sysadmin/kubespray-deploy-kubernetes
 https://github.com/kubernetes-sigs/kubespray/tree/master
 
```bash
# Copy the inventory/sample to inventory/$my-cluster-name$
cp -rfp inventory/sample inventory/$my-cluster-name$
```

Install additional requirements:

```bash
pip3 install -r contrib/inventory_builder/requirements.txt/requirements.txt
```

Generate the hosts.yml file:

```bash
CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

Use the provided template for hosts.yml, adjusting node names as per your configuration.
```bash
all:
  hosts:
    k8s-master01-lab-apc:
      ansible_host: 10.102.14.121
      ip: 10.106.192.1
      access_ip: 10.106.192.1
    k8s-master02-lab-apc:
      ansible_host: 10.102.14.122
      ip: 10.106.192.2
      access_ip: 10.106.192.2
    k8s-master03-lab-apc:
      ansible_host: 10.102.14.123
      ip: 10.106.192.3
      access_ip: 10.106.192.3
    k8s-worker01-lab-apc:
      ansible_host: 10.102.14.124
      ip: 10.106.192.4
      access_ip: 10.106.192.4
    k8s-worker02-lab-apc:
      ansible_host: 10.102.14.125
      ip: 10.106.192.5
      access_ip: 10.106.192.5
    k8s-worker03-lab-apc:
      ansible_host: 10.102.14.126
      ip: 10.106.192.6
      access_ip: 10.106.192.6
  children:
    kube_control_plane:
      hosts:
        k8s-master01-lab-apc:
        k8s-master02-lab-apc:
        k8s-master03-lab-apc:
    kube_node:
      hosts:
        k8s-master01-lab-apc:
        k8s-master02-lab-apc:
        k8s-master03-lab-apc:
        k8s-worker01-lab-apc:
        k8s-worker02-lab-apc:
        k8s-worker03-lab-apc: 
    etcd:
      hosts:
        k8s-master01-lab-apc:
        k8s-master02-lab-apc:
        k8s-master03-lab-apc:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```
#### Step 5: Variable Files Configuration

After setting the hosts.yml file according to your configuration, we need to change some variable files which are at the following locations:

- `inventory/my-cluster/group_vars/k8s-cluster.yml`
 ```bash
kube_proxy_strict_arp: true
kube_encrypt_secret_data: true
enable_coredns_k8s_endpoint_pod_names: true
deploy_netchecker: true
kube_network_plugin: calico
```
---------------------------------------------------------------------
- `/opt/kubespray/inventory/k8s-lab/group_vars/k8s_cluster/addons.yml`
```bash
---
dashboard_enabled: true
helm_enabled: true
registry_enabled: false
metrics_server_enabled: true
local_path_provisioner_enabled: false
local_volume_provisioner_enabled: false
cephfs_provisioner_enabled: false
rbd_provisioner_enabled: false
ingress_nginx_enabled: true
ingress_nginx_host_network: true
ingress_publish_status_address: ""
ingress_alb_enabled: false
cert_manager_enabled: false
metallb_enabled: false
metallb_speaker_enabled: "{{ metallb_enabled }}"
argocd_enabled: true
argocd_version: v2.9.6
argocd_namespace: argocd
argocd_admin_password: "argoadminpassword"
 # - https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli
krew_enabled: false
krew_root_dir: "/usr/local/krew"
```
-----------------------------------------------------------
Adjust settings as needed in both files.

#### Step 6: Ansible and Python Compatibility

Ensure Ansible and Python compatibility as mentioned in the [Kubespray documentation](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ansible.md#installing-ansible):

```plaintext
Ansible Version    Python Version
>= 2.15.5          3.9-3.11
```

Check that all required apps are listed in `requirements.txt`.

#### Step 7: Run Kubespray Playbook

Execute the playbook to deploy Kubespray with the Ansible playbook:

If you already have cluster you need to run this command:
```bash
ansible-playbook -i inventory/my-cluster/hosts.yml reset.yml
```

We are all set you can run the playbook that installs the cluster:
```bash
ansible-playbook -i inventory/k8s-lab/hosts.yaml --become -u master cluster.yml -v -Kk #(The Kk is for sudo passwd)
```

Try running `kubectl` commands, and if it's not working, install it.
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

#### Step 8: Accessing the Cluster from Bastion

After that, the cluster should be installed but the access is only from the masters, 
if you want to access it from the bastion you need to set the cluster API to Haproxy or F5 LB
IF you don't have it yet you can take the admin.conf file from one of the masters in: /etc/kubernetes/admin.conf
take this file to the bastion you can with scp and locate it in /root/admin.conf or /home/master/admin.conf locations
after that, edit this file to "server: https://kubernetes:6443".

make sure the Kubernetes is in the /etc/hosts with the Haproxy / F5 / Master IP

```bash
vim /etc/hosts
10.102.14.121 cluster.local kubernetes dashboard.kubernetes
```

#### Step 9: Adding Aliases

Add aliases for convenience:

```bash
echo '''
alias kp='kubectl get pods'
alias kd='kubectl get deployment'
alias k='kubectl'
alias ka='kubectl get all'
alias ks='kubectl get service'
alias d='docker'
''' >> .bashrc
```
