## Instalação cluster Kubernetes com KUBEADM


## Servidores Utilizados 

1. **192.168.56.15**
   - hostname: K8SHALB01
  
2. **192.168.56.10**
   - **hostname**: K8SMASTER01

3. **192.168.56.11**
   - **hostname**: K8SMASTER02
  
4. **192.168.56.12**
   - **hostname**: K8SWORKER01
  
5. **192.168.56.13**
   - **hostname**: K8SWORKER02

6. **192.168.56.251**
   - **hostname**: VMLASNPRD01


## Topologia cluster Kubernetes 



## Configuração do HAPROXY

Esta configuração é necessária para a comunicação entre os Servidores Masters, Workers e o API SERVER.

# Instalação e configuração do Haproxy

```bash
# Configuração da rede
nmcli con add con-name LAN type ethernet device enp0s8

#Adicionar IP 
nmcli con modify LAN ipv4.address 192.168.56.15/24 ipv4.method manual ipv4.dns 1.1.1.1,192.168.56.251

#Instalação do haproxy
yum install haproxy -y

#Arquivo de configuração do haproxy
/etc/haproxy/haproxy.cfg


#Principais configurações haproxy.cfg 

global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    option http-server-close
    #option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 5000
    compression algo gzip
    compression type text/plain text/css text/xml application/json application/javascript
    compression offload


#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------

#------------------------------#
#      FRONTEND KUBERNETES     #
#------------------------------#
frontend k8s_master
        #------------------#
        #        ACL       #
        #------------------#
        acl allowed_ips src 192.168.56.10
        acl allowed_ips src 192.168.56.11
        acl allowed_ips src 192.168.56.12
        acl allowed_ips src 192.168.56.13
        acl allowed_ips src 192.168.56.14
        #------------------#

        tcp-request connection accept if allowed_ips
        bind *:6443
        default_backend k8s_servers



backend k8s_servers
        mode tcp
        balance roundrobin
        server master1 K8SMASTER01:6443 weight 1 check inter 10s fall 3
        server master2 K8SMASTER02:6443 weight 1 check inter 10s fall 3

#Após relizar a configuração em haproxy.cfg, verificar sintaxe do arquivo
haproxy -f /etc/haproxy/haproxy.cfg -c

#Após verificar a sintaxe, reinicializar o serviço
systemctl restart haproxy
```

# Configurações globais 

- log **127.0.0.1 local2**: Define o endereço IP e o nível de log para onde o HAProxy enviará seus logs. Aqui, 127.0.0.1 é o localhost e local2 é um dos níveis de log do syslog.

- **chroot /var/lib/haproxy**: Define um diretório chroot para o HAProxy, limitando o acesso do processo ao diretório /var/lib/haproxy e seus subdiretórios.

- **pidfile /var/run/haproxy.pid**: Especifica o arquivo onde o PID (Process ID) do HAProxy será armazenado.

- **maxconn 4000**: Define o número máximo de conexões simultâneas que o HAProxy pode gerenciar.

- **user haproxy e group haproxy**: Define o usuário e grupo sob os quais o processo HAProxy será executado.

- **daemon**: Executa o HAProxy em segundo plano como um daemon.

- **stats socket /var/lib/haproxy/stats**: Habilita um socket UNIX para acesso às estatísticas e controle do HAProxy.

- **ssl-default-bind-ciphers PROFILE=SYSTEM e ssl-default-server-ciphers PROFILE=SYSTEM**: Define perfis de criptografia padrão para as conexões SSL/TLS, utilizando políticas de criptografia do sistema.

# Configurações defaults 

- **mode tcp**: Define o modo de operação como TCP, que é apropriado para balanceamento de carga de protocolos baseados em TCP.

- **log global**: Usa a configuração de log definida na seção global.

- **option tcplog**: Habilita o log de conexões TCP, o que é útil para monitoramento detalhado.

- **option dontlognull**: Não registra conexões que não geram tráfego (ex: conexões que terminam imediatamente após a conexão).

- **option http-server-close**: Fecha a conexão TCP após o servidor HTTP responder, em vez de mantê-la aberta para múltiplas requisições.

- **option redispatch**: Permite que o HAProxy redispatch conexões que falham após serem atribuídas a um servidor, tentando outros servidores disponíveis.

- **retries 3**: Define o número de tentativas de reenvio de uma solicitação em caso de falha.

- **timeout http-request 10s**: Tempo máximo permitido para que uma solicitação HTTP seja recebida completamente.

- **timeout queue 1m**: Tempo máximo que uma solicitação pode ficar na fila de espera antes de ser processada.

- **timeout connect 10s**: Tempo máximo para estabelecer uma conexão com o servidor backend.

- **timeout client 1m**: Tempo máximo para que uma solicitação seja completamente recebida do cliente.

- **timeout server 1m**: Tempo máximo para que uma resposta seja completamente recebida do servidor.

- **timeout http-keep-alive 10s**: Tempo máximo que uma conexão HTTP pode ser mantida aberta após uma resposta.

- **timeout check 10s**: Tempo máximo para verificar a saúde do servidor backend.

- **maxconn 5000**: Número máximo de conexões simultâneas permitido.

- **compression algo gzip**: Habilita a compressão usando o algoritmo gzip.

- **compression type text/plain text/css text/xml application/json application/javascript**: Define os tipos de conteúdo que serão comprimidos.

- **compression offload**: Permite que o HAProxy faça a compressão de dados antes de enviá-los ao cliente, descarregando esse trabalho do backend.

# Frontend Kubernetes

- **frontend k8s_master**: Define a seção frontend chamada k8s_master, que escuta por conexões TCP.

- **acl allowed_ips src ...**: Define uma lista de endereços IP permitidos para se conectar ao frontend.

- **tcp-request connection accept if allowed_ips**: Aceita conexões apenas dos IPs definidos nas ACLs.

- **bind *:6443**: Faz o HAProxy escutar na porta 6443 em todas as interfaces.

- **default_backend k8s_servers**: Define o backend padrão para este frontend, chamado k8s_servers.

# Backend Kubernetes 

- **backend k8s_servers**: Define a seção backend chamada k8s_servers.

- **mode tcp**: Define o modo de operação como TCP para o backend.

- **balance roundrobin**: Define o algoritmo de balanceamento de carga como round-robin, que distribui as conexões de forma uniforme entre os servidores.

- **server master1 K8SMASTER01:6443 weight 1 check inter 10s fall 3**: Define um servidor backend com o nome master1, que escuta em K8SMASTER01:6443, com um peso de 1 e uma verificação de saúde a cada 10 segundos. Se o servidor falhar 3 vezes consecutivas, ele será marcado como inativo.

- **server master2 K8SMASTER02:6443 weight 1 check inter 10s fall 3**: Define outro servidor backend com o nome master2, que escuta em K8SMASTER02:6443, com o mesmo peso e parâmetros de verificação.

## Instalação dos componentes do Kubernetes 

Nesta seção, todas as configurações serão repetidas em ambos os servidores. Como exemplo, usarei 192.168.56.10, mas você deve replicar essas configurações em ambos os servidores que fazem parte do cluster.

# Instalação e configurações de módulos 

```bash
#Desativando uso de swap no sistema
swapoff -a 

#Carregar módulos do Kernel essenciais para o funcionamento do kubernetes
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

#Habilitar módulos
modprobe overlay
modprobe br_netfilter

#Habilitar regras de firewalls em redes Bridge
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

#Atualiza todos parâmetros do Kernel
sysctl --system
```

# Instalação de componentes do Kubernetes 

Neste exemplo, foi realizado a instalação do kubeadm v1.30. Consulte a documentação oficial para mais detalhes. 

```bash
#Desabilitando Selinux
sudo setenforce 0

#Desabilitando Selinux ao iniicio do sistema
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#Adicionando repositórios oficiais

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

#Instalando os componentes
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

#Habilitar o kubelet após instalar o Kubeadm
systemctl enable --now kubelet
```

# Instalação do Docker 

```bash
#Instala os utilitários do yum
yum install -y yum-utils

#Adiciona o repositório oficial do Docker
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

#Instala todos os componentes do Docker
yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

#Inicializa o Docker
systemctl start docker

#Habilita o Docker ao boot do sistema
systemctl enable docker
```

# Configurando o Containerd com Cgroups

```bash
#Adiciona as configurações default ao config.toml
containerd config default | sudo tee /etc/containerd/config.toml

#Habilita o Cgroup do systemd para o Containerd
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

#Reinicializar o serviço
systemctl restart containerd
```

# Inicializar cluster kubernetes 

Após a configuração de todos esses passos, você deve ser capaz de inicializar o cluster. Neste exemplo, irei inicializar pela máquina 192.168.56.10(master1)

```bash
#Iniciliza o cluster em modo HA
kubeadm init --control-plane-endpoint "192.168.56.15:6443" --upload-certs --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=192.168.56.10
```

# Comandos para verificar seu cluster 

Após inicializar o cluster, irei dar uma breve explicação das informações do que foi feito

1. **--control-plane-endpoint**:
   - **Função**: IP que irá servir de "ponte" para comunicação do cluster como um todo.

2. **--apiserver-advertise-address**:
   - **Função**: Habilita de forma explicita o IP que ira expor as portas do api-server e etcd, para comunicação com os nós

3. **--pod-network-cidr**:
   - **Função**: Sub-rede utilizada pelos containers no cluster, normalmente utilizados pelo ClusterIP

```bash
#Verifica os nós
kubectl get nodes

#Verifica todos os pods
kubectl get pods -A

#Verifica todas namespaces
kubectl get ns

#Verifica os pods da namespace kube-system
kubectl get ns -n kube-system
```

## Detalhes da instalação 

Normalmente, você pode perceber que os nós não estão marcados como ativos após a instalação. Isso pode ocorrer devido à falta de um driver de rede. Na minha configuração, vou utilizar o Calico. Deixarei os arquivos na raiz deste projeto e farei atualizações futuras sobre como realizar a instalação.



