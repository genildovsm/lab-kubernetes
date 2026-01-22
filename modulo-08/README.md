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

## Secrets para armazenar certificado e chave TLS

Criar o `certificado` e a `chave privada`, que serão utilizados em um `pod do nginx`.

~~~sh
$ openssl req -x509 -nodes -days 365 -newkey rsa:2080 -keyout chave-privada.key -out certificado.crt
~~~

Criar o secret do tipo TLS.

~~~sh
$ kubectl create secret tls meu-secret-tls --cert=certificado.crt --key=chave-privada.key
~~~

Verificando o conteúdo do secret criado.

~~~sh
$ kubectl get secrets meu-secret-tls -o yaml

apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR4ekNDQXF1Z0F3SUJBZ0lVQ3gwUm5vcWRFYndMWGRqWGxVblF5SFBnUGdJd0RRWUpLb1pJaHZjTkFRRUwKQlFBd2J6RUxNQWtHQTFVRUJoTUNRbEl4RWpBUUJnTlZCQWdNQ1ZOaGJ5QlFZWFZzYnpFVk1CTUdBMVVFQnd3TQpVSEpoYVdFZ1IzSmhibVJsTVJBd0RnWURWUVFLREFkTmVTQkliMjFsTVNNd0lRWUpLb1pJaHZjTkFRa0JGaFJuClpXNXBiR1J2ZG5OdFFHZHRZV2xzTG1OdmJUQWVGdzB5TmpBeE1qRXlNelUzTXpKYUZ3MHlOekF4TWpFeU16VTMKTXpKYU1HOHhDekFKQmdOVkJBWVRBa0pTTVJJd0VBWURWUVFJREFsVFlXOGdVR0YxYkc4eEZUQVRCZ05WQkFjTQpERkJ5WVdsaElFZHlZVzVrWlRFUU1BNEdBMVVFQ2d3SFRYa2dTRzl0WlRFak1DRUdDU3FHU0liM0RRRUpBUllVCloyVnVhV3hrYjNaemJVQm5iV0ZwYkM1amIyMHdnZ0VtTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElCRXdBd2dnRU8KQW9JQkJRRG9CUGhHVTUvTUhabm53a2JQTWNJV3VFbXQ2bml5VmhFWEFORE5BOTVrblppZUYyNUhTZjV4SkRwLwp5Z1FnUThaK0M1ODdRWHByTGpCNnlQdmxVdXRva3RqbWtvUTEzc1k4MlJmNHVoU3RXbVgyUEQ4NXhLVWtHRmNWClZCZGJNa2l4bzBSTWp4V0VwY29ubjAvZm1IdWlOWjdMZ0VZRk1KNWpkSVplWUpEdE1XV1MveGV0T0x1aHpsQmQKdG96eVdudHRjMXRpaEtqQ3U2dEJkWmdXRHBPN3hVc214TkFSZ2h1OWZ2VXQ0L0RCUGxXcEE2L1JYckVTV2taaApROHhUb1RIL080Yms3QnFqcGMrL1M5am85Z0MyUVA2TVNtQXhHdW8zenY2QnV5UUhpaXFLY215YzBkVFAxRjRjCjArRE5mKzM1REVxZmtlYTVhUHo2bFdZaGtqUXZkd0RvbXdJREFRQUJvMU13VVRBZEJnTlZIUTRFRmdRVTRVSVcKNUJGbDFpc083N3RMc1paMFRpay9UUEF3SHdZRFZSMGpCQmd3Rm9BVTRVSVc1QkZsMWlzTzc3dExzWlowVGlrLwpUUEF3RHdZRFZSMFRBUUgvQkFVd0F3RUIvekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUVVBbExGYWN6OU1MMkp4CmZ3MFZRNmVrNWpTd2lYcktzYURHbzFCL01hQTFEb2FPdVVLRG1FQStwZ09YSzhrV2FDSThJbDdMMFk1eGpUaGgKZGluL3ZXaEc1YmxrNERuZU9ZQy9PdjV3WU02UjAvdGZaekNmbUllNXhRaG5rWU1uQUlobGhmelU4Sm52TW94bwo4YVJ2eWwzNkFkdUMzU1EzakR0UTZiZm90MG4xc0NwQmRGZU1uOTZtc2grSC9xeDNteGw1bnB0b2dMbGVHWElLCk82NEh3VXo1UExGUURNYnNQa3NNOE5reVBjNGRUL2E4Z3R3Rkd2ZlFsYnpYNDUrS3VzSURKc1hVRlFBU2VBSmYKeFBST3lBRGlNdmVOaWJjV2ZobTJRK2tOMXdIdVBlY2toNGpGaW9NVlc1dmVJY0doMjZIQmJkUVcvM0Ywb3YvUQozczhaVHAzZXBJSXA5Y1E9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV6d0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQkxrd2dnUzFBZ0VBQW9JQkJRRG9CUGhHVTUvTUhabm4Kd2tiUE1jSVd1RW10Nm5peVZoRVhBTkROQTk1a25aaWVGMjVIU2Y1eEpEcC95Z1FnUThaK0M1ODdRWHByTGpCNgp5UHZsVXV0b2t0am1rb1ExM3NZODJSZjR1aFN0V21YMlBEODV4S1VrR0ZjVlZCZGJNa2l4bzBSTWp4V0VwY29uCm4wL2ZtSHVpTlo3TGdFWUZNSjVqZElaZVlKRHRNV1dTL3hldE9MdWh6bEJkdG96eVdudHRjMXRpaEtqQ3U2dEIKZFpnV0RwTzd4VXNteE5BUmdodTlmdlV0NC9EQlBsV3BBNi9SWHJFU1drWmhROHhUb1RIL080Yms3QnFqcGMrLwpTOWpvOWdDMlFQNk1TbUF4R3VvM3p2NkJ1eVFIaWlxS2NteWMwZFRQMUY0YzArRE5mKzM1REVxZmtlYTVhUHo2CmxXWWhralF2ZHdEb213SURBUUFCQW9JQkJEcVFOcEtadlBYcWF0U0N4eGk1T0lJL2xlbGVDNFVJRUZ3OENuZ1YKQitkaG1Bb2ZZK2grbHNpOEdqL3pIOE45Ri9ic3ZBNWE0cWwzQ1NtVTNXL3UxQmliS0VCYXJ5QmgwS3UvS0R2Ugp6REpOWlBzWURlVm82ejRISHNQMWE4ZkxFMm96Q2FSQllXOFA0Y3pLZTRDMm5rNDlObzJySFFGbVdqUkVUejQzCkpaMlpJRWZWSTgwcWNpTzl3a3NuWlRLTlRZaTNWSk5lbXJOcFIzN0owQS9lNDBXYTN5cEtRWlpmcHhkMkY3QTAKdTIzMVNzWmhzRkpPWDlTNnQ1N3dMVTBsZ3JoWUNIY1VJeGNoRWdLa2lKNTkyUmlTc1dteUhjeVRaRTZpdzg1aApJYnA5dFNBZzBxcEZYczhEaWF3M1B1SVZ6U1B4NTVnM3AwN2VEU29DLzRPc2dNZGpvOWpSQW9HREFQcGF0eW56Cm51YkRoYWEvZmJ1M2UvVllucFpNVnl5TG53R293d09SV1RNc29UV3FUcStOSUNia25GaVNnTWpxdlVZZ2lCMXcKemFwN01qenJaQXJBN1VRWkRZNHlJZFdiMWNkeGQ4NmVhS3VxTGlOUHd3V2JBUnliK2p0Y3Npek1mZDEwY3E2ZwpHdStVTng3MVd2bjVNd2RTNEJQUU4yWDVwOS94enpxUEJ5RkJzUzhDZ1lNQTdVQm9VN0NyeHd6d0xZSzFYTzRoClJKdFFob3hzKzYvOXpTK1ZxcnQvbWVCS2l5Z2FoUmY5dlZuclJDOXhRZzdJZ3EzVjIxdEZtMVJONDM5VFJScG8KZnN4cGN0d2JYSDBHSzMzbG5mOGd2T0xGQ3UvanBrMjZRVS9vNDVoNmk5VGY0NE1XeXRLVm1Lc1N5Zk1pT1hTNwpJNlhaTWFVZjErMlFMUTd4N1prbWFOTXNWUUtCZ2trNzdpYWNlRmdpeTk3cVZ6cHBReDZURE5rRWZkK3UvQlY5Cks0YklwdUk4WlBBUTRMR2p3OHI4eHV0MTk2eE9WbzNFQ0cwc1NVMWNlbWF0cVBjb1ZuKzhJR1gvTGp5Uk9HaisKUFVDNHYvK3ZhWTIwMEdTOFlnZmZiTVNlcWhSR3dXN2RtSXFTbFM2T0djMjVraUpiamx6UEZuTlZUazlMUjV0UAozZ0hRUXhLc1o4c0NnWU1BZ0dmUmp5b1pibnYwS2MyS2R5ZHkzZnpwa2tqQ1cxNGZFVVJsenFmNEljSWcxanY0ClRueHptbDNtVlZzUUExNlk2eEZHbzVnOGpoc01wTW91dVVIWHVIak53WnFiUEcxMlAyZStOTXIyWHdTay9JeGwKTzRicC9adFFRbzR1RlN3N21KbEVacldldmFncFhSKzRNRHliWkduSXFYUGpUaXlIVWJ1NitJdGhIRzdlbVFLQgpnZ0RFb2lDWlM2M1NVUEoyZFI0TmxhdU9yeTkwdFpJUnBnZ3RjbmMvbG1PT0cvWDVPWjhVMHdIQmJjdC9RbVRiCmU2dnkvcjlxaGladjZpbHlZNzZTSDRITmpaOFBvUjMwaDhtQ1kyLzlWcU5DSFpMQit1NEc5T0NrWjEySzJLamYKQkswUkUxYUlNREs1dDlxOGdSTHMyVzJBQitCQXhLR0dCaVdHcWZHL1BnZ240K009Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K
kind: Secret
metadata:
  creationTimestamp: "2026-01-22T00:02:58Z"
  name: meu-secret-tls
  namespace: default
  resourceVersion: "325620"
  uid: c30a5daa-2272-4590-9bd4-470d4e629bc2
type: kubernetes.io/tls
~~~

O conteúdo da chave e do certificado foram convertidos em base64 pelo comando de create.

### ConfigMaps

- Guarda informações relacionadas a configurações, para que possam ser utilizadas nos containers. **Não são codificadas em base64.**

Criar o arquivo de configuração `nginx.conf`  que será utilizado pelo container do nginx.

~~~nginx
http {
	server {
		listen 80;
		listen 443 ssl;
		ssl_certificate /etc/nginx/tls/certificado.crt
		ssl_certificate_key /etc/nginx/tls/chave-privada.key

		location / {
			return 200 'Olá mundo!\n';
			add_header Content-Type text/plain;
		}
	}
}
~~~

Criar o **configmap** com base no arquivo `nginx.conf`.

~~~sh
$ kubectl create configmap nginx-config --from-file nginx.conf
~~~

Manifesto `configmap.yaml` criado manualmente.

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: default
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        listen 443 ssl;

        ssl_certificate /etc/nginx/tls/certificado.crt;
        ssl_certificate_key /etc/nginx/tls/chave-privada.key;

        location / {
          return 200 'Olá mundo!\n';
          default_type 'text/plain; charset=utf-8';
        }
      }
    }
~~~

Criar o pod onde o secret tls e o configmap serão usados.

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap
  labels:
    app: nginx
spec:
  containers:
  - name: girafa
    image: nginx:stable-alpine
    ports:
    - containerPort: 80
    - containerPort: 443
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    - name: nginx-tls
      mountPath: /etc/nginx/tls
  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-config
  - name: nginx-tls
    secret:
      secretName: meu-secret-tls
      items:
      - key: tls.crt
        path: certificado.crt
      - key: tls.key
        path: chave-privada.key
~~~

Expondo a porta do pod criando um service, para que seja acessado de fora do cluster.

~~~sh
$ kubectl expose pod pod-configmap
~~~

Criando um redirecionamento para o clusterIP criado com o comando anterior, de modo que seja possível acessar o pod de fora do cluster.

~~~sh
$ kubectl port-forward services/pod-configmap 4443:443
~~~

`kubectl port-forward`  
Cria um túnel entre a máquina local e um recurso do Kubernetes (Pod, Service ou Deployment).

`services/pod-configmap`  
Indica que o port-forward será feito para um Service chamado pod-configmap.  
O kubectl irá escolher automaticamente um dos Pods associados a esse Service.  

`4443:443`  
4443 → porta local (na máquina)  
443 → porta remota (no Service/Pod dentro do cluster)

Acessar via navegador: `http://localhost:4443`

- O port-forward **funciona apenas enquanto o comando estiver em execução** (Ctrl+C encerra).
  - Não expõe o serviço para a rede externa, apenas para localhost.
  - Se o Service não expuser a porta 443, o comando falhará.

Para Pods diretamente, o formato seria:
  
~~~sh
$ kubectl port-forward pod/pod-configmap 4443:443
~~~

### Pontos importantes sobre o uso de ConfigMaps no Kubernetes:

- Atualizações: Os ConfigMaps não são atualizados automaticamente nos pods que os utilizam. Se você atualizar um ConfigMap, os pods existentes não receberão a nova configuração. Para que um pod receba a nova configuração, você precisa recriar o pod.

- Múltiplos ConfigMaps: É possível usar múltiplos ConfigMaps para um único pod. Isso é útil quando você tem diferentes aspectos da configuração que quer manter separados.

- Variáveis de ambiente: Além de montar o ConfigMap em um volume, também é possível usar o ConfigMap para definir variáveis de ambiente para os containers no pod.

- Imutabilidade: A partir da versão 1.19 do Kubernetes, é possível tornar ConfigMaps (e Secrets) imutáveis, o que pode melhorar o desempenho de sua cluster se você tiver muitos ConfigMaps ou Secrets.