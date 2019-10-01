# Poc Env setup for kubespray

## Architecture

### Full architecture

![origin-architecure](../imgs/kubespray-test-architecture.png)

### Light architecture

- It is excluded build servers, Nginx Gateway
- dops vm will also perform nginx reverse proxy instead of ngw vm

![minimum-architecure](../imgs/kubespray-test-architecture-mini.png)

## EC2 List

|Name            |desc         |subnet            |
|:--------------:|:-----------:|:----------------:|
|   poc-tf-vm    |  terraform  |  outside         |
|   poc-jmp-01   |  jumphost   |  subnet-jmp      |
|   poc-dops-01  |  devops     |  subnet-devops   |
|   poc-bld-01   |  jenkins    |  subnet-devops   |
|   poc-bld-02   |  jenkins    |  subnet-devops   |
|   poc-con-01   |  concourse  |  subnet-devops   |
|   poc-ngw-lb   |  NginxGW    |  subnet-frontend |
|   poc-ngw-01   |  NginxGW    |  subnet-frontend |
|   poc-ngw-02   |  NginxGW    |  subnet-frontend |
|   poc-km-01    |  k8s-master |  subnet-middle   |
|   poc-km-02    |  k8s-master |  subnet-middle   |
|   poc-km-03    |  k8s-master |  subnet-middle   |
|   poc-wrk-01   |  k8s-worker |  subnet-middle   |
|   poc-wrk-02   |  k8s-worker |  subnet-middle   |
|   poc-wrk-03   |  k8s-worker |  subnet-middle   |

## Terraform setup and Iaas Deploy

### 1. Git clone kubespray-test repository (from poc-tf-01 host)
```
ubuntu@poc-tf-vm:~$ git clone <kubespray-test repo>
```

### 2. Install terraform
```
ubuntu@poc-tf-vm:~$ cd ~/kubespray-test/full/tf
ubuntu@poc-tf-vm:~$ bash _tf-init.sh 
```

### 3. Configure .envrc file (open .envrc file and modify ENV, AWS key info)
```
ubuntu@poc-tf-vm:~/kubepray-test$ cp envrc_template .envrc
ubuntu@poc-tf-vm:~/kubepray-test$ vi .envrc
ubuntu@poc-tf-vm:~/kubepray-test$ . .envrc  
```

### 4. IaaS Deploy

This script perfomrs the following tasks
- create ansible keypair
- create terraform template (ec2-list, subnet-list)
- terraform init
- terraform apply

```
ubuntu@poc-tf-vm:~/kubepray-test$ cd full/tf
ubuntu@poc-tf-vm:~/kubepray-test$ bash _run-tf.sh
``` 

## Initialize Devops Env

### 1. Connect dops vm (from poc-jmp-01 host)
```
ubuntu@poc-jmp-01:~$ ssh poc-dops-01
ubuntu@poc-dops-01:~$ sudo su -ansible
```

### 2. Copy ansible_rsa file from poc-tf-01 to poc-dops-01
```
ansible@poc-dops-01:~$ echo "-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----" >> ~/.ssh/id_rsa
ansible@poc-dops-01:~$ chmod 600 ~/.ssh/id_rsa
```

### 3. Git clone kubespray-test repository
```
ansible@poc-dops-01:~$ git clone <kubespray-test repo>
```

### 4. Configure .envrc file (copy envrc file from poc-tf-01 host)
```
ansible@poc-dops-01:~$ cd ~/kubespray-test
ansible@poc-dops-01:~$ vi .envrc
<< copy envrc file from poc-tf-vm and modify>>
ansible@poc-dops-01:~$ . .envrc
```

### 5. Run init-script for dops vm

This script performs the following tasks
- disk volume
- install and configure ansible
- install docker
- install direnv
```
ansible@poc-dops-01:~$ cd ~/kubespray-test/full/devops-init
ansible@poc-dops-01:~$ bash dops-init.sh
ansible@poc-dops-01:~$ exit
ubuntu@poc-dops-01:~$ sudo su - ansible
ansible@poc-dops-01:~$ cd ~/kubespray-test
ansible@poc-dops-01:~$ direnv allow
```

### 6. Run script for install vault and safe
```
ansible@poc-dops-01:~$ cd ~/kubespray-test/full/devops-init
ansible@poc-dops-01:~$ bash dops-init-post.sh
```

### 7. Vault init
```
ansible@poc-dops-01:~$ sudo chown systemd-network:ubuntu /vault/data
ansible@poc-dops-01:~$ sudo docker exec -it vault /bin/sh
  > export VAULT_ADDR='http://127.0.0.1:8200'
  > chown vault:vault /vault/data 
  > vault operator init
  > vault operator unseal [unseal_key_1]
  > vault operator unseal [unseal_key_2]
  > vault operator unseal [unseal_key_3]
  > vault status
  > exit
```
### 8. Configure .envrc file (vault token)
```
ansible@poc-dops-01:~$ vi ~/kubespray-test/.envrc
<< input valut token value >>
ansible@poc-dops-01:~$ direnv allow
```

## Init concourse

### 1. Install concourse
This script performs the following tasks
 - run ansible-playbook concourse-master.yml
 - run ansible-playbook concourse-slave.yml

```
ansible@poc-dops-01:~$ cd ~/kubespray-test/full/devops-init
ansible@poc-dops-01:~$ bash concourse-install.sh
```

### 2. Register vault credential
```
ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/bitbucket/user value=<email>
ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/bitbucket/passwd value=<password>

ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/dops/ansible/host value@/etc/ansible/hosts
ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/dops/ansible/id_rsa value@/home/ansible/.ssh/id_rsa

ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/ssu/uchiwa_password value=<password>
ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/ssu/rmq_password value=<password>
```

### 3. Fly pipeline
This script performs the following tasks
 - fly login
 - fly set-pipeline (with ci-param.yml)
 - fly unpause-pipeline 

```
ansible@poc-dops-01:~$ cd ~/kubespray-test/full/devops-init
ansible@poc-dops-01:~$ bash fly-pipeline-deploy.sh
```

### 4. Configure tunneling
- Use SSH Tunnel manager / Proxy SwitchyOmega program

### 5. Access concourse web and make pipeline status running
- Access 'http://poc-dops-01:8080' and check pipeline

## Init Jekins

### 1. Run jenkins-master playbook
- Use ansible tag 'configure_master' and run only one time
```
ansible@poc-dops-01:~$ ansible-playbook ~/kubespray-test/full/playbooks/bld/main.yml -t configure_master
```

### 2. Register jenkins master key info vault credential
```
ansible@poc-dops-01:~$ cd ~/kubespray-test/full/playbooks/bld
ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/bld/bitbucket/id_rsa value@./bitbucket
ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/bld/ansible/id_rsa_pub value@./master_id_rsa.pub

```

### 3. Run jenkins-slave playbook
```
ansible@poc-dops-01:~$ ansible-playbook ~/kubespray-test/full/playbooks/bld/main.yml -t configure_slave
```

### 4. connect to jekins UI
 - http://poc-bld-01:8080

### 5. Add credentials for slave node
> Menu : Credentials >> System >> Global credentials >> Add Credentials

- Kind : SSH Username with private key
- Scope : Global (Jenkins, nodes, items, all child items, etc)
- Username : jenkins
- Private Key : Copying the id_rsa value of the master node
- Passphrase : blank
- ID : ssh_slave_nodes
- Description : SSH  Access to login to Slave Nodes

### 6. Add slave node
> Menu : Manage Jenkins >> Manage Nodes >> New Node

- Node name : Enter slave node name
- Permanent Agent : Choice
- Name : Node name
- Description : Description of slave node
- of executors : Number of build executors
- Remote root directory : /var/lib/jenkins
- Labels : Labels of slave node
- Usage : Use this node as much as possible
- Launch method : Launch slave agents via SSH
- Host : Hostname of slave node
- Credentials : Select credential for slave node 
- Host Key Verification Strategy : Known hosts file Verification Strategy
- Availability : Keep this agent online as much as possible
- Node Properties : Not selected

## Initialize k8s cluster with kubespray using concourse

### 1. Configure k8s host file
fill up k8s node info following file
```
[all]
nodename ansible_ssh_host=nodeip ip=nodeip
...
...

[kube-master]
nodename
...
...

[etcd]
nodename
..
..

[kube-node]
nodename
..
..

[k8s-cluster:children]
kube-master
kube-node
```

### 2. Register host/rsa vault credential
```
ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/k8s/ansible/host value@/home/ansible/kubespray-test/full/hosts.ini
ansible@poc-dops-01:~$ safe set secret/${REGION}/${ENV}/k8s/ansible/id_rsa value@/home/ansible/.ssh/id_rsa
```

### 3. Running deploy-k8s pipeline
```
ansible@poc-dops-01:$ cd ~/kubespray-test/full/playbooks/k8s
ansible@poc-dops-01:$ ansible-playbook -i inventory/mycluster/hosts.ini -b cluster.yml --flush-cache
```

### 4. Register kubeconfig
```
ansible@poc-dops-01:~$ ssh poc-km-01
ansible@poc-km-01:~$ mkdir -p $HOME/.kube
ansible@poc-km-01:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
ansible@poc-km-01:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5. Check kubernetes cluster status
```
ansible@poc-km-01:~$ kubectl cluster-info
Kubernetes master is running at https://10.11.20.105:6443
coredns is running at https://10.11.20.105:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
KubeDNS is running at https://10.11.20.105:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://10.11.20.105:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

ansible@poc-km-01:~$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
poc-km-01    Ready    master   5m32s   v1.15.3
poc-km-02    Ready    master   4m49s   v1.15.3
poc-km-03    Ready    master   4m50s   v1.15.3
poc-wrk-01   Ready    <none>   4m1s    v1.15.3
poc-wrk-02   Ready    <none>   4m1s    v1.15.3
poc-wrk-03   Ready    <none>   4m2s    v1.15.3

```

### 6. Access kubernetes from remote

- Before install k8s, configure kubespray option for kubectl 
```
kubeconfig_localhost: true
kubectl_localhost: true
```

- Run the following commands when k8s is installed
```
sudo mv {kubespray_inventroy_path}/artifacts/kubectl /usr/bin/
mkdir ~/.kube
cp {kubespray_inventroy_path}/artifacts/admin.conf ~/.kube/config
```

### 7. Access kubernetes dashboard using ingress-controller

- kubernetes-dashboard and Nginx Ingress-controller addon can be installed with kubespray variable config

This command performs the following tasks
- add clusterRoleBinding 
- add ingress resource for kubernetes dashboard
- expose ingress service with nodeport
```
ansible@poc-dops-01:~$ cd ~/kubespray-test/full/yaml
ansible@poc-dops-01:~$ kubectl apply -f kubernetes-dashboard/
```

- Use this command <kubectl -n kube-system edit service kubernetes-dashboard>, then modify like below
``` 
ports:
  - port: 80
    protocol: TCP
    targetPort: 9090
```

- Use this command <kubectl -n kube-system edit deploy kubernetes-dashboard>, then modify like below
```
 ports:
  - containerPort: 9090
    protocol: TCP
    args:
    #- --auto-generate-certificates
```

- Configure reverse proxy config
```
upstream kubernetes-dashboard {
    server poc-wrk-01:30000;
    server poc-wrk-02:30000;
    server poc-wrk-03:30000;
}
proxy_cache_path /var/lib/nginx/cache_kubernetes-dashboard keys_zone=cache_kubernetes-dashboard:10m;
server {
    listen 80;
    server_name dashboard.lab.io;
    client_max_body_size 20M;
    proxy_set_header Host $host;
    location / {
        proxy_pass http://kubernetes-dashboard;
        proxy_cache cache_kubernetes-dashboard;
        proxy_connect_timeout 1200;
        proxy_send_timeout 1200;
        proxy_read_timeout 1200;
        send_timeout 1200;
    }
}
```

- Configure hosts file

```
# poc
<prx ip> grafana.lab.io
<prx ip> alerta.lab.io
<prx ip> dashboard.lab.io
```

8. Access ingress host url by web browser

![kubernetes-dashboard](../imgs/kubernetes-dashboard.png)