# Volumes no Kubernetes

## StorageClass

É uma forma de categorizar os tipos de armazenamento. O conceito de StorageClass do Kubernetes é semelhante aos "perfis" em alguns outros projetos de sistemas de armazenamento.

O **storageClass** a seguir é o **padrão do Kind.**

~~~sh
kubectl get sc 
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  5d23h
~~~

- Nome da StorageClass: standard  
- **(default)** indica que ela é a **StorageClass padrão do cluster**
- Será usada automaticamente quando um PVC não especificar storageClassName

<br>

**PROVISIONER**  
rancher.io/local-path

- Define quem cria o volume físico
    - Provisionador do Local Path Provisioner (comum em clusters Rancher, k3s, kind)
    - Cria volumes usando diretórios locais no nó
    - Não é storage distribuído (não é NFS, Ceph, EBS, etc.)

**Implicação importante:**

- O volume fica preso ao nó onde foi criado
- Se o pod mudar de nó, o volume não acompanha

<hr>

**RECLAIMPOLICY**  
Delete


- Define o que acontece com o volume físico (PV) quando o PVC é deletado
    - O PV e os dados são apagados automaticamente

- Alternativa comum:
    - Retain → o PV e os dados permanecem após o PVC ser excluído

>**Atenção:**  
Com Delete, os dados não são preservados

<hr>

**VOLUMEBINDINGMODE**  
WaitForFirstConsumer

- Controla quando o PV é criado e associado ao PVC
    - O volume só é provisionado quando um Pod for agendado
    - O Kubernetes escolhe o nó correto primeiro
    - Evita criar volumes em nós errados (crítico para storage local)

- Alternativa: Immediate 
    - PV é criado imediatamente ao criar o PVC

<hr>

**ALLOWVOLUMEEXPANSION**  
false

- Indica se é possível expandir o volume após criado
    - Não é possível aumentar o tamanho do PVC
    - Alterar spec.resources.requests.storage não terá efeito

Indicado para storage local simples

**Resumo:** 

- É a padrão do cluster
- Usa armazenamento local do nó
- Cria o volume apenas quando o Pod é agendado
- Não permite expansão
- Apaga os dados ao remover o PVC
- Não é indicada para workloads stateful críticos

Verificando os detalhes do StorageClass default.

~~~sh
kubectl describe storageclasses.storage.k8s.io standard 
Name:            standard
IsDefaultClass:  Yes
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"},"name":"standard"},"provisioner":"rancher.io/local-path","reclaimPolicy":"Delete","volumeBindingMode":"WaitForFirstConsumer"}
,storageclass.kubernetes.io/is-default-class=true
Provisioner:           rancher.io/local-path
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
~~~

### Criando o primeiro StorageClass

~~~yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
	name: armazem
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
~~~

## Persistent Volume (PV)

Exemplo de PV usando HostPath.

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  labels:
    storage: lento
  name: meu-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
  storageClassName: armazem
~~~

Exemplo de PV usando NFS

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  labels:
    storage: nfs
  name: pv-nfs
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.0.200
    path: /mnt/nfs
  storageClassName: armazem
~~~

## Persistent Volume Claim

O **PVC** é uma solicitação de armazenamento feita pelos usuários ou aplicativos no cluster Kubernetes. Ele permite que os usuários solicitem um volume específico, com base em tamanho, tipo de armazenamento e outras características. O PVC age como uma "assinatura" que reivindica um PV para ser usado por um contêiner. O Kubernetes tenta associar automaticamente um PVC a um PV compatível, garantindo que o armazenamento seja alocado corretamente.  

Todo PVC é associado a um Storage Class ou a um Persistent Volume. O Storage Class é um objeto que descreve e define diferentes classes de armazenamento disponíveis no cluster. Já o Persistent Volume é um recurso que representa um volume de armazenamento disponível para ser usado pelo cluster.

- Manifesto de deployment do Redis usando PVC.

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redis
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app:
    spec:
      containers:
        - image: redis
          name: redis
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
      volumes:
        - name: redis-data
          persistentVolumeClaim:
            claimName: meu-pvc
~~~

Manifesto de um Pod do nginx usando PVC.

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:stable-alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: meu-pvc
          mountPath: /usr/share/nginx/html
  volumes:
    - name: meu-pvc
      persistentVolumeClaim:
        claimName: meu-pvc
~~~
