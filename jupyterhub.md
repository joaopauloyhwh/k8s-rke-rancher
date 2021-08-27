# Stack - Rancher + Kubernetes + JupyterHUB + SSL + LDAP 

### Servidores:

* node2000

* node2001

* node2002

* node2003 (rancher server)


# INÍCIO 

## 1 - Desativar o SWAP - Em Todos os Nodes do cluster

**Edite o arquivo /etc/fstab e remova (ou comente) a linha do Swap:**
```
$ sudo vim /etc/fstab
....
#/dev/mapper/centos-swap swap swap defaults 0 0
```

**Desative o Swap, sem precisar reiniciar o servidor:**

`$ sudo swapoff -a` 

**Confirme se o Swap está desativado no servidor:**

`sudo swapon -s`



## 2 - Instalar o docker em todos os nodes do cluster

**Pode ser utilizado o gerenciador de pacote zypper ou via script**

```
$ zypper in docker
or
$ sudo curl https://releases.rancher.com/install-docker/19.03.sh | sh
$ sudo usermod -aG docker suporte 

```


## 3 - Configuração do proxy no docker em TODOS os hosts

**Crie o direrotio .docker e o arquivo config.jason no home do usuário**

```
$ mkdir .docker
$ vim .docker/config.json

---
{
 "proxies":
 {
   "default":
   {
     "httpProxy": "http://proxycorporativo.empresa.com.br:3128",
     "httpsProxy": "http://proxycorporativo.empresa.com.br:3128",
     "noProxy": "localhost,127.0.0.0/8,10.220.40.0/24,*.homologacao.com.br,.homologacao.com.br"
   }
 }
}
---
```


**Configure o proxy no "sistema operacional"**

```
$ sudo vim /etc/sysconfig/proxy

## Path:        Network/Proxy
## Description:
## Type:        yesno
## Default:     no
## Config:      kde,profiles
#
# Enable a generation of the proxy settings to the profile.
# This setting allows to turn the proxy on and off while
# preserving the particular proxy setup.
# 
PROXY_ENABLED="yes"**

 

## Type:        string
## Default:     ""
#
# Some programs (e.g. lynx, arena and wget) support proxies, if set in
# the environment. 
# Example: HTTP_PROXY="http://proxy.provider.de:3128/"
HTTP_PROXY="http://proxycorporativo.empresa.com.br:3128"

 

## Type:        string
## Default:     ""
#
# Some programs (e.g. lynx, arena and wget) support proxies, if set in
# the environment. 
# This setting is for https connections
HTTPS_PROXY="http://proxycorporativo.empresa.com.br:3128"

 

## Type:        string
## Default:     ""
#
# Example: FTP_PROXY="http://proxy.provider.de:3128/"
#
FTP_PROXY="http://proxycorporativo.empresa.com.br:3128"

 

## Type:        string
## Default:     ""
#
# Example: GOPHER_PROXY="http://proxy.provider.de:3128/"
#
GOPHER_PROXY=""

 

## Type:        string
## Default:     ""
#
# Example: SOCKS_PROXY="socks://proxy.example.com:8080"
#
SOCKS_PROXY=""

 

## Type:        string
## Default:     ""
#
# Example: SOCKS5_SERVER="office-proxy.example.com:8881"
#
SOCKS5_SERVER=""

 

## Type:        string(localhost)
## Default:     localhost
#
# Example: NO_PROXY="www.me.de, .do.main, localhost"
#
NO_PROXY="localhost, 127.0.0.1,.homologacao.com.br,.empresa.com.br"
---
```



## 4 - Instalar o rancher-server via docker no node node2003

**Vamos usar as opções "-v" para persistir os dados e "-p" para o mapeamento das portas 80 e 443**

```

$ sudo docker run -d --name rancher --restart=unless-stopped -p 80:80 -p 443:443 -v /opt/rancher:/var/lib/rancher --privileged rancher/rancher:latest

```

- **Acessar o rancher-server via web (https://node2003.homologacao.com.br/), defina uma senha para o usuário admin e defina a url de acesso**


## 5 - Provisionar e configurar o Cluster Kubernetes com RKE

**Entrar no rancher server e selecionar a opção *Add Cluster***

figura1

**Selecione a opção *Existing nodes***

figura2

**Coloque o nome do Cluster**

figura3

**Desabilite o Ingress Nginx e clique en Next**

figura4

**Selecione os serviços *etcd*, *Control Plane* e *Worker* para executar em todos os nodes do cluster. Copie o código gerado pelo rancher e clique em Done**

figura5


- **Execute o comando gerado em TODOS os nodes do cluster node2000, node2001 e node2002.**

```
Exemplo:
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.5.9 --server https://node2003.homologacao.com.br --token rnsl92dn2wd785598d5nd2dp66v52brcq2jg4dbj4hvtlxm7cscmk5 --ca-checksum 8f576d7fc8cd22e7212eede591ad2aea244405543416fdfde36152a8fa3d625b --etcd --controlplane --worker

```


## 6 - Instalar e configurar o kubectl no rancher-server (Opcional)

**Instalar o kubectl no no host do node2003**

```

$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

```

**Copiar o kubeconfig no rancher-server *Cluster Local***

figura6

figura7


**Depois de copiar o kubeconfig no rancher-server criar o arquivo config e inserir o conteúdo**

```
$ mkdir .kube
$ vim ~/.kube/config
$ kubectl get nodes
```

[Referencia: Instalação do kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)


## 7 - Instalar e configurar o Longhorn 

**Obs: No SUSE, Instalar o pacote open-iscsi como pré-requisitos em TODOS OS NODES DO CLUSTER**

```
# zypper in open-iscsi
# systemctl enable --now iscsid
# systemctl status iscsid

```

**Faça os seguintes passos para a instalação do longhorn:**

1. Selecione o cluster **jupter-hub**
2. Selecione o Projeto/namespace **default** 
3. Selecione **Apps**
4. Clique em Launch e faça a busca pelo **longhorn**

figura8


## 8 - Instalar e Configurar o MetalLB (Se o ambiente já estiver usando o balanciador, o metalLB não é necessário)

**Instalação via manifest**

```

$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml

```

**Criar o arquivo de configuração do metalLB como no exemplo abaixo**

```

$ vim metallb.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
---
```

**Aplicar a configuração do configMap do MetalLB**

`kubectl apply -f metallb.yaml`



      
**Realizar um teste no metalLB**

**OBS: Depois remover o deployment e service de teste**

```
kubectl create deployment my-nginx-metalllb --image=nginx
kubectl expose deployment my-nginx-metalllb --port 80  --type LoadBalancer
kubectl get svc
````


[Referencia: Instalação do MetallLB](https://metallb.universe.tf/installation/)

[Referencia: Configuração do MetalLB](https://metallb.universe.tf/configuration/)


## 9 - Instalar e configurar o Ingress com o traefik - DNS 

**Instalação via manifest**

```
$ kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v1.7/examples/k8s/traefik-rbac.yaml
$ kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v1.7/examples/k8s/traefik-ds.yaml
```


**Crie e aplique o service e ingress para acessar o traefik:**

```

$ vim svc-ingress.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik.empresa.com.br
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web
---

```
**Aplique o manifesto**

```
$ kubectl apply -f svc-ingress.yaml
$ kubectl get svc -n kube-system
```

**OBS: Editar o serviço do traefik (traefik-web-ui) para  LoadBalancer**

`$ kubectl edit svc traefik-web-ui -nkube-system`


[Referencia: Instalação do traefik](https://doc.traefik.io/traefik/v1.7/user-guide/kubernetes/)



## 10 - Instalar e configurar o Helm 

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
$ chmod 700 /home/usuario/.kube/config
```

[Referencia: Instalação do helm](https://helm.sh/docs/intro/install/)


## 11 - Instalar e Configurar o Jupyter HUB via Helm

**Adicionar o repositorio do jupyterhub no helm**

`$ helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/`

**Atualizar o repositorio**

`$ helm repo update`

**Baixar o conf do jupterhub e realizar as configurações necessárioas antes do deployment**

`$ helm show values jupyterhub/jupyterhub > config.yaml` 



[Referencia: Instalação do Jupterhub]https://zero-to-jupyterhub.readthedocs.io/en/latest/jupyterhub/installation.html


 
## 12 - Criar o certifica echave para configuração do SSL,criar o arquivo "openssl.cfg"

```

$ vim openssl.cfg
 ---
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
[req_distinguished_name]
C = BR
ST = ESTADO
L = CIDADE
O = Empresa de TI
OU = Area de Infraestrutura
CN = *.homologacao.com.br
[v3_req]
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.homologacao.com.br
---
```


**Depois gera o certificado com a chave**

`$ openssl.exe req -nodes -sha256 -newkey rsa:2048 -keyout "jupyter.key" -out "jupyter.csr" -config "openssl.cfg"`


**Solicitar assinatura do certificado**



## 13 - Habilitando o SSL no Jupyterhub

**Edite o arquivo config.yaml e proucure a entrada "https" e cole o conteudo do certificado e chave para habilitar o SSL e salve o arquivo**

```
$ vim config.yaml
---
 https:
    enabled: true
    type: manual
    manual:
      key: |
        -----BEGIN RSA PRIVATE KEY-----
        MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDOHM7ac2vhGVT3
        ..............
        AaHp/xI1M7wt7nyRuRsasDSOFZZ09mftOj3aTvJnc7602HgsvvObJ3wPcEWNNPaV
        8Bx4D0KoN5wLtT5wAmjj8Uwy
        -----END RSA PRIVATE KEY-----
      cert: |
        -----BEGIN CERTIFICATE-----
        MIIGpDCCBYygAwIBAgITOAACgDeXJXCr1Ir/7QABAAKANzANBgkqhkiG9w0BAQsF
        ...........
        B31vgNdg4GD7IxvheNDaGRFUH5K0ZcUI
        -----END CERTIFICATE-----
---
```


## 14 - Configuração do LDAP no jupyterhub

**Edite o arquivo config.yaml e proucure a entrada "hub" e a subentrada "config" e realize a configuração de acordo com o ambiente e salve o arquivo**

**Obs:altere a entrada "service" para type: "LoadBalancer"

```
$ vim config.yaml
---
hub:
  config:
    JupyterHub:
      authenticator_class: ldapauthenticator.LDAPAuthenticator
    LDAPAuthenticator:
      bind_dn_template:
        - cn={username},OU=GECOQ,OU=Usuario empresa,DC=homologacao,DC=com,DC=br
      escape_userdn: False
      lookup_dn: True
      lookup_dn_search_filter: ({login_attr}={login})
      lookup_dn_search_password: XXXXXXXXXXXX
      lookup_dn_search_user: CN=SVC-ldap,OU=Usuarios de Servico,DC=homologacao,DC=com,DC=br
      lookup_dn_user_dn_attribute: cn
      server_address: ldap.homologacao.com.br
      user_attribute: sAMAccountName
      user_search_base: dc=homologacao,dc=com,dc=br
---
```


## 15 - Depois dos ajustes no arquivo config.yaml do jupyterhub, execute a instalação

**Realize o deployment com a criação do namespace jupyterhub e utilize o arquivo config.yaml com os ajustes de "SSL" e "LDAP"**

```
$ helm upgrade --cleanup-on-fail \
  --install jupyterhub jupyterhub/jupyterhub \
  --namespace jupyterhub \
  --create-namespace \
  --version=1.1.1 \
  --values config.yaml
```

**Verifique os pods e services do Jupyterhub**

```
$ kubectl get pod --namespace jupyterhub
$ kubectl get service --namespace jupyterhub
```


## 16 - Configuração do ingress para acessar a aplicação web do Jupyterhub

```
$ touch ingress-jupyterhub.yaml

$ vim ingress-jupyterhub.yaml

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jupyter-ingress
  namespace: jupyterhub
spec:
  rules:
  - host: jupyter.homologacao.com.br
    http:
      paths:
      - path: /
        backend:
          serviceName: proxy-public
          servicePort: 80
---

$ kubectl apply -f ingress-jupyterhub.yaml 
$ kubectl get ing -n jupyterhub 
```



# FIM 
