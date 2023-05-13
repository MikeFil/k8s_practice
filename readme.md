# readme.md of tot-kot/k8s_practice
Kubespray взят с репо SouthBridgeio https://github.com/southbridgeio/kubespray

`git clone https://github.com/tot-kot/k8s_practice`
`cd k8s_practice/kubespray`
`pip3 install -r requirements.txt`

Выполненная адаптация под окружение
```
cd k8s_practice/kubespray/inventory
cp -r s000 s056570
cd s056570
```
edit inventory


edit group_vars/k8s_cluster/k8s-cluster.yml

edit group_vars/k8s_cluster/k8s-net-flannel.yml

# отключить установку ingress
edit group_vars/k8s_cluster/addons

edit _deploy_cluster.sh