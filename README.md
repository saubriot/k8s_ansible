# Install a Kubernetes cluster using Ansible

# 1. Introduction

We will use Ansible to install a Kubernetes Cluster including its Manageùent Dashboard.

# 2. Architecture

Demo architecture used for the cluster (see https://github.com/saubriot/k8s_vagrant/)

![Architecture](https://github.com/saubriot/k8s_ansible/blob/master/images/k8s_architecture.png)

# 3. Prerequisites
- Ansible installed on the local host that will run the playbook
- Create an ansible user account with sudo and NO_PASSWD on all hosts (/etc/sudoers.d/ansible_no_password)
- Allow the local ansible account to connect remote hosts using certificate (ssh-copy)
- Requires hosts pre installed. See https://github.com/saubriot/k8s_vagrant for the VMs creation using vagrant for the demo environment.

# 4. Cluster installation
## 4.1. How it works : short explanation 

The ansible directory structure has been defined as followed :
- **ansible** : contains the main playbooks, the environments to deploy (under inventories) and the roles and tasks to execute (under roles) 
  - k8s_admin.yml : install administration host
  - k8s_master_node.yml : install kubernetes master node
  - k8s_nodes.yml : install kubernetes node with master node join
  - k8s_core.yml : install the core kubernetes components : cni, metallb, nginx, dashboard
  - platform.yml : the all in one playbook
  - **inventories** : contains information about the environments to deploy
    - **demo** : demo environment
      - hosts : contains the hosts list per role. Note : roles are matching the main playbooks
      - **group_vars** :
        - all : global settings : all settings and kubernetes settings
        - k8s_admin : settings for administration host (role k8s_admin) : bind and openvpn settings
  - **roles**
    - **common** : common installation tasks to execute
    - **docker** : docker packages installation tasks to execute
    - **k8s** : kubernetes packages installation tasks to execute
    - **k8s_admin** : administration installation tasks to execute
    - **k8s_core** : kubernetes core components installation tasks to execute
    - **k8s_master_nodes** : kubernetes master node installation tasks to execute
    - **k8s_nodes** : kubernetes nodes installation tasks to execute (not including master node)
    - **k8s_reset** : specific tasks to reset kubernetes installation : revert to a clean pre installation


## 4.2. Install k8s administration host
### 4.2.a. Objective
The administration host includes 2 main components :
- A DNS Server for .europe domain name resolution including hostnames (ex : paris.europe) and kubernetes domain names (ex : dashboard.k8s.europe).
- An OpenVPN Server to connect the Cluster (usefull if you want restrict external access on a specific interface)

### 4.2.b. Install using vagrant account
Move to ansible directory (assuming git repo is installed in ~/k8s_ansible) and run the playbook k8s_admin.yml.
```
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_admin.yml -u vagrant
```
```

PLAY [k8s_admin] ********************************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [strasbourg.europe]
...
...

PLAY RECAP **************************************************************************************
strasbourg.europe          : ok=43   changed=33   unreachable=0    failed=0   

```
### 4.2.c. Using --extra-vars to customize installation
The playbook accepts 2 extra vars :
- operation : could be either "install" or "delete"
- task : could be either "all" or the task to execute. In our case : "openvpn", "bind" and "others".

Examples :

Install all :
```
ansible-playbook -i inventories/demo k8s_admin.yml -u vagrant --extra-vars="operation=install task=all"
```
Delete all :
```
ansible-playbook -i inventories/demo k8s_admin.yml -u vagrant --extra-vars="operation=delete task=all"
```
Install bind :
```
ansible-playbook -i inventories/demo k8s_admin.yml -u vagrant --extra-vars="operation=install task=bind"
```
Delete bind :
```
ansible-playbook -i inventories/demo k8s_admin.yml -u vagrant --extra-vars="operation=delete task=bind"
```

### 4.2.d. Install using ansible account
Log in as ansible, move to ansible directory (assuming git repo is installed in ~/k8s_ansible) and run the playbook k8s_admin.yml without -u parameter.
```
sudo su - ansible
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_admin.yml
```

## 4.3. Install the Kubernetes master node
### 4.3.a. Objective
Install docker, kubernetes packages on the master node.

Configure the master node (kubeadm init) and the kubeadmin user

### 4.3.b. Install using vagrant account
Move to ansible directory (assuming git repo is installed in ~/k8s_ansible) and run the playbook k8s_master_nodes.yml.
```
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_master_nodes.yml -u vagrant
```
```
PLAY [k8s_master_nodes] *************************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [paris.europe]

...
...

PLAY RECAP **************************************************************************************
paris.europe               : ok=39   changed=29   unreachable=0    failed=0   

```

### 4.3.c. Install using ansible account
Log in as ansible, move to ansible directory (assuming git repo is installed in ~/k8s_ansible) and run the playbook k8s_master_nodes.yml without -u parameter.
```
sudo su - ansible
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_master_nodes.yml
```

### 4.3.d. Verify installation
Connect the master node, log in as kubeadmin and execute kubectl command to check installtion.
```
ssh vagrant@192.168.10.10
sudo su - kubeadmin
kubectl get nodes
```
```
NAME            STATUS     ROLES    AGE   VERSION
paris.europe    NotReady   master   60s   v1.18.6
```
```
kubectl get pods -o wide --all-namespaces
```
```
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE    IP              NODE           NOMINATED NODE   READINESS GATES
kube-system   coredns-66bff467f8-dffgs               0/1     Pending   0          95s    <none>          <none>         <none>           <none>
kube-system   coredns-66bff467f8-khmv4               0/1     Pending   0          95s    <none>          <none>         <none>           <none>
kube-system   etcd-paris.europe                      1/1     Running   0          103s   192.168.10.10   paris.europe   <none>           <none>
kube-system   kube-apiserver-paris.europe            1/1     Running   0          103s   192.168.10.10   paris.europe   <none>           <none>
kube-system   kube-controller-manager-paris.europe   1/1     Running   0          103s   192.168.10.10   paris.europe   <none>           <none>
kube-system   kube-proxy-j5rvp                       1/1     Running   0          95s    192.168.10.10   paris.europe   <none>           <none>
kube-system   kube-scheduler-paris.europe            1/1     Running   0          103s   192.168.10.10   paris.europe   <none>           <none>
```

## 4.4. Install the Kubernetes nodes
### 4.4.a. Objective
Install docker, kubernetes packages on the "non master" nodes.

Join the nodes to the master node (kubeadm join).

### 4.4.b. Install using vagrant account
Move to ansible directory (assuming git repo is installed in ~/k8s_ansible) and run the playbook k8s_nodes.yml.
```
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_nodes.yml -u vagrant
```
```
PLAY [k8s_nodes] ********************************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [roma.europe]
ok: [madrid.europe]
ok: [lisboa.europe]

...
...

PLAY RECAP **************************************************************************************
lisboa.europe              : ok=34   changed=25   unreachable=0    failed=0   
madrid.europe              : ok=34   changed=25   unreachable=0    failed=0   
roma.europe                : ok=34   changed=25   unreachable=0    failed=0   

```

### 4.4.c. Install using ansible account
Log in as ansible, move to ansible directory (assuming git repo is installed in ~/k8s_ansible) and run the playbook k8s_nodes.yml without -u parameter.
```
sudo su - ansible
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_nodes.yml
```

### 4.4.d. Verify installation
Connect the master node, log in as kubeadmin and execute kubectl command to check installtion.
```
ssh vagrant@192.168.10.10
sudo su - kubeadmin
kubectl get nodes
```
```
NAME            STATUS     ROLES    AGE     VERSION
lisboa.europe   NotReady   <none>   13s     v1.18.6
madrid.europe   NotReady   <none>   13s     v1.18.6
paris.europe    NotReady   master   4m34s   v1.18.6
roma.europe     NotReady   <none>   13s     v1.18.6
```
```
kubectl get pods -o wide --all-namespaces
```
```
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE    IP              NODE            NOMINATED NODE   READINESS GATES
kube-system   coredns-66bff467f8-dffgs               0/1     Pending   0          5m1s   <none>          <none>          <none>           <none>
kube-system   coredns-66bff467f8-khmv4               0/1     Pending   0          5m1s   <none>          <none>          <none>           <none>
kube-system   etcd-paris.europe                      1/1     Running   0          5m9s   192.168.10.10   paris.europe    <none>           <none>
kube-system   kube-apiserver-paris.europe            1/1     Running   0          5m9s   192.168.10.10   paris.europe    <none>           <none>
kube-system   kube-controller-manager-paris.europe   1/1     Running   0          5m9s   192.168.10.10   paris.europe    <none>           <none>
kube-system   kube-proxy-j5rvp                       1/1     Running   0          5m1s   192.168.10.10   paris.europe    <none>           <none>
kube-system   kube-proxy-qdpkz                       1/1     Running   0          59s    192.168.10.14   lisboa.europe   <none>           <none>
kube-system   kube-proxy-sf7c6                       1/1     Running   0          59s    192.168.10.12   roma.europe     <none>           <none>
kube-system   kube-proxy-tn22j                       1/1     Running   0          59s    192.168.10.13   madrid.europe   <none>           <none>
kube-system   kube-scheduler-paris.europe            1/1     Running   0          5m9s   192.168.10.10   paris.europe    <none>           <none>
```

## 4.5. Install the Kubernetes core components
### 4.5.a. Objective
Install the kubernetes core components :
- cni : network : weave by default
- cfssl : tools to generate certificates
- nginx : ingress controller
- metallb : external load balancer
- dashboard : kubernetes dashboard

### 4.5.b. Install using vagrant account
Move to ansible directory (assuming git repo is installed in ~/k8s_ansible) and run the playbook k8s_core.yml.
```
cat << EOF > ~/.ansible/ansible.cfg
[defaults]
allow_world_readable_tmpfiles=true
EOF
export ANSIBLE_CONFIG=~/.ansible/ansible.cfg 

cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_core.yml -u vagrant
```
```
PLAY [k8s_core] *********************************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [paris.europe]

...
...

PLAY RECAP **************************************************************************************
paris.europe               : ok=28   changed=21   unreachable=0    failed=0   

```

### 4.5.c. Install using ansible account
Log in as ansible, move to ansible directory (assuming git repo is installed in ~/k8s_ansible) and run the playbook k8s_core.yml without -u parameter.
```
sudo su - ansible
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_core.yml
```

### 4.5.d. Verify installation
Connect the master node, log in as kubeadmin and execute kubectl command to check installtion.
```
ssh vagrant@192.168.10.10
sudo su - kubeadmin
kubectl get pods -o wide --all-namespaces
```
```
kube-system            coredns-66bff467f8-dffgs                     0/1     Running   0          8m31s   10.32.0.2       lisboa.europe   <none>           <none>
kube-system            coredns-66bff467f8-khmv4                     0/1     Running   0          8m31s   10.40.0.1       roma.europe     <none>           <none>
kube-system            etcd-paris.europe                            1/1     Running   0          8m39s   192.168.10.10   paris.europe    <none>           <none>
kube-system            kube-apiserver-paris.europe                  1/1     Running   0          8m39s   192.168.10.10   paris.europe    <none>           <none>
kube-system            kube-controller-manager-paris.europe         1/1     Running   0          8m39s   192.168.10.10   paris.europe    <none>           <none>
kube-system            kube-proxy-j5rvp                             1/1     Running   0          8m31s   192.168.10.10   paris.europe    <none>           <none>
kube-system            kube-proxy-qdpkz                             1/1     Running   0          4m29s   192.168.10.14   lisboa.europe   <none>           <none>
kube-system            kube-proxy-sf7c6                             1/1     Running   0          4m29s   192.168.10.12   roma.europe     <none>           <none>
kube-system            kube-proxy-tn22j                             1/1     Running   0          4m29s   192.168.10.13   madrid.europe   <none>           <none>
kube-system            kube-scheduler-paris.europe                  1/1     Running   0          8m39s   192.168.10.10   paris.europe    <none>           <none>
kube-system            weave-net-g8gkr                              2/2     Running   0          113s    192.168.10.12   roma.europe     <none>           <none>
kube-system            weave-net-mkprz                              2/2     Running   0          113s    192.168.10.10   paris.europe    <none>           <none>
kube-system            weave-net-ntmg8                              2/2     Running   0          113s    192.168.10.13   madrid.europe   <none>           <none>
kube-system            weave-net-s5272                              2/2     Running   0          113s    192.168.10.14   lisboa.europe   <none>           <none>
kubernetes-dashboard   dashboard-metrics-scraper-694557449d-bh4cx   1/1     Running   0          96s     10.46.0.1       madrid.europe   <none>           <none>
kubernetes-dashboard   kubernetes-dashboard-9774cc786-bvc4n         1/1     Running   1          96s     10.46.0.0       madrid.europe   <none>           <none>
metallb-system         controller-5f98465b6b-9wkcz                  1/1     Running   0          103s    10.32.0.3       lisboa.europe   <none>           <none>
metallb-system         speaker-ktz5z                                1/1     Running   0          67s     192.168.10.10   paris.europe    <none>           <none>
metallb-system         speaker-nsc27                                1/1     Running   0          78s     192.168.10.12   roma.europe     <none>           <none>
metallb-system         speaker-pbnr7                                1/1     Running   0          78s     192.168.10.13   madrid.europe   <none>           <none>
metallb-system         speaker-ztm7q                                1/1     Running   0          78s     192.168.10.14   lisboa.europe   <none>           <none>
```

### 4.5.e. Restart Libvirt or Virtualbox VMs
If you deploy Kubernetes on VMs using vagrant (see https://github.com/saubriot/k8s_vagrant/) you need to restart the VMs to get load balancer metallb running correctly otherwise you won't be able to connect Kubernetes dashboard.

For libvirt :
```
cd ~/k8s_vagrant/vagrant/libvirt/demo
vagrant halt
vagrant up
```
For Virtualbox :
```
cd ~/k8s_vagrant/vagrant/virtualbox/demo
vagrant halt
vagrant up
```

### 4.5.f. Verify DNS configured on Administration host is working correctly

The DNS should return the IP address 192.168.10.190 for dashboard.k8s.europe : 
```
dig dashboard.k8s.europe @192.168.10.254 +short
```
```
192.168.10.190
```

### 4.5.g. Access Kubernetes dashboard : configure additional DNS server

Configure an additional DNS server for additionnal search domains for ipv4 setting of your network interface card :
- Additional DNS servers: 192.168.10.254
- Additional search domains: .europe

Note : on Debian using Network Manager, a disconnect/connect is necessary on the network interface to get DNS resolution working

### 4.5.h. Access Kubernetes dashboard : get token for login

Run the playbook k8s_core.yml with extra vars operation=check and task=ui-login.
```
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_core.yml -u vagrant --extra-vars="operation=check task=ui-login"
```
```
PLAY [k8s_core] *************************************************************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************************************************
ok: [paris.europe]
...
...

TASK [k8s_core : copy/paste admin user token for k8s dashboard access environment "demo"] ***********************************************************************************************
ok: [paris.europe] => {
    "msg": "eyJhbGciOiJSUzI1NiIsImtpZCI6Inl0ZUZsTTE2Ylp6SlBEc1U3cE5Jc29TT1FLeE9BM1E3WE1CR25NNDJBMVkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXBwNnZjIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjOWY3YTliYi1kZWQ3LTQ5NTAtYjg3OS04YTQ2MWFhMWNmMGQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.CJBPjkypUAy4vA8MNw4HeaodVfQE37EwyD2qF9PKsfpBr2h6eje45xXNZiRGTNhFTbmpRx0Tzd7rdWatgn6VYCgDflozPUJR14Qobu_itoVHTYqwKuFi232wrdHLRiw9xYyxGShGjtb3XZkCk9yCf8kkwFPXIhWtB52x8M-CSq3P0M9NHkBMUL6CKpzjojHfBHI60x7K-BK8BUVAShn6Dxz5Ya1cCwTu2SSrhs1Ne_TjSD8ZaFd3ZD2XdPHi1T6-JBBi8dF-2U5XKVYCPlFqFzvxoA3RpDKqdcTKfQrBwvzC6nuCUxUep4QQ7rPeMVeBDHUtwKb5QjEwE_5QW0bK0A"
}

PLAY RECAP ******************************************************************************************************************************************************************************
paris.europe               : ok=7    changed=3    unreachable=0    failed=0   

```
Copy/paste the content of msg (between quotes) :

```
eyJhbGciOiJSUzI1NiIsImtpZCI6Inl0ZUZsTTE2Ylp6SlBEc1U3cE5Jc29TT1FLeE9BM1E3WE1CR25NNDJBMVkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXBwNnZjIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjOWY3YTliYi1kZWQ3LTQ5NTAtYjg3OS04YTQ2MWFhMWNmMGQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.CJBPjkypUAy4vA8MNw4HeaodVfQE37EwyD2qF9PKsfpBr2h6eje45xXNZiRGTNhFTbmpRx0Tzd7rdWatgn6VYCgDflozPUJR14Qobu_itoVHTYqwKuFi232wrdHLRiw9xYyxGShGjtb3XZkCk9yCf8kkwFPXIhWtB52x8M-CSq3P0M9NHkBMUL6CKpzjojHfBHI60x7K-BK8BUVAShn6Dxz5Ya1cCwTu2SSrhs1Ne_TjSD8ZaFd3ZD2XdPHi1T6-JBBi8dF-2U5XKVYCPlFqFzvxoA3RpDKqdcTKfQrBwvzC6nuCUxUep4QQ7rPeMVeBDHUtwKb5QjEwE_5QW0bK0A
```

### 4.5.i. Access Kubernetes dashboard : login

Open your browser (Firefox in our case) at https://dashboard.k8s.europe 

Note : The certificate is a self certificate generated usng cfssl. Click on [Advanced...] 

![Dashboard Security Warning](https://github.com/saubriot/k8s_ansible/blob/master/images/dashboard-security-warning-advanced.png)

- Click on [View Certificate]

![Dashboard View Certificate](https://github.com/saubriot/k8s_ansible/blob/master/images/dashboard-security-warning-advanced-view-certificate.png)


- Click on [Accept the Risk and Continue]

![Dashboard Login Page](https://github.com/saubriot/k8s_ansible/blob/master/images/dashboard-login-page.png)

- Paste the token
- Click [Sign in]

![Dashboard Overview Page](https://github.com/saubriot/k8s_ansible/blob/master/images/dashboard-overview-page.png)

> You can now manage your Kubernetes cluster using the dashboard 

### 4.5.j. Using --extra-vars to customize installation
The playbook accepts 2 extra vars :
- operation : could be either "install", "delete" or "check" (only for task "ui-login")
- task : could be either "all" or the task to execute. In our case : "cni", "cfssl", "nginx", "ui" and "ui-login" (only for operation check).

Examples :

Install all :
```
ansible-playbook -i inventories/demo k8s_core.yml -u vagrant --extra-vars="operation=install task=all"
```
Delete dashboard :
```
ansible-playbook -i inventories/demo k8s_core.yml -u vagrant --extra-vars="operation=delete task=ui"
```
Install dashboard :
```
ansible-playbook -i inventories/demo k8s_core.yml -u vagrant --extra-vars="operation=install task=ui"
```

## 4.6. Cleanup Kubernetes cluster installation

Run the playbook k8s_reset.yml to remove the kubernetes cluster. After this operation you are able to reinstall a fresh cluster : master + nodes + core components. 
```
cd ~/k8s_ansible/ansible
ansible-playbook -i inventories/demo k8s_reset.yml -u vagrant
```
