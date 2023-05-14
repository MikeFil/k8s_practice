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
### Запуск установки куба
```
bash _deploy_cluster.sh s056570
```

Подготовка серевеа gitlab

проверка yum-utils
yum install yum-utils

установка docker и docker-compose
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io -y
systemctl enable --now docker
curl -L "https://github.com/docker/compose/releases/download/1.28.6/docker-compose-Linux-x86_64" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

клонирование проекта
mkdir gitlab && cd gitlab/
git clone git@gitlab.slurm.io:edu/workshop.git

включение swap
dd if=/dev/zero of=/swapfile count=4096 bs=1MiB
ls -lh /swapfile
сhmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

установка gitlub и runner
yum install -y curl policycoreutils-python perl
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
yum install gitlab-ce
yum install gitlab-runner
systemctl enable gitlab-runner --now
