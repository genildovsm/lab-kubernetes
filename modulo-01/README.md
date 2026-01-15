# Módulo - 01

## Instalando o kubectl 

Programa usado no gerenciamento do cluster.

[Link da documentação](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

~~~sh
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
~~~

Adicionar permissão de execução no arquivo baixado e mover para o diretório de binários que esteja no PATH.

~~~sh
$ chmod +x kubectl && sudo mv kubectl /usr/local/bin/
~~~

Verificando a versão do kubectl instalada.

~~~sh
$ kubectl version

Client Version: v1.35.0
Kustomize Version: v5.7.1
The connection to the server localhost:8080 was refused - did you specify the right host or port?
~~~

### Adicionar o autocomplete 

Apenas para o usuário

~~~sh
$ echo 'source <(kubectl completion bash)' >>~/.bashrc
~~~

Para o sistema.

~~~sh
$ kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
$ sudo chmod a+r /etc/bash_completion.d/kubectl
~~~

Adicionar um alias para o comando `kubectl`.

~~~sh
$ echo 'alias k=kubectl' >>~/.bashrc
$ echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
~~~

## Instalando o KIND para criar o cluster kubernetes em contêiners

[Link da documentação](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

Download do arquivo binário, adicionar a permissão de execução e mover para o diretório de binários no PATH.

~~~sh
$ [ $(uname -m) = x86_64 ] && \
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64 && \
chmod +x ./kind && \
sudo mv ./kind /usr/local/bin/kind
~~~

Verificando a versão do kind.

~~~sh
$ kind version

kind v0.31.0 go1.25.5 linux/amd64
~~~

## Instalar o docker

~~~sh
$ curl -fsSL https://get.docker.com | sudo bash
~~~

### Criando um cluster com a opção padrão

Dessa forma será criado um cluster com apenas 1 nó e esse será o control-plane. Por padrão o nome do contexto será **kind-kind**.

~~~sh
$ kind create cluster

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
$ kubectl get nodes

NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   3m15s   v1.35.0
~~~

Excluindo o cluster de testes criado anteriormente

~~~sh
$ kind delete cluster
~~~

### Criar um arquivo de configuração para deploy de um cluster multi nodes

[Documentação](https://kind.sigs.k8s.io/docs/user/configuration/)

~~~sh
$ mkdir kind
$ vim kind/config.yaml
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
$ kind create cluster --config kind/config.yaml

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
$ kubectl get nodes

NAME                       STATUS     ROLES           AGE     VERSION
snoopzilla-control-plane   NotReady   control-plane   7m22s   v1.35.0
snoopzilla-worker          NotReady   <none>          6m12s   v1.35.0
snoopzilla-worker2         NotReady   <none>          6m13s   v1.35.0
~~~

Dados dos **pods** que estão no namespace `kube-system`.

- A opção `-o wide` exibe informações mais detalhadas.
- Para exibir todos os pods de todos os namespaces do cluster usar a opção `-A`.

~~~sh
$ kubectl get pods -n kube-system -o wide
~~~

Exibir todos os `deployments` do namespace kube-system. Pode usar a abreviação.

~~~sh
$ kubectl get deployments -n kube-system -o wide
~~~

**DICA:** Para listar todos os recursos e suas respectivas abreviações e particularidades, use o comando a seguir:

~~~sh
$ kubectl api-resources | less

NAME                                SHORTNAMES   APIVERSION                        NAMESPACED   KIND
bindings                                         v1                                true         Binding
componentstatuses                   cs           v1                                false        ComponentStatus
configmaps                          cm           v1                                true         ConfigMap
endpoints                           ep           v1                                true         Endpoints
events                              ev           v1                                true         Event
limitranges                         limits       v1                                true         LimitRange
namespaces                          ns           v1                                false        Namespace
nodes                               no           v1                                false        Node
persistentvolumeclaims              pvc          v1                                true         PersistentVolumeClaim
persistentvolumes                   pv           v1                                false        PersistentVolume
pods                                po           v1                                true         Pod
...
~~~

## Criando o primeiro POD

Criar um pod com nome giropos com base na imagem do nginx.

~~~sh
$ kubectl run --image nginx --port 8888 giropops
~~~

Exibindo informações do POD criado.

~~~sh
$ kubectl get po -o wide

NAME       READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
giropops   1/1     Running   0          18s   10.244.2.2   snoopzilla-worker2   <none>           <none>
~~~

Acessando o `pod` internamente.

~~~sh
$ kubectl exec -ti giropops -- bash

root@giropops:/# 
~~~

Deletando o `pod` com mínimo de delay possível.

~~~sh
$ k delete po giropops --now
~~~
