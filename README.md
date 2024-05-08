# Kubespray Installation Documentation

## Date: 22/01

### K8s Cluster Deployment on APC Site using Kubespray

#### Step 1: Virtual Machines Setup on Proxmox APC

Create 7 VMs on Proxmox APC using the template named '150 (k8s-lab-template)' with the following IP configurations:

| VM Name               | MGMT             | DMZ              | Cluster           |
|-----------------------|------------------|------------------|-------------------|


#### Step 2: Network Configuration with "netplan"

Configure the network for each VM using "netplan."
For the masters there is only 1 netplay file:
```bash
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens18:
      addresses:
      - 10.102.14.142/24
      nameservers:
        addresses:
        - 10.102.82.6
        search: []
      routes:
      - to: 10.99.0.0/16
        via: 10.102.14.1
    ens19:
      addresses:
      - 10.102.45.54/25
      gateway4: 10.102.45.1
      nameservers:
        #addresses: []
        search: []
    ens21:
      addresses:
      - 10.106.0.24/24
  version: 2
```
The Worker and Storage nodes need anouther file:
```bash vim /etc/netplan/01-netcfg.yaml ```
```bash

network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: no
    eno2:
      dhcp4: no
    ens1f1np1: {}
    ens2f0np0: {}
    ens1f0np0: {}
    ens2f1np1: {}
  bonds:
   bond0:
    interfaces:
        - ens1f1np1
        - ens2f1np1
    parameters:
      mode: 802.3ad
      transmit-hash-policy: layer3+4
      mii-monitor-interval: 100
   bond1:
    interfaces:
        - ens1f0np0
        - ens2f0np0
    parameters:
      mode: 802.3ad
      transmit-hash-policy: layer3+4
      mii-monitor-interval: 100

  vlans:
    bond0.4080:
      id: 4080
      link: bond0
      addresses: [10.106.0.1/24]
      routes:
      - to: 10.202.110.8/29 #F5-LB
        via: 10.106.0.254
      mtu: 9000
  version: 2
```
#### Step 3: Installing Kubespray on Bastion

Follow the provided links to install Kubespray on the Bastion:

- [Kubespray GitHub](https://github.com/kubernetes-sigs/kubespray)
- [Red Hat Sysadmin Kubespray Deployment](https://www.redhat.com/sysadmin/kubespray-deploy-kubernetes)

Run the following commands on the Bastion:

```bash
vim /etc/hosts
k8s-bastion-lab-apc
k8s-master01-lab-apc
k8s-master02-lab-apc
k8s-master03-lab-apc
k8s-worker01-lab-apc
k8s-worker02-lab-apc
k8s-worker03-lab-apc

10.105.255.254 cluster.local Kubernetes
10.105.255.249 cluster.local Kubernetes dashboard.Kubernetes
10.101.206.20 cluster.local kubernetes dashboard.kubernetes
```

Ensure Bastion has access to all nodes using ssh-copy-id, `use this command to run ssh all the nodes`:
```bash
for server in k8s-bastion-prod-apc k8s-master01-prod-apc k8s-master02-prod-apc k8s-master03-prod-apc k8s-worker01-prod-apc k8s-worker02-prod-apc k8s-worker03-prod-apc k8s-worker04-prod-apc k8s-storage01-prod-apc k8s-storage02-prod-apc k8s-storage03-prod-apc; do ssh-copy-id -o "StrictHostKeyChecking no" master@$server; done
```
Configuring Kernel for packet forwarding at All nodes
```bash
echo net.ipv4.ip_forward=1 > /etc/sysctl.conf && sysctl -p #loading the file changes
```

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
---
kube_config_dir: /etc/kubernetes
kube_script_dir: "{{ bin_dir }}/kubernetes-scripts"
kube_manifest_dir: "{{ kube_config_dir }}/manifests"
kube_cert_dir: "{{ kube_config_dir }}/ssl"
kube_token_dir: "{{ kube_config_dir }}/tokens"
kube_api_anonymous_auth: true
kube_version: v1.28.6
local_release_dir: "/tmp/releases"
retry_stagger: 5
kube_owner: kube
kube_cert_group: kube-cert
kube_log_level: 2
credentials_dir: "{{ inventory_dir }}/credentials"
kube_network_plugin: calico
kube_network_plugin_multus: false
kube_service_addresses: 10.233.0.0/18
kube_pods_subnet: 10.233.64.0/18
kube_network_node_prefix: 24
enable_dual_stack_networks: false
kube_service_addresses_ipv6: fd85:ee78:d8a6:8607::1000/116
kube_pods_subnet_ipv6: fd85:ee78:d8a6:8607::1:0000/112
kube_network_node_prefix_ipv6: 120
kube_apiserver_ip: "{{ kube_service_addresses | ipaddr('net') | ipaddr(1) | ipaddr('address') }}"
kube_apiserver_port: 6443  # (https)
kube_proxy_mode: ipvs
kube_proxy_strict_arp: false
kube_proxy_nodeport_addresses: >-
  {%- if kube_proxy_nodeport_addresses_cidr is defined -%}
  [{{ kube_proxy_nodeport_addresses_cidr }}]
  {%- else -%}
  []
  {%- endif -%}
kube_encrypt_secret_data: true
cluster_name: cluster.local
ndots: 2
dns_mode: coredns
enable_nodelocaldns: true
enable_nodelocaldns_secondary: false
nodelocaldns_ip: 169.254.25.10
nodelocaldns_health_port: 9254
nodelocaldns_second_health_port: 9256
nodelocaldns_bind_metrics_host_ip: false
nodelocaldns_secondary_skew_seconds: 5
enable_coredns_k8s_external: false
coredns_k8s_external_zone: k8s_external.local
enable_coredns_k8s_endpoint_pod_names: true
resolvconf_mode: host_resolvconf
deploy_netchecker: false
skydns_server: "{{ kube_service_addresses | ipaddr('net') | ipaddr(3) | ipaddr('address') }}"
skydns_server_secondary: "{{ kube_service_addresses | ipaddr('net') | ipaddr(4) | ipaddr('address') }}"
dns_domain: "{{ cluster_name }}"
container_manager: crio
kata_containers_enabled: false
kubeadm_certificate_key: "{{ lookup('password', credentials_dir + '/kubeadm_certificate_key.creds length=64 chars=hexdigits') | lower }}"
k8s_image_pull_policy: IfNotPresent
kubernetes_audit: false
default_kubelet_config_dir: "{{ kube_config_dir }}/dynamic_kubelet_dir"
volume_cross_zone_attachment: false
persistent_volumes_enabled: false
event_ttl_duration: "1h0m0s"
auto_renew_certificates: true
auto_renew_certificates_systemd_calendar: "Mon *-*-1,2,3,4,5,6,7 03:{{ groups['kube_control_plane'].index(inventory_hostname) }}0:00"
kubeadm_patches:
  enabled: false
  source_dir: "{{ inventory_dir }}/patches"
  dest_dir: "{{ kube_config_dir }}/patches"
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
ansible-playbook -i inventory/k8s-lab/hosts.yaml --become -u master cluster.yml -v -Kk #(The Kk is for sudo passwd. needed to install sshpass)
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
''' >> ~/.bashrc
```


#### Step 9: Day 2 Operations

Add aliases for convenience:

```bash
echo '''
alias kp='kubectl get pods'
alias kd='kubectl get deployment'
alias k='kubectl'
alias ka='kubectl get all'
alias ks='kubectl get service'
alias d='docker'
''' >> ~/.bashrc
```

Disable Workloads on Kubernetes Control Plane Nodes:
Do it for all the Master nodes;
```bash
kubectl taint nodes k8s-master-prod-01 node-role.kubernetes.io/master=true:NoSchedule
kubectl taint nodes k8s-master-prod-01 node-role.kubernetes.io/control-plane=true:NoSchedule
```

#### Step 10: Configure registry mirroring
```bash
vim /etc/containers/registries.conf.d/01-unqualified.conf 
#####
[[registry]]
location="registry-1.docker.io"
[[registry.mirror]]
location="harbor-"

[[registry]]
location="docker.io"
[[registry.mirror]]
location="harbor-

[[registry]]
location="quay.io"
[[registry.mirror]]
location="harbor-"

```
# restart the crio service
systemctl restart crio
