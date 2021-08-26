# Instalação Kubernetes com RKE + RANCHER + LestsEncrypt


# Procedimento executado em todos os nodes do cluster
## INICIO ##

**1 - Desabilitando o selinux:**
```
# setenforce 0
# vim /etc/selinux/config
SELINUX=disabled
```

**2 - Parando o firewall e desabilitando sua inicialização automática:**
```
# systemctl stop firewalld
# systemctl disable firewalld
```

**3 - Configuração de DNS**
```
# vim /etc/hosts
10.96.112.90 master1.example.com master1
10.96.112.91 worker1.example.com worker2
10.96.112.92 worker1.example.com worker3
```

**4 - Atualização do SO, Instalação de pacotes utilitários, adição do repositório docker e instalação da
versão 18.09.7:**
```
# yum -y update
# yum -y install yum-utils telnet rsync net-tools lvm2 wget vim curl bash-completion bash-completion-extras
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum install -y docker-ce-18.09.7-3.el7.x86_64
```


### OBS: A versão 18.09.7 foi instalada devido à estabilidade com a versão mais recente do Rancher.

**5 - Excluir o docker de atualizações não programadas**
`# echo exclude=docker* containerd* >> /etc/yum.conf`


**6 - Criação do usuário ‘k8s’ e o adicionamos nos grupos ‘wheel’ e ‘docker’**
```
# useradd k8s Senha: s@b1N@2020
# gpasswd -a k8s wheel
# gpasswd -a k8s docker
```

**7 - Configurar o SYSCTL para uso do docker, network bridges para o correto funcionamento do cluster
kubernetes. Para isso, foi criado um arquivo dentro de /etc/sysctl.d/ com essas
informações, e em seguida, foi executado o comando para carregar esse arquivo no kernel:**
```
# vim /etc/sysctl.d/99-k8s.conf
net.bridge-nf-call-ip6tables = 1
net.bridge-nf-call-iptables = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

**Recaregar as configurações**
`# sysctl --system`


**8 - O swap foi desabilitado conforme recomendação na documentação do Kubernetes:**
```
# swapoff -a
# sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```


**9 - Ative o NTP**
```
# yum install ntp -y
# systemctl enable ntpd
# systemctl start ntpd
```

**10 - Iniciando e habilitando a inicialização automática:**
```
# systemctl start docker
# systemctl enable docker
```

**11 - Reinicie os nodes**

### FIM 




## Procedimento executado apenas no node Master

**1 - Criar uma chave ssh e a replicação desta para os demais nodes. O procedimento precisa ser executado a partir do usuário ‘k8s’:**

**Gere um chave ssh com o usuário k8s**
```
$ su - k8s
$ ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/k8s -C rke (Nova Forma de gerar chave ssh)
$ ssh-keygen -f ~/.ssh/k8s
```

**Copie a chave ssh para todos os nodes do cluster**
```
# ssh-copy-id -i ~/.ssh/k8s  k8s@node01:
# ssh -i ~/.ssh/k8s k8s@node01
```

**2 - Remover a senha da conta k8s em todos os nodes**
`# passwd --delete k8s`

**3 - Desabite autenticação por senha do ssh e reinicie o serviço**
```
# vim /etc/ssh/sshd_config
  PasswordAuthentication no
# systemctl restart sshd
```


**2 - Baixe o RKE para Linux**
```
# wget https://github.com/rancher/rke/releases/download/v1.0.4/rke_linux-amd64
# wget https://github.com/rancher/rke/releases/download/v1.0.8/rke_linux-arm64
# mv rke_linux-amd64 rke
# chmod +x rke
# mv rke /usr/local/bin (como root)
# rke help
```




**3 - Baixe o KubeCTL**
```
# curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/linux/amd64/kubectl
# chmod 755 kubectl
# mv kubectl /usr/local/bin/
# kubectl version
```



**4 - 3 - Baixe o Kubeadm (OPICIONAL)**
```
# curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/
linux/amd64/kubeadm
# chmod 755 kubeadm
# mv kubeadm /usr/local/bin
# kubeadm version
```



5 - Instale o Helm na maquina de orquestração
```
# wget https://get.helm.sh/helm-v2.14.2-linux-amd64.tar.gz
# tar -xzvf helm-v2.14.2-linux-amd64.tar.gz
# mv linux-amd64/helm /usr/local/bin
# mv linux-amd64/tiller /usr/local/bin
# rm -rf linux-amd64
# rm -rf helm-v2.14.2-linux-amd64.tar.gz
# helm version
# tiller -version
```


6 - Centralizar os arquivos de configuração do cluster
```
$ mkdir cluster
$ cd cluster
$ touch cluster-sabin.yml

Inserido o conteúdo abaixo:

...
nodes:
  - address: 10.168.201.90
  user: k8s
  role: [controlplane,etcd]
  ssh_key_path: /home/k8s/.ssh/k8s
  hostname_override: kmaster1
  - address: 10.168.201.91
  user: k8s
  role: [controlplane,etcd]
  ssh_key_path: /home/k8s/.ssh/k8s
  hostname_override: kmaster2
  - address: 10.168.201.92
  user: k8s
  role: [controlplane,etcd]
  ssh_key_path: /home/k8s/.ssh/k8s
  hostname_override: kmaster3
  - address: 10.168.201.93
  user: k8s
  role: [worker]
  ssh_key_path: /home/k8s/.ssh/k8s
  hostname_override: kworker1
  - address: 10.96.112.94
  user: k8s
  role: [worker]
  ssh_key_path: /home/k8s/.ssh/k8s
  - hostname_override: kworker2

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h

network:
  plugin: weave
  options: {}
...
```



7 - Em seguida, execute o seguinte comando para dar início na instalação do cluster:
```
# rke up --config cluster-sabin.yml
# rke -d up --config cluster-sabin.yml (subir em modo debug)
# docker stats em todos os nodes para verificar
```

Saida deverá ser..
`# Finished building Kubernetes cluster successfully`



8 - Criando a variável de ambiente do Kubeconfig:
```
# export KUBECONFIG=/root/cluster/kube_config_cluster-sabin.yml
Para que essa variável torne-se permanente, foi inserido o comando no arquivo
/root/.bashrc conforme abaixo:



# vim /root/.bashrc
# mkdir ~/.kube
# cp ~/cluster/kube_config_cluster-sabin.yml ~/.kube/config
export KUBECONFIG=/root/cluster/kube_config_cluster-sabin.yml
# source <(kubectl completion bash)
# echo "source <(kubectl completion bash)"
# source ~/.bashrc
# kubectl get nodes
# kubectl get nodes,pods,services,deployments –A
```
## FIM ##



#### PREPARAÇÃO E INSTALAÇÃO DO RANCHER ############################################
############# Procedimento executado apenas no node Master #########################
## INICIO ##
1 - Criando um service account para o tiller em nosso cluster
`# kubectl -n kube-system create serviceaccount tiller`


2 - Criando uma ClusterRoleBinding para permitir acesso do tiller ao cluster.
`# kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`

3 - Inicializando o tiller
`# helm init --service-account tiller --output yaml | sed 's@apiVersion: extensions/v1beta1@apiVersion: apps/v1@' | sed 's@  replicas: 1@  replicas: 1\n  selector: {"matchLabels": {"app": "helm", "name": "tiller"}}@' | kubectl apply -f -`

4 - Verificando o status do deploy do tiller 
`# kubectl -n kube-system rollout status deploy/tiller-deploy`

A saída esperada é:
deployment "tiller-deploy" successfully rolled out

`# kubectl -n kube-system  get deploy -A`

## FIM ##







#########################  Instalação do Rancher propriamente dita ######################
## INICIO ##
- Configura o repositorio do rancher
`# helm repo add rancher-latest https://releases.rancher.com/server-charts/latest`

- Instala o cert-manager, necessário para gerenciar certificados letsencrypt ou internos da CA do propio rancher.
```
# kubectl create namespace cert-manager
# helm repo add jetstack https://charts.jetstack.io
# helm repo update
# kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.crds.yaml

# helm install --name cert-manager --namespace cert-manager --version v0.15.0 jetstack/cert-manager

# kubectl get pods --namespace cert-manager

# kubectl -n kube-system rollout status deploy/cert-manager
```

-- Instalacao do certificado gerado pelo proprio rancher
`# helm install rancher-latest/rancher --name rancher --namespace=cattle-system --set hostname=rancher.example.com`

## Fim ##





####################################################################################
Instalacao utilizando certificado local --
https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/helm-rancher/
https://rancher.com/docs/rancher/v2.x/en/installation/options/tls-secrets/




```
####################################### HA Proxy #####################################
global
	log   	127.0.0.1 local2
	chroot	/var/lib/haproy
	pidfile	/var/run/haproxy.pid
	user 	haproxy
	group 	haproxy
	daemon

defaults
	log global
	mode tcp
	option tcplog
	option tcp-check
	timeout connect 500
	timeout client 50000
	timeout server 50000


frontend k8s_frontend_http
  mode tcp
  bind 192.168.10.10:80
  default_backend k8s_backend_http


frontend k8s_frontend_https
  mode tcp
  bind 192.168.10.10:443
  default_backend k8s_backend_https



backend k8s_backend_http
  mode tcp
  server worker01 192.168.10.39:80 weight 1 maxconn 5000 inter 2s rise 2 fall 2 check
  server worker02 192.168.10.40:80 weight 1 maxconn 5000 inter 2s rise 2 fall 2 check



backend k8s_backend_https
  mode tcp
  server worker01 192.168.10.40:443 weight 1 maxconn 5000 inter 2s rise 2 fall 2 check
  server worker02 192.168.10.41:443 weight 1 maxconn 5000 inter 2s rise 2 fall 2 check

##########################################################################################

```



# Manutenção no Kubernetes
### backup DO Etcd do HA ##
Basta fazer backup do ETCd, realizando o procedimento no node master01
# rke etcd snapshot-save --name  <snapshot>.db --config rancher-cluster.yml


**Fazendo o restore do ETCD**
`rke etcd snapshot-restore --name  <snapshot>.db --config ./rancher-cluster-restore.yml`

### Renovando os Certs do cluster
# rke cert rotate --config cluster-jac.yml
