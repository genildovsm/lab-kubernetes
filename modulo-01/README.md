# Módulo - 01

Instalando o **kubectl**, que é o programa usado para gerenciamento do cluster.

[Link da documentação](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

~~~sh
root@lab01:~# curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
~~~

Adicionar permissão de execução no arquivo baixado e mover para o diretório de binários que esteja no PATH.

~~~sh
root@lab01:~# chmod +x kubectl && mv kubectl /usr/local/bin/
~~~

Verificando a versão do kubectl instalada.

~~~sh
root@lab01:~# kubectl version
Client Version: v1.35.0
Kustomize Version: v5.7.1
The connection to the server localhost:8080 was refused - did you specify the right host or port?
~~~

## Instalando o KIND para criar o cluster kubernetes em contêiners

[Link da documentação](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

Download do arquivo binário, adicionar a permissão de execução e mover para o diretório de binários no PATH.

~~~sh
root@lab01:~# [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64 && chmod +x ./kind && mv ./kind /usr/local/bin/kind
~~~

Verificando a versão do kind.

~~~sh
root@lab01:~# kind version
kind v0.31.0 go1.25.5 linux/amd64
~~~

## Instalar o docker

~~~sh
root@lab01:~# curl -fsSL https://get.docker.com | bash
~~~

### Criando um cluster com a opção padrão

Dessa forma será criado um cluster com apenas 1 nó e esse será o control-plane. Por padrão o nome do contexto será **kind-kind**.

~~~sh
root@lab01:~# kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.35.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
~~~

Obtendo informações sobre os nós do cluster.

~~~sh
root@lab01:~# kubectl get nodes
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   3m15s   v1.35.0
~~~

Excluindo o cluster de testes criado anteriormente

~~~sh
kind delete cluster
~~~

### Criar um arquivo de configuração para deploy de um cluster multi nodes

[Documentação](https://kind.sigs.k8s.io/docs/user/configuration/)

~~~sh
root@lab01:~# mkdir kind
root@lab01:~# vim kind/config.yaml
~~~

Conteúdo do arquivo *config.yaml*

~~~
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: snoopzilla
nodes:
  - role: control-plane
  - role: worker
  - role: worker
~~~

Criar o cluster com base no arquivo de configuração

~~~sh
root@lab01:~# kind create cluster --config kind/config.yaml
Creating cluster "snoopzilla" ...
 ✓ Ensuring node image (kindest/node:v1.35.0) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-snoopzilla"
You can now use your cluster with:

kubectl cluster-info --context kind-snoopzilla

Thanks for using kind! 😊
~~~

Obtendo as informações sobre os nós do cluster.

~~~sh
root@lab01:~# kubectl get nodes
NAME                       STATUS     ROLES           AGE     VERSION
snoopzilla-control-plane   NotReady   control-plane   7m22s   v1.35.0
snoopzilla-worker          NotReady   <none>          6m12s   v1.35.0
snoopzilla-worker2         NotReady   <none>          6m13s   v1.35.0
~~~

