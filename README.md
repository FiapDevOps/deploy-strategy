# deploy-strategy
Deployment strategy examples using Kubernetes

---

## Blue Green Deployment

Usando o Kubernetes, podemoscriar um blue-green deployment com base no recurso service que atuará como nosso roteador e dois deployments distintos aplicados na implantação. Neste formato podemos alternar o Selector do service para rotear as requisições na aplicação entre as entregas nas versões blue e green;

Importante: **Os modelos deste exemplo já foram disponibilizados no diretório kubernetes;**

1. Primeiro, crie uma configuração de service para atuar como roteador para nossas requisições e uma configuração de ingress que atuará como frontend permitindo o acesse externo (qualquer requisição HTTP com base no endereço de dns configurado) ao cluster;

```sh
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  type: ClusterIP
  selector:
    app: node-react
    deployment: ${DEPLOYMENT}
  ports:
  - containerPort: 5000
    port: 80
```

```sh
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: app-loadbalancer
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: ${API_DNS}
    http:
      paths:
      - path: /
        backend:
          serviceName: app
          servicePort: 80
```


A variável ${DEPLYOMENT} será responsável por configurar o seletor deployment, tornando a entrega das versões Blue e Green um tipo de template que possa ser alterado rapidamente via linha de comando.

2. Aplqie as configurações iniciais do serviço que atuará como roteador das requisições e do load balancer que fará a exposição da aplicação para acesso externo:

```sh
DEPLOYMENT=blue \
envsubst < kubernetes/service.yml | kubectl apply -f -

API_DNS="apiSEURM.fiaplabs.com" \
envsubst < kubernetes/ingress.yml | kubectl apply -f -
```

> Você também poderia usar o patch do kubernetes para fazer as altrações no serviço, mas é mais fácil modelar o arquivo e usar o kubectl apply para fins de exemplo neste laboratório);

3. Agora crie o arquivo com o deploy da aplicação:

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-${DEPLOYMENT}
spec:
  replicas: 2
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: node-react
        deployment: ${DEPLOYMENT}
    spec:
      containers:
      - name: node-react
        image: devfiap/node-react-app:${IMAGE_TAG}
        ports:
        - containerPort: 5000
        readinessProbe:
           tcpSocket:
              port: 5000
        initialDelaySeconds: 30
        periodSeconds: 5
```

A variavel ${IMAGE_TAG} será usado para que possamos chavear rapidamente a tag da imagem do docker para o próximo deployment alternando entre blue e green;

4. Crie o deployment blue conforme o arquivo:

```sh
DEPLOYMENT=blue IMAGE_TAG=v1.0 \
envsubst < kubernetes/deployment.yml | kubectl apply -f -
```

5. Aguarde até que esteja totalmente entregue:

```sh
kubectl rollout status deployment app-blue
```

6. Crie o deployment green conforme o arquivo:

```sh
DEPLOYMENT=green IMAGE_TAG=v2.0 \
envsubst < kubernetes/deployment.yml | kubectl apply -f -
```

7. Agora que ambas as versões estão disponíveis basta a alteração no seletor do serviço para a versão green entra em uso chaveando o balanceador de carga da aplicação:

```sh
DEPLOYMENT=green \
envsubst < kubernetes/service.yml | kubectl apply -f -
```

---
