# ReplicaSets, DaemonSets e Probes

>**Não é possível** fazer a criação de **ReplicaSets e DaemonSets** usando o comando `kubectl create`.

## ReplicaSets

O ReplicaSet é o controlador responsável por garantir a quantidade de réplicas configuradas no Deployment e é sempre gerenciado por ele. O ReplicaSet, por si só, não possui capacidade de gerenciar versões nem de aplicar estratégias de RollingUpdate nos Pods.  

Quando realizamos a atualização de uma versão de um Pod por meio de um Deployment, o próprio Deployment cria um novo ReplicaSet para assumir o gerenciamento das réplicas dos Pods atualizados. Ao final do processo de atualização, o Deployment reduz as réplicas do ReplicaSet antigo e mantém ativas apenas as do novo ReplicaSet. 

Entretanto, o ReplicaSet antigo não é removido, pois pode ser utilizado para realizar um rollback caso algo dê errado. Quando é necessário reverter uma atualização, o Deployment simplesmente passa a utilizar novamente o ReplicaSet anterior para gerenciar as réplicas dos Pods. Dessa forma, a cada atualização de versão realizada por um Deployment, um novo ReplicaSet é criado para gerenciar os Pods atualizados, enquanto o ReplicaSet anterior é mantido para possibilitar uma reversão, se necessário.

~~~yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  labels:
    app: nginx-app
  name: nginx-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 1
            memory: 128Mi
          requests:
            cpu: 500m
            memory: 64Mi
~~~

Se o replicaset for alterado e executado `kubectl apply -f my-replicaset.yaml`, os dados dos pods só serão atualizados se os pods forem `deletados`. Daí o replicaset irá recriar a quantidade de pods configurada com a nova especificação dos containers.

**Boa prática:**   
Sempre criar Deployment ao invés de ReplicaSet.

## DaemonSet

Objeto que cria um Pod que é executado em todos os nós do cluster, garantindo que teremos ao menos 1 réplica em execução em cada nó. Útil por exemplo quando queremos instalar um agente do datadog, um exporter do Prometheus, um agente do zabbix, execução de proxy de rede como kube-proxy ou wave net, execução de agentes de segurança como Sysdig.  
Quando um nó é adicionado ao cluster, o DaemonSet vai criar uma réplica desse Pod no novo nó.

~~~yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: node-exporter-daemonset
  name: node-exporter-daemonset
spec:
  selector:
    matchLabels:
      app: node-exporter-daemonset
  template:
    metadata:
      labels:
        app: node-exporter-daemonset
    spec:
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.10.2
        ports:
        - containerPort: 9100
          hostPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys

~~~
  
*Caso o número de pods não corresponda a quantidade de nodes do cluster, verificar se há alguma TAINT no node `control-plane` com essa restrição.*

Ignorando a Taint e permitindo que o pod do DaemonSet seja criado no node control-plane. **Recomendado apenas em caso de monitoração do node.**

~~~yaml
spec:
  template:
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
~~~

## Probes

Garante que os Pods estão executando corretamente.

### Liveness

Monitora se o pod continua respondendo

### Startup

Executa um teste no início da execução do Pod

### Readiness

Verifica se o pod já está pronto para receber requisição
