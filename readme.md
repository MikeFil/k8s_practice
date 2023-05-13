*readme.md of tot-kot/k8s_practice*  
Kubespray взят с репо SouthBridgeio https://github.com/southbridgeio/kubespray
### Выполнить перед установкой:
```
git clone https://github.com/tot-kot/k8s_practice
cd k8s_practice/kubespray
pip3 install -r requirements.txt
```
#### Адаптация под окружение:
```
cd k8s_practice/kubespray/inventory
cp -r s000 s056570
cd s056570
```
edit inventory
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
edit group_vars/k8s_cluster/k8s-cluster.yml
```
kube_network_plugin: flannel
kube_proxy_mode: iptables
cluster_name: s056570.local
```
edit group_vars/k8s_cluster/k8s-net-flannel.yml
```
flannel_interface_regexp: '172\\.16\\.232\\.\\d{1,3}'
```
*Отключить установку ingress*  
edit group_vars/k8s_cluster/addons
```
ingress_nginx_enabled: false
```
edit deploy_cluster.sh