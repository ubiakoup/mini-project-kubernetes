# PayMyBuddy

![homepage-paymybuddy](src/main/resources/readme/home.png)

# PayMyBuddy – Kubernetes Deployment with Manifests

## Context

This project demonstrates how to deploy a Spring Boot application (PayMyBuddy) with a MySQL database on Kubernetes using only YAML manifests (without Helm).

The objective is to understand how Kubernetes components work together:

* Deployments
* Services
* Persistent Volumes (PV/PVC)
* Secrets
* Application debugging

---

## Prerequisites

Install Java and Maven on your Ubuntu VM:

```bash
sudo apt install openjdk-17-jre-headless maven -y
```

---

## Build the Application

```bash
mvn clean install
```

This command generates the executable:

```
target/paymybuddy.jar
```

---

## Project Structure

The following structure is obtained after building the application:

```
PayMyBuddy/
├── Dockerfile
├── pom.xml
├── target/
│   └── paymybuddy.jar
├── src/main/resources/database/
│   ├── create.sql
│   └── data.sql
├── mysql-deployment.yaml
├── mysql-service.yaml
├── paymybuddy-deployment.yaml
├── paymybuddy-service.yaml
├── db-pv.yaml
├── db-pvc.yaml
├── pmb-pv.yaml
├── pmb-pvc.yaml
└── README.md
```

---

## Docker Image Build

```bash
docker build -t ipaymybuddy-backend .
```

If using Minikube with Docker driver:

```bash
eval $(minikube docker-env)
```

If using `--driver=none`, build directly.

---

## Secrets Configuration

```bash
kubectl create secret generic mysql-secrets \
  --from-literal=mysql_root_password=root \
  --from-literal=mysql_password=paymybuddy123

kubectl create secret generic pmb-secrets \
  --from-literal=pmb-password=paymybuddy123
```

Important: the password values must be identical across MySQL and backend configuration.

---

## Architecture Overview

```
User
  ↓
NodePort (30007)
  ↓
PayMyBuddy Pod (Spring Boot)
  ↓
ClusterIP Service (mysql)
  ↓
MySQL Pod
  ↓
Persistent Volume
```

---

## MySQL Deployment

* One replica
* Credentials managed via Kubernetes Secrets
* Persistent storage enabled
* Database initialized automatically using SQL scripts

Environment variables:

```yaml
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secrets
      key: mysql_root_password

- name: MYSQL_DATABASE
  value: db_paymybuddy

- name: MYSQL_USER
  value: paymybuddyuser

- name: MYSQL_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secrets
      key: mysql_password
```

---

## Database Initialization

SQL scripts are located in:

```
src/main/resources/database/
```

Mounted into the container:

```
/docker-entrypoint-initdb.d
```

This allows automatic execution of:

* create.sql (schema)
* data.sql (initial data)

---

## MySQL Service

```yaml
type: ClusterIP
```

Accessible internally via:

```
mysql:3306
```

---

## PayMyBuddy Deployment

* Uses locally built Docker image
* Connects to MySQL via environment variables
* Uses Kubernetes Secrets for password

```yaml
env:
- name: SPRING_DATASOURCE_URL
  value: jdbc:mysql://mysql:3306/db_paymybuddy

- name: SPRING_DATASOURCE_USERNAME
  value: paymybuddyuser

- name: SPRING_DATASOURCE_PASSWORD
  valueFrom:
    secretKeyRef:
      name: pmb-secrets
      key: pmb-password
```

---

## Backend Service

```yaml
type: NodePort
nodePort: 30007
```

Access the application at:

```
http://<NODE-IP>:30007
```

---

## Persistent Storage

Two persistent storage configurations are used:

### MySQL

* PV: db-pv.yaml
* PVC: db-pvc.yaml
* Mount path: /var/lib/mysql
* Host path: /data-pv

### Backend

* PV: pmb-pv.yaml
* PVC: pmb-pvc.yaml
* Mount path: /data
* Host path: /data

---

## Deployment

```bash
kubectl apply -f .
```

---

## Debugging

```bash
kubectl get pods
kubectl logs -l app=paymybuddy
kubectl logs -l app=mysql
kubectl describe pod <pod-name>
```

---

## Common Issue

```
Access denied for user 'paymybuddyuser'
```

Cause:

* Password mismatch between MySQL and backend
* MySQL initialized with previous values

Solution:

```bash
kubectl delete pvc db-pvc
kubectl delete pod -l app=mysql
kubectl apply -f .
```

## running deployment

<img width="630" height="403" alt="image" src="https://github.com/user-attachments/assets/c714d360-9789-44ea-b8ec-049ce025aa81" />


