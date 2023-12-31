# Twoge Web App Deployment via Minikube and EKS Cluster

![dia](https://github.com/cth9191/a4/blob/main/3.JPG)

During this project's first phase, we will deploy the web application Twoge on our local machine via Minikube. Once that deployment is successful, we will then shift to using AWS Elastic Kubernetes Service to host our app.


### Git Repo
```
https://github.com/chandradeoarya/twoge/tree/k8s
```
#### Git clone Repo
```
 git clone -b k8s git@github.com:chandradeoarya/twoge.git
```

### Create Dockerfile
```
FROM python:alpine
RUN apk update && \
    apk add --no-cache build-base libffi-dev openssl-dev
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 8080
CMD python app.py
```

#### Build Dockerfile
```
docker build -t cth91/twoge-new .
docker push cth91/twoge-new 
```

### Create Namespase.yml file
```
apiVersion: v1
kind: Namespace
metadata:
  name: twoge-ns
  labels:
    name: twoge-ns
```

### Create Resourcequota.yml file
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: twoge-quota
  namespace: twoge-ns
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

#### Create Configmap.yml file
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-configmap
data:
  database_url: "postgres-service"
  database_port: "5432"
```

### Create secrets.yml file
```
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  postgres-database: <your database name>
  postgres-username: <your username>
  postgres-password: <your password>
```

### Create twoge web deployment yml file with readiness probe
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: twoge-dep
  labels:
    app: twoge-k8s
spec:
  selector:
    matchLabels:
      app: twoge-k8s
  replicas: 1
  template:
    metadata:
      labels:
        app: twoge-k8s
    spec:
      containers:
        - name: twoge-container
          image: cth91/twoge-new
          ports:
            - containerPort: 8080
          readinessProbe:
           tcpSocket:
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 20
          env:
            - name: DB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: postgres-database
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: postgres-username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: postgres-password
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: postgres-configmap
                  key: database_host
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: postgres-configmap
                  key: database_port
```

### Create twoge web service yml file
```
apiVersion: v1
kind: Service
metadata:
  name: twoge-k8s-service
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30036
  selector:
    app: twoge-k8s
```

### Create twoge Postgres deployment yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres
        env:
          - name: POSTGRES_DB
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-database
          - name: POSTGRES_USER
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-username
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-password
```

### Create twoge Postgres service yml file
```
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  type: ClusterIP
  ports:
  - port: 5432
```




### Kubetl Commands
```
kubectl apply -f Namespace.yaml                                 # create namespace
kubectl apply -f ResourceQuota.yml                              # create resourcequota
kubectl apply -f . --namespace twoge-ns                         # attach all resource to namespace
kubectl get resourcequotas --namespace twoge-ns                 # apply resourcequota twoge-ns resourcequota
kubectl config set-context --current --namespace=twoge-ns       # set namespace preference
kubectl get all                                                 # confirm all pods are up and running

```

###  Confirm that Twoge is up and running on your local host
```
minikube service twoge-k8s-service -n twoge-ns --url
```



![twoge2](https://github.com/asanni2022/twoge_eks/assets/104282577/eb5f5fe7-afca-4ae7-8cec-321125499434)



# Deployment Via EKS Cluster

Now that we have successfully deployed our app via Minikube, we will now make some small configuration changes and set up our EKS cluster.

### Update Service.yml file by editing the Port type
```
apiVersion: v1
kind: Service
metadata:
  name: twoge-k8s-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: twoge-k8s
```

### Creater EKS Cluster 
```
eksctl create cluster --region us-west-2 --node-type t2.small --nodes 1 --nodes-min 1 --nodes-max 1 --name penny-twoge
```

Once the cluster has been created (this may take several minutes), you can log into AWS and verify that your application is being hosted on an EC2 instance. 

![eks](https://github.com/cth9191/a4/blob/main/1.JPG)





## Challenges
```
Initial Pod Creation
Initial EKS Creation
```


