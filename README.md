
# Kubernetes

# Instalação e comandos:

## [Instalando k3d](https://k3d.io/v5.5.1/#install-current-latest-release)

```bash
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

## [Kubernets-tools](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux)

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version --client
```

## Extensões vscode

```bash
Intall docker
install kubernet
```

## Criando um Cluster-k3d

```bash
k3d cluster create

k3d cluster create "nome-do-cluster" --servers "quantidade-de-servers" --agents "quantidade-de-agents"

k3d cluster delete nome #apaga um cluster e todas as suas instâncias 

k3d cluster list
```

## Criando o yaml

```bash
apiVersion: v1
kind: Pod
metadata:
  name: meupod
spec:
  containers:
  - name: meucontainer
    image: fabricioveronez/web-page:blue
    ports:
    - containerPort: 80
```

```bash
#aplicando o yaml no cluster kubernet
kubectl apply -f meupod.yaml #atualiza as modificações ~do yaml
kubectl get pod
kubectl describe pod meupod
```

## Portas

```bash
#acessando a aplicação do container 
kubectl port-forward pod/meupod "porta-local:porta-container" 
```

## Deplyment

```bash
kubectl apply -f meudeployment.yaml
kubectl get deployment
```

```bash
kubectl get all #lista todos as instancias do kubernets em running.
```

```bash
#pods, replicas, services e loadbalancer
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meudeploymentset
spec:
  replicas: 2
  selector:
    matchLabels:
      app: meupod
  template:
    metadata:
      labels:
        app: meupod
    spec:
      containers:
      - name: site
        image: fabricioveronez/web-page:green
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: meusdeploymentset-service
spec:
  selector:
    app: meupod
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30000
  type: NodePort
```

## Bind de portas e Loadbalancer

```bash
k3d cluster create meucluster --servers 2 --agents 2 -p "30000:30000@loadbalancer"
```

# Projeto chatGPT-kubernetes:

## Criando o cluster:

```bash
k3d cluster create meucluster -p "3000:30000@loadbalancer" -p "8080:30001@loadbalancer"
```

## Criando a imagem docker

```bash
git clone git@github.com:KubeDev/idc-ms-chatgpt.git

cd chatservice

docker build -t klysman08/imersao-chatservice:v1 .

docker image ls #para verificar se a imagem foi criada com sucesso

docker push #para subir a image para o dockerhub

docker tag klysman08/imersao-chatservice:v1 klysman08/imersao-chatservice:latest #criando a versionamento da imagem
```

## Deploy chatservice.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: chatservice-mysql
spec:
  selector:
    matchLabels:
      app: chatservice-mysql
  template:
    metadata:
      labels:
        app: chatservice-mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8
          ports: 
          - containerPort: 3306
          env:
          - name: MYSQL_ROOT_PASSWORD
            value: "root"
          - name: MYSQL_DATABASE
            value: "chat_service"
---
apiVersion: v1
kind: Service
metadata:
  name: chatservice-mysql
spec:
  selector: 
    app: chatservice-mysql
  ports:
  - port: 3306
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: chatservice
spec: 
  selector:
    matchLabels:
      app: chatservice
  template:
    metadata:
      labels:
        app: chatservice
    spec: 
      containers:
        - name: chatservice
          image: klysman08/imersao-chatservice:v1
          ports:
          - containerPort: 8080
            name: http
            protocol: TCP
          - containerPort: 50051
            name: grpc
            protocol: TCP 
          env: 
          - name: DB_DRIVER
            value: mysql
          - name: DB_HOST
            value: chatservice-mysql
          - name: DB_PORT
            value: "3306"
          - name: DB_USER
            value: root
          - name: DB_NAME
            value: chat_service
          - name: DB_PASSWORD
            value: root
          - name: WEB_SERVER_PORT
            value: "8080"
          - name: GRPC_SERVER_PORT
            value: "50051"     
          - name: INITIAL_CHAT_MESSAGE
            value: "Seu nome é Vivy. Você é a inteligência artificial da iniciativa DevOps && Cloud. Você da suporte a programadores e profissionais de infraestrutura."
          - name: OPENAI_API_KEY
            value: "api=key"
          - name: AUTH_TOKEN
            value: "123456"
---
apiVersion: v1
kind: Service
metadata:
  name: chatservice
spec: 
  selector: 
    app: chatservice
  type: ClusterIP
  ports:
    - port: 8080
      protocol: TCP
      name: http
    - port: 50051
      protocol: TCP
      name: grpc
```

```bash
kubectl apply -f deploy-chatservice.yaml
```

## Deploy keycloak.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:21.0
        args: ["start-dev"]
        env:
        - name: KEYCLOAK_ADMIN
          value: "admin"
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: "admin"
        - name: KC_PROXY
          value: "edge"
        ports:
        - name: http
          containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  labels:
    app: keycloak
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    nodePort: 30001
  selector:
    app: keycloak
  type: NodePort
```

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-mysql
spec:
  selector:
    matchLabels:
      app: webapp-mysql
  template:
    metadata:
      labels:
        app: webapp-mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "root"
        - name: MYSQL_DATABASE
          value: "webchat"
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-mysql
spec:
  selector:
    app: webapp-mysql
  ports:
  - port: 3306
    targetPort: 3306
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: fabricioveronez/imersao-gpt-webapp:v1
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        env:
        - name: DATABASE_URL
          value: mysql://root:root@webapp-mysql:3306/webchat
        - name: NEXTAUTH_URL
          value: http://a8319ee95d7fe42d783ab924aa7c8b36-765495673.us-east-1.elb.amazonaws.com/api/auth
        - name: NEXTAUTH_SECRET
          value: "123456"
        - name: KEYCLOAK_CLIENT_ID
          value: gpt-webapp
        - name: KEYCLOAK_CLIENT_SECRET
          value: y7K9rmm2FUBLZOBRZHko3HhL0TTQ0eBP
        - name: KEYCLOAK_ISSUER
          value: http://aed6935ab669d403ea1eaec2f2507bb8-869034135.us-east-1.elb.amazonaws.com/realms/master
        - name: CHATSERVICE_URL
          value: "chatservice:50051"
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
      targetPort: 3000
      #nodePort: 30000
      name: http
```
