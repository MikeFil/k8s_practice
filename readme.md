*readme.md of tot-kot/k8s_practice*  
# 1.2 Развернуть кластер Kubernetes с помощью Kubespray
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

### Удаление кластера, если что-то пошло не по плану
```
ansible-playbook -u "s056570" -i inventory/s056570/inventory.ini reset.yml -b --diff
```

# 1.3 Установить Helm, Nginx Ingress Controller и Cert-manager
## Helm
```
cd k8s_practice/helm

# Установка repo с Helm
sudo yum install http://rpms.southbridge.ru/southbridge-rhel7-stable.rpm -y

# Установка Helm v3.6.3
sudo yum install helm -y

# Добавление repo для Helm
helm repo add stable https://charts.helm.sh/stable
helm repo add jetstack https://charts.jetstack.io/
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Загружаем values для ingress-nginx
helm show values ingress-nginx/ingress-nginx > nginx-values.yaml
```

edit *nginx-values.yaml*
```
  hostNetwork: true
  tolerations:
    - key: "node-role.kubernetes.io/ingress"
      operator: "Exists"
  nodeSelector:
    node-role.kubernetes.io/ingress: ""
```

Устанавливаем ingress-nginx 
```
helm install ingress-nginx ingress-nginx/ingress-nginx -f nginx-values.yaml
```

## Cert-manager
```
cd k8s_practice/cert-manager

# Установка CRDS сущностей для работы с сертификатами
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

# Создадим namespace для Cert-manager
kubectl create namespace cert-manager

# Установим Cert-manager с помощью Helm
helm install cert-manager \
 --namespace cert-manager \
 --version v1.7.1 \
 --set ingressShim.defaultIssuerName=letsencrypt \
 --set ingressShim.defaultIssuerKind=ClusterIssuer \
 jetstack/cert-manager

# Проверим 3 запущенных пода
kubectl -n cert-manager get pod
```
#### Создаем два манифеста
*clusterissuer-stage.yaml*
```
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: slurm_k8s_du_3@mail.ru
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: stage-issuer-account-key
    # Enable the HTTP01 challenge mechanism for this Issuer
    solvers:
    - http01:
        ingress:
          class:  nginx
...
```
*tls-ingress.yaml*
```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-ingress-nginx
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  rules:
  - host: flask.s056570.edu.slurm.io
    http:
      paths:
      - pathType: ImplementationSpecific
        backend:
          service:
            name: my-service
            port:
              number: 80
  tls:
  - hosts:
    - flask.s056570.edu.slurm.io
    secretName: flask-tls
...
```
Для тестового stage существующий e-mail указывать нет необходимости  
Значения полей '- host(s)' нужно поменять на существующий dns адрес по которому будет доступен ingress
```
# Применяем манифесты cert-manager
kubectl apply -f clusterissuer-stage.yaml
kubectl apply -f tls-ingress.yaml -n default

# Проверяем созданный сертификат и секреты
kubectl get certificate flask-tls -o yaml
kubectl get secret flask-tls -o yaml

# Убедимся, что сертификат подписан stage CA от letsencrypt
curl https://flask.s056570.edu.slurm.io -k -v
```

# Подготовка серевра gitlab
```
# проверка yum-utils
yum install yum-utils

# установка docker и docker-compose
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io -y
systemctl enable --now docker
curl -L "https://github.com/docker/compose/releases/download/1.28.6/docker-compose-Linux-x86_64" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# клонирование проекта
mkdir gitlab && cd gitlab/
git clone git@gitlab.slurm.io:edu/workshop.git

# включение swap
dd if=/dev/zero of=/swapfile count=4096 bs=1MiB
ls -lh /swapfile
сhmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# установка gitlub и runner
yum install -y curl policycoreutils-python perl
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
yum install gitlab-ce
yum install gitlab-runner
systemctl enable gitlab-runner --now
```
