# Pods e Limites de recursos

Exibir detalhes de um `pod` específico no namespace `kube-system`

~~~sh
kubectl describe po etcd-snoopzilla-control-plane -n kube-system
~~~

Ou 

~~~sh
kubectl get po poseidon -o yaml
~~~

Todos os pods que possuem uma label chamada `run`.

~~~sh
kubectl get po -L run -o yaml
~~~

Útil para especificar várias labels no mesmo comando. 

>**-L, --label-columns=[]:**
>
>Aceita uma lista de rótulos separados por vírgulas que serão apresentados como colunas. Os nomes diferenciam maiúsculas de minúsculas.
>
>Você também pode usar várias opções de sinalização, como -L label1 -L label2...

### Kubectl attach e exec

Um `pod` pode conter 1 ou mais contêiners, então, para entrar em um contêiner específico utilizamos o comando a seguir. O parâmetro `-c` indica qual o conteiner será acessado.

~~~sh
kubectl attach my-pod -c my-conteiner -ti
~~~

Quando um container não possui um processo em seu entrypoint, ele cria e termina o container. Para conteiners que se enquadram na situação mencionada e que não possuem um tty por padrão como o alpine, segue um manifesto com as opções necessárias para mantê-lo em `Running`:

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: zeus
  labels:
    run: zeus
spec:
  containers:
  - name: zeus-1
    image: ubuntu
    resources:
      requests:
        cpu: 1
        memory: 128Mi
      limits:
        cpu: 1
        memory: 128Mi
    stdin: true
    tty: true
    command: ["bash"]
  dnsPolicy: ClusterFirst
  restartPolicy: Always
~~~

### Verificando erros ou mensagens nos logs

Verificar possíveis mensagens de erro durante a execução dos containers nos pods. Com o parâmetro `-f` executa semelhante ao comando `tail`.

~~~sh
kubectl logs poseidon -c poseidon -f
~~~

### Volumes

Criando um `pod` com um `volume` do tipo `emptyDir`. Este tipo de volume persiste as informações armazenadas **enquando o pod existir no mesmo local de origem**. Caso o pod seja removido, ou movido para outro local, o conteúdo do volume será perdido.

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  labels:
    run: webserver
spec:
  containers:
  - name: webserver-1
    image: nginx:alpine
    volumeMounts:
    - mountPath: "/giropops"
      name: primeiro-volume
    resources:
      requests:
        cpu: "500m"
        memory: "128Mi"
      limits:
        cpu: 1
        memory: "128Mi"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: primeiro-volume
    emptyDir:
      sizeLimit: "256Mi"
~~~

