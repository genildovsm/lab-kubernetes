# Ingress

Etapas de deploy:

- Instalar o ingress-nginx (vide documentação do [kind](https://kind.sigs.k8s.io/docs/user/ingress/) ou [github](https://github.com/kubernetes/ingress-nginx/tree/main/deploy/static/provider))

~~~sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
~~~

Verificar se os pods ficam Ready antes de 1m30s.

~~~sh
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
~~~

- [redis](redis-deployment.yaml)
- [service do redis](redis-service.yaml)
- [app-senhas](app-deployment.yaml)
- [service do app-senhas](app-service.yaml)
  
Testar com port-forward se a app está funcionando
- `kubectl port-forward services/giropops-senhas 5000:5000`

