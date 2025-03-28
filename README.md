## DeployeDataBaseAppOnEKSOnAWS

## AWS EKS Setup and Deployment Guide

## Step 1: Create AMI User with Admin Access
1. Create an AMI user with Administrator access.
2. Inside the admin account, update security credentials.
3. Create an access key.
4. Retrieve the access key and secret key for CLI authentication.

## Step 2: Install AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install --bin-dir /usr/bin --install-dir /usr/bin/aws-cli --update
aws --version
aws configure
```
- Set the region: `us-east-1`
- Set the output format: `json`

## Step 3: Install kubectl
```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.16.8/2020-04-16/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
kubectl version --short --client
```

## Step 4: Install eksctl
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/bin
eksctl version
```

## Step 5: Provision EKS Cluster
```bash
eksctl create cluster --name dev --region us-east-1 --nodegroup-name standard-workers --node-type t3.medium --nodes 1 --nodes-min 1 --nodes-max 4 --managed
eksctl get cluster
aws eks update-kubeconfig --name dev --region us-east-1
kubectl config get-contexts
```

## Step 6: Deploy MySQL on Kubernetes
### Create Secret
```bash
nano mysql-secret.yaml
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: kubernetes.io/basic-auth
stringData:
  password: root
```
```bash
kubectl apply -f mysql-secret.yaml
```

### Create Persistent Volume
```bash
nano mysql-storage.yaml
```
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```
```bash
kubectl apply -f mysql-storage.yaml
```

### Deploy MySQL
```bash
nano mysql-deployment.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:latest
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
```
```bash
kubectl apply -f mysql-deployment.yaml
```
# Kubernetes Commands and Configuration Notes

## Retrieve Pods
```sh
kubectl get pod
```

## Access MySQL Pod
```sh
kubectl exec --stdin --tty <pod-name> -- /bin/bash
```

## Retrieve Services
```sh
kubectl get svc
```

## Update `application.properties`
After retrieving the services, we need to copy the MySQL cluster IP and update the `application.properties` file:

```sh
jdbc:mysql://<cluster-ip>:3306/shipwreck
username=root
password=root
```

## Step 7: Install Docker
```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
apt-cache policy docker-ce
sudo apt install docker-ce
sudo systemctl status docker
sudo usermod -aG docker ${USER}
exit
```

## Step 8: Create and Push Docker Image
### Dockerfile
```dockerfile
# Build stage
FROM maven:3.6.3-jdk-8-slim AS build
COPY src /home/app/src
COPY pom.xml /home/app
RUN mvn -f /home/app/pom.xml clean install -DskipTests

# Package stage
FROM openjdk:8-jdk-alpine
COPY --from=build /home/app/target/*.jar app.jar
EXPOSE 8181
ENV SPRING_PROFILES_ACTIVE=prod
ENTRYPOINT ["java","-jar","app.jar"]
```
```bash
docker build -t shipwreckapp .
docker tag shipwreckapp:latest bhavindalsaiya/mysqlshipwreck:latest
docker login
docker push bhavindalsaiya/mysqlshipwreck:latest
```

## Step 9: Deploy Shipwreck Application
### Deployment
```bash
nano shipwreck-deployment.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipwrackbhavin-deployment
  labels:
    app: shipwrackbhavin
    env: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shipwrackbhavin
      env: dev
  template:
    metadata:
      labels:
        app: shipwrackbhavin
        env: dev
    spec:
      containers:
      - name: shipwrackbhavin-container
        image: bhavindalsaiya/mysqlshipwreck:latest
        ports:
        - containerPort: 8181
```
```bash
kubectl apply -f shipwreck-deployment.yaml
```

### Service
```bash
nano shipwreck-service.yaml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: shipwrackbhavin-service
  labels:
    app: shipwrackbhavin
    env: dev
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8181
  selector:
      app: shipwrackbhavin
      env: dev
```
```bash
kubectl apply -f shipwreck-service.yaml
```
```bash
kubectl get deploy
```
```bash
kubectl expose deployment shipwrackbhavin-deployment --name=shipwreck-svc --type=LoadBalancer --port=80 --target-port=8181
```
```bash
kubectl get svc
```

# Kubernetes Services

## Service Details
```
NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP
           PORT(S)        AGE
kubernetes                ClusterIP      10.100.0.1      <none>
           443/TCP        45m
mysql                     ClusterIP      10.100.43.24    <none>
           3306/TCP       36m
shipwrackbhavin-service   LoadBalancer   10.100.248.4    a5420ac73d75f4baaa70b41393cc143e-401312477.us-east-1.elb.amazonaws.com    80:30984/TCP   24m

This above load balancer will work.

shipwreck-svc             LoadBalancer   10.100.157.42   ab0b34330bac647b58ab84aef3d21604-1340199434.us-east-1.elb.amazonaws.com   80:31912/TCP   24m
