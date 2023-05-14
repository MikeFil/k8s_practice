*readme.md of tot-kot/k8s_practice*  
Kubespray взят с репо SouthBridgeio https://github.com/southbridgeio/kubespray  
### Минимальные требования
Для установки master'ов нужны 2GB RAM, для запуска node достаточно 1GB RAM.  
Минимальные требования для node с 1GB RAM занижены в файле:  
edit *kubespray/roles/kubernetes/preinstall/defaults/main.yml*
```
# Default value:
# minimal_node_memory_mb: 1024
minimal_node_memory_mb: 768
```

### Выполнить перед установкой:
```
git clone https://github.com/tot-kot/k8s_practice
cd k8s_practice/kubespray
sudo pip3 install -r requirements.txt

# На master-1 установить libselinux-python3, иначе не встанет kubectl:
sudo yum install libselinux-python3 -y
```

#### Адаптация под окружение:
```
cd k8s_practice/kubespray/inventory
cp -r s000 s056570
cd s056570
```

edit *inventory.ini*
```
master-1.s056570.slurm.io ansible_host=172.16.232.2 ip=172.16.232.2
master-2.s056570.slurm.io ansible_host=172.16.232.3 ip=172.16.232.3
master-3.s056570.slurm.io ansible_host=172.16.232.4 ip=172.16.232.4
ingress-1.s056570.slurm.io ansible_host=172.16.232.5 ip=172.16.232.5
node-1.s056570.slurm.io ansible_host=172.16.232.6 ip=172.16.232.6
node-2.s056570.slurm.io ansible_host=172.16.232.7 ip=172.16.232.7

[kube_control_plane]
master-1.s056570.slurm.io
master-2.s056570.slurm.io
master-3.s056570.slurm.io

[etcd]
master-1.s056570.slurm.io
master-2.s056570.slurm.io
master-3.s056570.slurm.io

[kube_node]
node-1.s056570.slurm.io
node-2.s056570.slurm.io
ingress-1.s056570.slurm.io

[kube_ingress]
ingress-1.s056570.slurm.io

[k8s_cluster:children]
kube_node
kube_control_plane
```

edit *group_vars/k8s_cluster/k8s-cluster.yml*
```
kube_network_plugin: flannel
kube_proxy_mode: iptables
cluster_name: s056570.local
kube_version: v1.21.4
```

edit *group_vars/k8s_cluster/k8s-net-flannel.yml*
```
flannel_interface_regexp: '172\\.16\\.232\\.\\d{1,3}'
```

*Отключить установку ingress*  
edit *group_vars/k8s_cluster/addons*
```
ingress_nginx_enabled: false
```

edit *k8s_practice/kubespray/_deploy_cluster.sh*
```
ansible-playbook -u "$1" -i inventory/s056570/inventory.ini cluster.yml -b --diff
```

### Запуск установки кластера куба
Установку можно производить с любого хоста, но **kubectl** будет установлен на master-1 под указанным пользователем, а kubeconfig будет лежать под root в *.kube/config*
```
bash _deploy_cluster.sh s056570
```

### Удаление кластера
```
ansible-playbook -u "s056570" -i inventory/s056570/inventory.ini reset.yml -b --diff
```