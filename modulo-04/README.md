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

~~~sh
kubectl explain deployments.spec.template.spec.containers.livenessProbe
~~~

Monitora se o pod continua respondendo. No exemplo a seguir o teste de health check é executado:

- testar conectividade TCP na porta 80 
- aguardar 10s antes do primeiro teste, para que o pod tenha tempo de carregar seu processo
- executyar o teste a cada 20 segundos
- considerar falha se não houver resposta em até 5 segundos (por tentativa)
- reiniciar o container após 3 falhas consecutivas

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx-deploy
  strategy: 
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 2
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - image: nginx:1.19.1
        name: nginx
        resources: 
          limits:
            cpu: 1
            memory: 128Mi
          requests:
            cpu: 1
            memory: 64Mi
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 20
          timeoutSeconds: 5
          failureThreshold: 3
~~~

Testando se a porta está aceitando conexão enviando apenas SYN ACK. O kubelet apenas tenta abrir uma conexão TCP na porta 80 do container.

- kubelet faz um SYN → SYN/ACK → ACK
- se a conexão for aceita → probe SUCESSO
- se falhar ou expirar → probe FALHA
- nenhuma requisição HTTP é enviada
- Como o tcpSocket não executa HTTP, o NGINX não gera entrada de log

~~~yaml
livenessProbe:
          tcpSocket:
            port: 80
~~~

**Onde eu posso ver falhas do livenessProbe?**

Os eventos ficam no Node, não no container:

~~~sh
kubectl describe pod <pod-name>
~~~

Ou nos eventos:

~~~sh
kubectl get events --sort-by=.metadata.creationTimestamp
~~~

**Forma correta de ver os logs no Kubernetes**

~~~sh
kubectl logs <pod-name> -c webapp
~~~

Em Kubernetes, logs pertencem ao container runtime, não ao filesystem do container.

**Onde os logs ficam de verdade**

O container não gerencia logs. O node gerencia.

Exemplo (containerd)

~~~
/var/log/pods/<namespace>_<pod>_<uid>/<container>/0.log
~~~

Ou

~~~
/var/log/containers/<pod>_<namespace>_<container>-<container-id>.log
~~~

**Rotação e limpeza:**

Docker

~~~json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
~~~

Containerd (/etc/containerd/config.toml)

~~~toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".containerd]
  max_container_log_line_size = 16384
~~~

A rotação é feita fora do Kubernetes, no Node.

### Startup

Executa um teste no início da execução do Pod (somente 1 vez no início)

~~~sh
kubectl explain deployments.spec.template.spec.containers.startupProbe
~~~

- Enquanto startupProbe estiver falhando:
  - liveness é ignorada
  - Evita restart loop em aplicações lentas

Regra principal da startupProbe

- Enquanto a startupProbe não tiver sucesso:
  - livenessProbe é IGNORADA
  - readinessProbe é IGNORADA
  - Apenas a startupProbe é avaliada

Isso evita restart loop durante inicialização lenta.

### Readiness

- Verifica continuamente se o pod está apto a receber requisição.

Ver parâmetros:

~~~sh
kubectl explain deployments.spec.template.spec.containers.readinessProbe
~~~

**Em caso de falha:** Readiness remove o Pod do tráfego, sem parar ou reiniciar o container.
