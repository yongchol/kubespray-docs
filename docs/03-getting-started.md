
# Kubespray getting started

## Test Environment 

### OS/SW 
- Ubuntu 16.04
- Kubespray v2.11 
- Ansible v2.7.8
- kubernetes v1.15.3

### Nodes
- 3 Master nodes
- 3 Worker nodes

## Download kubespray

```
wget -O kubespray-v2.11.0.tar.gz https://github.com/kubernetes-incubator/kubespray/archive/v2.11.0.tar.gz
tar zxvf kubespray-v2.11.0.tar.gz
```

## Install python library

```
sudo apt install python-pip -y
pip install -r ./kubespray-v2.11.0/requirements.txt
```

## Configure hosts file
```
echo "[all]
poc-km-01 ansible_ssh_host=${KM_01_IP} ip=${KM_01_IP}
poc-km-02 ansible_ssh_host=${KM_02_IP} ip=${KM_02_IP}
poc-km-03 ansible_ssh_host=${KM_03_IP} ip=${KM_03_IP}
poc-wrk-01 ansible_ssh_host=${WRK_01_IP} ip=${WRK_01_IP}
poc-wrk-02 ansible_ssh_host=${WRK_02_IP} ip=${WRK_02_IP}
poc-wrk-03 ansible_ssh_host=${WRK_03_IP} ip=${WRK_03_IP}

[kube-master]
poc-km-01
poc-km-02
poc-km-03

[etcd]
poc-km-01
poc-km-02
poc-km-03

[kube-node]
poc-wrk-01
poc-wrk-02
poc-wrk-03

[k8s-cluster:children]
kube-master
kube-node" >> ./inventory/{custom}/hosts.ini
```

## Review and change parameters

- inventory/{custom}/group_vars/all/all.yml
- inventory/{custom}/group_vars/k8s-cluster/addons.yml
- inventory/{custom}/group_vars/k8s-cluster/k8s-cluster.yml

> some parameter options to check it out
    
        # Choose network plugin (cilium, calico, contiv, weave or flannel)
        kube_network_plugin: calico

        ## Change this to use another Kubernetes version, e.g. a current beta release
        kube_version: v1.14.6

        # audit log for kubernetes
        kubernetes_audit: false

        # Make a copy of kubeconfig on the host that runs Ansible
        kubeconfig_localhost: true
        
        # Download kubectl onto the host that runs Ansible in {{ bin_dir }}
        kubectl_localhost: true

        # Helm deployment
        helm_enabled: true

        # Nginx ingress controller deployment
        ingress_nginx_enabled: true

> kubelet_cgroup_driver variables should be modifed for add nodes
  (kubelet cgroup dirver must be the same as docker cgroup driver)

        # default value - systemd 
        kubelet_cgroup_driver: cgroupfs 


## Deploy k8s cluster

```
ansible-playbook -i inventory/{custom}/hosts.ini -b cluster.yml
```

## Add nodes

- configure hosts.ini files to add new node information
```
[all]
...
...
new_node ansible_host={ip} ip={ip}
...
...
[kube-node]
...
...
new_node
```

- run sclae.yml playbook
```
ansible-playbook -i inventory/{custom}/hosts.ini -b scale.yml
```

## Remove nodes

- Use --extra-vars "node={nodename}" to select the node you want to delete
```
ansible-playbook -i inventory/{custom}/hosts.ini -b remove-node.yml --extra-vars "node=nodename"
```

## Upgrade

- if you upgrade k8s v1.6.0 in case there must be at least 1 kube-master already deployed
```
ansible-playbook -i inventory/{custom}/hosts.ini -b upgrade-cluster.yml -e kube_version=v1.6.0
```

- component-based upgrades
  - These commands are useful only for upgrading fully-deployed, healthy, existing hosts
```
ansible-playbook -i inventory/{custom}/hosts.ini -b cluster.yml --tags=docker
ansible-playbook -i inventory/{custom}/hosts.ini -b cluster.yml --tags=etcd
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=node --skip-tags=k8s-gen-certs,k8s-gen-tokens
ansible-playbook -i inventory/{custom}/hosts.ini -b cluster.yml --tags=master
```

- refer to tags docs : https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ansible.md
- refers to upgrade docs : https://github.com/kubernetes-sigs/kubespray/blob/master/docs/upgrades.md

## Reset

- Reset the entire cluster for fresh installation
```
ansible-playbook -i inventory/{custom}/hosts.ini -b reset.yml
```

## References

https://kubespray.io

https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md

https://schoolofdevops.github.io/ultimate-kubernetes-bootcamp/cluster_setup_kubespray/#high-available-kubernetes-cluster-setup-using-kubespray

https://medium.com/@IrekRomaniuk/kubespraying-on-premise-8c09d74b6e3a
