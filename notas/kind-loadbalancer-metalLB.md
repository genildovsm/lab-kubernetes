### Como configurar no "kind v0.31.0 go1.25.5 linux/amd64", de modo que eu consiga acessar um pod que executa uma aplicação na porta 80 http no meu cluster, usando um IP da minha rede privada "192.168.0.0/24"?

A solução mais correta e comum é usar MetalLB dentro do cluster kind.

Vou te mostrar o caminho padrão, funcional e usado em produção local

## Solução recomendada: kind + MetalLB

**O que vai acontecer**

- O MetalLB vai simular um LoadBalancer
- Ele vai atribuir um IP da sua rede 192.168.0.0/24
- Você acessa o serviço direto:
👉 http://192.168.0.X

### 1. Criar o cluster kind (config básico)

Crie um arquivo kind-config.yaml:

~~~yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
~~~

Crie o cluster:

~~~sh
kind create cluster --config kind-config.yaml
~~~

### 2. Instalar o MetalLB

~~~sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
~~~

Espere os pods subirem:

~~~sh
kubectl get pods -n metallb-system
~~~

### 3. Descobrir o range da sua rede Docker

Rode:

~~~sh
docker network inspect kind
~~~

Procure algo assim:

~~~json
"Subnet": "172.18.0.0/16"
~~~

⚠️ Importante  

O IP do LoadBalancer precisa:
- Estar acessível pelo host
- Não conflitar com DHCP
- Estar na mesma rede do host

Se sua máquina realmente está em `192.168.0.0/24`, escolha um range livre, por exemplo:

~~~
192.168.0.240 - 192.168.0.250
~~~

### 4. Configurar o MetalLB para usar IPs da sua rede

Crie `metallb-config.yaml`:

~~~yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: private-network-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.0.240-192.168.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2adv
  namespace: metallb-system
spec: {}
~~~

Aplique:

~~~sh
kubectl apply -f metallb-config.yaml
~~~

### 5. Criar o Service (porta 80 HTTP)

Exemplo de Service para seu pod:

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: http-service
spec:
  selector:
    app: example
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
~~~

Aplicar:

~~~sh
kubectl apply -f service.yaml
~~~

Ver o IP atribuído:

~~~sh
kubectl get svc
~~~

Saída esperada:

~~~sh
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
http-service   LoadBalancer   10.96.12.34    192.168.0.241    80:XXXXX/TCP
~~~

Testar no navegador ou curl

~~~sh
curl http://192.168.0.241
~~~

#### Observações importantes

- Isso funciona apenas na rede local
- Não depende de NodePort
- Funciona para múltiplos serviços
- É a mesma abordagem usada em:
  - kind
  - kubeadm
  - bare-metal Kubernetes