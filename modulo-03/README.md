# Deployment

Exemplo de deployment com estratégia tipo `RollingUpdate`, que atualiza os pods de forma incremental, respeitando a quantidade de pods adicionais e os que podem ficam off-line durante o processo de atualização.

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: rolling-update
  name: rolling-update
  namespace: narnia
spec:
  replicas: 5 
  selector:
    matchLabels:
      app: rolling-update
  strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 2
  template:
    metadata:
      labels:
        app: rolling-update
    spec:
      containers:
      - image: nginx@sha256:284999d967248984ccd9fffa669c0c49b923e00de28d4c43f5b8c621d881eb44
        name: narnia
        resources:
          limits:
            cpu: 1
            memory: 128Mi
          requests:
            cpu: 1
            memory: 64Mi
~~~

Com a opção de estratégia configurada para `Recreate` todos os pods são recriados `de uma única vez`. Útil quando a aplicação sofreu uma alteração significativa que não permita acesso a duas versões diferentes.

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-recreate
  name: deploy-recreate
spec:
  replicas: 5 
  selector:
    matchLabels:
      app: deploy-recreate
  strategy: 
    type: Recreate
  template:
    metadata:
      labels:
        app: deploy-recreate
    spec:
      containers:
      - image: nginx:stable-alpine3.23
        name: patriota
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 1
            memory: 128Mi
          requests:
            cpu: 1
            memory: 64Mi
~~~

**Dica:** Para acessar o help dos parâmetros disponíveis nos arquivos YAML.

~~~sh
kubectl explain deployments.spec.template.spec
~~~

## Rollout

### Status

Acompanha o status do rollout em tempo real, bloqueando até a conclusão (ou falha).

~~~sh
kubectl rollout status deployment/nginx
~~~

Saídas comuns:

- successfully rolled out
- waiting for deployment "nginx" rollout to finish
- progress deadline exceeded

Opções importantes:

~~~sh
--watch=true|false     # Acompanha continuamente (default: true)
--timeout=60s          # Tempo máximo de espera
~~~

Exemplo sem bloquear:

~~~sh
kubectl rollout status deployment/nginx --watch=false
~~~

### History

Exibe o histórico de revisões do recurso.

~~~sh
kubectl rollout history deployment/nginx
~~~

Saída típica:

~~~
REVISION  CHANGE-CAUSE
1         kubectl apply -f deploy.yaml
2         kubectl set image deployment/nginx nginx=nginx:1.25
~~~

Ver detalhes de uma revisão específica:

~~~sh
kubectl rollout history deployment/nginx --revision=2
~~~

### Undo

Faz rollback para uma versão anterior.

~~~sh
kubectl rollout undo deployment/nginx
~~~

Voltar para uma revisão específica:

~~~sh
kubectl rollout undo deployment/nginx --to-revision=1
~~~

O rollback cria uma nova revisão, não “reativa” exatamente a antiga.

### Pause

Pausa o processo de rollout, impedindo que novas alterações avancem.

~~~sh
kubectl rollout pause deployment/nginx
~~~

### Resume

Retoma um rollout pausado.

~~~sh
kubectl rollout resume deployment/nginx
~~~

### Restart

Recria todos os pods do deployment.

~~~sh
kubectl rollout restart deployment -n mynamespace nginx-deployment
~~~

## Scale

Altera o número de réplicas de um deployment

~~~sh
kubectl scale deployment -n mynamespace --replicas 3 nginx-deployment
~~~
