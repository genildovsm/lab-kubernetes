# Secrets e ConfigMaps

### Criando um secret do tipo Opaque

- Criando a string em base64 para o nome do usuário. O parâmetro `-n` não adiciona o caracter de `new line` à string gerada, evitando problemas na decodificação.

~~~sh
$ echo -n 'apiproduction' | base64
YXBpcHJvZHVjdGlvbg==
~~~

- Criando a senha em base64.

~~~sh
$ echo -n 'm!nh@S&nh4S3cret4' | base64
bSFuaEBTJm5oNFMzY3JldDQ=
~~~

- Criando o manifesto para o secret.

~~~yaml
apiVersion: v1
kind: Secret
metadata:
  name: meu-secret
type: opaque
data:
  username: YXBpcHJvZHVjdGlvbg==
  password: bSFuaEBTJm5oNFMzY3JldDQ=
~~~

Atribuindo valores à **variáveis de ambiente** de um **Pod** usando **secrets.**

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: meu-pod
  labels:
    app: meu-pod
spec:
  containers:
    - name: girafa
      image: nginx:stable-alpine
      env:
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: meu-secret
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: meu-secret
              key: password
~~~

Verificando as variáveis de ambiente no Pod.

~~~sh
$ kubectl exec meu-pod -ti -- sh -c 'env | grep "^\(USERNAME\|PASSWORD\)"'
USERNAME=apiproduction
PASSWORD=m!nh@S&nh4S3cret4
~~~

### Secret para autenticar no DockerHub

Logar no DockerHub.

~~~sh
$ docker login
~~~

- Após confirmar os caracteres exibidos no browser a autenticação será concluída.
- Será criado automaticamente o arquivo `~/.docker/config.json`, contendo a credencial de acesso.

Criar uma string em base64 com o conteúdo desse arquivo. Copiar a string gerada na saída padrão.

~~~sh
$ base64 ~/.docker/config.json
~~~

Criar o manifesto do secret.

~~~yaml
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: |
    lnPOhNJn9hOomiJomIOPjlKNĩohNkjbhkvbJGvbcJNUçhJikgbIUbjkBNChJnjnxovjmunYhBb6vbvRfgnKoHDFdvnMkLpUljHtgbvVBGHbngfgGjJbgggnjiJnbGfgBggMNmjkjnnNgxXXdvfvF
~~~

Manifesto do Pod com a configuração de **imagem privada do DockerHub.**

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-private-image
spec:
  containers:
    - name: my-container
      image: genildovsm/imagem-privada:1.0
  imagePullSecrets:
    - name: dockerhub-secret
~~~
