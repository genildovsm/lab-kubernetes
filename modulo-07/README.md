# StatefullSets e Services

Ao criar esse StatefulSet, os PVCs serão criados automaticamente e os PVs também, desde que exista um StorageClass default com provisionamento dinâmico.

~~~yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
          - containerPort: 80
            name: http
        volumeMounts:
          - name: nginx-persistent-storage
            mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata: 
      name: nginx-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
~~~

Para cada réplica do StatefulSet, o Kubernetes cria um PVC. **Com replicas: 3, serão criados:**

~~~
nginx-persistent-storage-nginx-0
nginx-persistent-storage-nginx-1
nginx-persistent-storage-nginx-2
~~~

Se não for informado o campo "storageClassName", então o Kubernetes usa automaticamente o StorageClass marcado como (default).

**Criação automática de PV**

- O PVC é criado imediatamente
- O StorageClass default possui um provisioner dinâmico
- O provisioner cria automaticamente:
  - um PV
  - já associado ao PVC
  - respeitando tamanho, accessMode e reclaimPolicy

- Não é necessário criar PV manualmente.

- **O PV não será criado automaticamente se:**
  - não existir StorageClass default
  - o StorageClass default não suportar provisionamento dinâmico
  - o storageClassName for explicitamente definido para um SC inválido

Nesse caso, o PVC ficará em Pending.

**Observação importante (CKA)**

Se o StorageClass usar: 

~~~
volumeBindingMode: WaitForFirstConsumer
~~~

- O PVC pode aparecer como Pending até o Pod ser agendado
- Depois do agendamento → PVC e PV ficam Bound

Cada Pod terá seu nome associado a um índice incrementado

~~~sh
kubectl get pods

NAME      READY   STATUS    AGE
nginx-0   1/1     Running   30s
nginx-1   1/1     Running   25s
nginx-2   1/1     Running   22s
~~~

**Cada Pod monta seu próprio volume exclusivo:**

| Pod | PVC montado |
|-- |-- |
| nginx-0 | nginx-persistent-storage-nginx-0 |
| nginx-1 | nginx-persistent-storage-nginx-1 |
| nginx-2 | nginx-persistent-storage-nginx-2 |

## Service Headless

Definição de um Service Headless, usado normalmente com StatefulSet.

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: http
  clusterIP: None
  selector:
    app: nginx
~~~

- **CLUSTER-IP = None** → confirma que é **headless**
- Não há load-balancing via IP virtual
- O Service apenas **expõe endpoints DNS**

~~~sh
$ kubectl describe svc nginx

Name:              nginx
Namespace:         default
Labels:            app=nginx
Selector:          app=nginx
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>
~~~

- **Endpoints ainda vazios,** porque nenhum Pod com label `app=nginx` **existe ainda.**

#### Situação após o StatefulSet estar rodando

Supondo que o StatefulSet `nginx` (3 réplicas) já esteja ativo:

~~~sh
$ kubectl get endpoints

NAME    ENDPOINTS                                      AGE
nginx   10.244.0.12:80,10.244.0.13:80,10.244.0.14:80   30s
~~~

Cada IP corresponde a um Pod: nginx-0, nginx-1 e nginx-2.

#### DNS gerado automaticamente (ponto-chave)

| Nome DNS | Resolve para |
|-- |-- |
| nginx-0.nginx.default.svc.cluster.local | IP do Pod nginx-0 |
| nginx-1.nginx.default.svc.cluster.local | IP do Pod nginx-1 |
| nginx-2.nginx.default.svc.cluster.local | IP do Pod nginx-2 |

E também:

~~~
nginx.default.svc.cluster.local
→ lista de IPs dos Pods
~~~

## Services

Os **services** utilizam as **labels** como referência para **redirecionamento das requisições**.

Tipos de services: 
- ClusterIP 
  - Expõe o Service em um IP interno no cluster. Este tipo torna o Service acessível apenas dentro do cluster.
- NodePort 
  - Expõe o Service na mesma porta de cada Node selecionado no cluster usando NAT. Torna o Service acessível de fora do cluster usando
- LoadBalancer 
  - Cria um balanceador de carga externo no ambiente de nuvem atual (se suportado) e atribui um IP fixo, externo ao cluster, ao Service.
- ExternalName
  - Mapeia o Service para o conteúdo do campo externalName (por exemplo, foo.bar.example.com), retornando um registro CNAME com seu valor.

### NodePort

Criando um deployment via linha de comando e expondo os pods de forma simples.

~~~sh
$ kubectl create deploy --image nginx:stable-alpine --replicas=1 webapp
$ kubectl expose deployment webapp --port 8888
~~~

Verificando o serviço criado pela opção expose. O tipo padrão quando não especificado é **ClusterIP.**

~~~sh
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    54m
webapp       ClusterIP   10.96.74.121   <none>        8888/TCP   6s   <------ service criado.
~~~

