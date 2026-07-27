# Домашнее задание к занятию «Сетевое взаимодействие в Kubernetes»

**Выполнил:** Студент курса DevOps / Kubernetes  
**Дата:** 27 июля 2026 г.  
**Среда:** Minikube (v1.38.1), `kubectl` (v1.35.1/v1.36.3), Ingress Addon (`ingress-nginx:v1.14.3`)

---

## Файлы манифестов

### Задание 1
* [deployment-multi-container.yaml](./deployment-multi-container.yaml) — Манифест Deployment мульти-контейнерного приложения (`nginx` + `multitool`, 3 реплики).
* [service-clusterip.yaml](./service-clusterip.yaml) — Манифест Service `ClusterIP` (порт `9001` -> nginx, порт `9002` -> multitool).
* [service-nodeport.yaml](./service-nodeport.yaml) — Манифест Service `NodePort` (порт `80` -> nodePort `30080`).

### Задание 2
* [deployment-frontend.yaml](./deployment-frontend.yaml) — Манифест Deployment приложения `frontend` (`nginx`).
* [deployment-backend.yaml](./deployment-backend.yaml) — Манифест Deployment приложения `backend` (`multitool`).
* [service-frontend.yaml](./service-frontend.yaml) — Манифест Service `ClusterIP` для `frontend`.
* [service-backend.yaml](./service-backend.yaml) — Манифест Service `ClusterIP` для `backend`.
* [ingress.yaml](./ingress.yaml) — Манифест Ingress контроллера (маршруты `/` и `/api`).

---

## Задание 1. Настройка Service (ClusterIP и NodePort)

### 1.1. Создание Deployment с двумя контейнерами
Создан манифест [`deployment-multi-container.yaml`](./deployment-multi-container.yaml). В одном поде размещены два контейнера:
1. `nginx` — порт `80`.
2. `multitool` — порт `8080` (конфликт портов устранен передачей переменной `HTTP_PORT=8080`).

Количество реплик: `3`.

Содержимое манифеста [`deployment-multi-container.yaml`](./deployment-multi-container.yaml):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-container-app
  labels:
    app: multi-container
spec:
  replicas: 3
  selector:
    matchLabels:
      app: multi-container
  template:
    metadata:
      labels:
        app: multi-container
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        env:
        - name: HTTP_PORT
          value: "8080"
        - name: HTTPS_PORT
          value: "11443"
        ports:
        - containerPort: 8080
        - containerPort: 11443
```

### 1.2. Создание Service типа ClusterIP
Создан манифест [`service-clusterip.yaml`](./service-clusterip.yaml), открывающий:
- Порт `9001` для доступа к `nginx` (targetPort 80).
- Порт `9002` для доступа к `multitool` (targetPort 8080).

Содержимое манифеста [`service-clusterip.yaml`](./service-clusterip.yaml):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-container-clusterip
spec:
  type: ClusterIP
  selector:
    app: multi-container
  ports:
  - name: nginx-port
    port: 9001
    targetPort: 80
  - name: multitool-port
    port: 9002
    targetPort: 8080
```

Применяем манифест:
```bash
kubectl apply -f deployment-multi-container.yaml
kubectl apply -f service-clusterip.yaml
```

Проверяем список созданных подов и сервисов:
```bash
kubectl get pods,svc -o wide
```
Вывод команды:
```console
NAME                                       READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
pod/multi-container-app-86c6f75f7d-95wht   2/2     Running   0          28m   10.244.0.28   minikube   <none>           <none>
pod/multi-container-app-86c6f75f7d-df24t   2/2     Running   0          28m   10.244.0.29   minikube   <none>           <none>
pod/multi-container-app-86c6f75f7d-zqvcc   2/2     Running   0          28m   10.244.0.27   minikube   <none>           <none>

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE   SELECTOR
service/multi-container-clusterip   ClusterIP   10.108.58.196    <none>        9001/TCP,9002/TCP   28m   app=multi-container
```

#### Проверка доступности изнутри кластера (curl):
Проверяем доступность портов `9001` и `9002` через сервис `multi-container-clusterip`:

```bash
kubectl exec standalone-multitool -- curl -s -I http://multi-container-clusterip:9001
```
Вывод (`nginx` на порту 9001):
```console
HTTP/1.1 200 OK
Server: nginx/1.25.5
Date: Mon, 27 Jul 2026 17:52:37 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
```

```bash
kubectl exec standalone-multitool -- curl -s http://multi-container-clusterip:9002
```
Вывод (`multitool` на порту 9002):
```console
WBITT Network MultiTool (with NGINX) - multi-container-app-86c6f75f7d-zqvcc - 10.244.0.27 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
```

### 1.3. Создание Service типа NodePort
Создан манифест [`service-nodeport.yaml`](./service-nodeport.yaml), открывающий внешний доступ к `nginx` на порту узла `30080`.

Содержимое манифеста [`service-nodeport.yaml`](./service-nodeport.yaml):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-container-nodeport
spec:
  type: NodePort
  selector:
    app: multi-container
  ports:
  - name: nginx-nodeport
    port: 80
    targetPort: 80
    nodePort: 30080
```

Применяем манифест:
```bash
kubectl apply -f service-nodeport.yaml
```

#### Проверка доступности снаружи:
Выполняем запрос к IP-адресу узла Minikube и внешнему порту `30080`:

```bash
curl -s -I http://<node-ip>:30080
```
Вывод:
```console
HTTP/1.1 200 OK
Server: nginx/1.25.5
Date: Mon, 27 Jul 2026 18:03:24 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
```

---

## Задание 2. Настройка Ingress

### 2.1. Включение Ingress-контроллера
Включаем аддон Ingress в Minikube:
```bash
minikube addons enable ingress
```

Проверяем работу Ingress-контроллера:
```bash
kubectl get pods -n ingress-nginx
```
Вывод:
```console
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-769d4d5464-9b8tw   1/1     Running   0          10m
```

### 2.2. Развертывание Deployment и Service для Frontend и Backend

1. **Frontend (`nginx`):**
   - Манифест Deployment: [`deployment-frontend.yaml`](./deployment-frontend.yaml)
   - Манифест Service: [`service-frontend.yaml`](./service-frontend.yaml)

2. **Backend (`multitool`):**
   - Манифест Deployment: [`deployment-backend.yaml`](./deployment-backend.yaml) (переменная `HTTP_PORT=8080`)
   - Манифест Service: [`service-backend.yaml`](./service-backend.yaml)

Применяем манифесты:
```bash
kubectl apply -f deployment-frontend.yaml
kubectl apply -f deployment-backend.yaml
kubectl apply -f service-frontend.yaml
kubectl apply -f service-backend.yaml
```

### 2.3. Создание манифеста Ingress
Создан манифест [`ingress.yaml`](./ingress.yaml) с rewrite-правилом и маршрутизацией:
- Путь `/` перенаправляет на `frontend-svc:80`.
- Путь `/api` перенаправляет на `backend-svc:80`.

Содержимое манифеста [`ingress.yaml`](./ingress.yaml):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 80
```

Применяем манифест:
```bash
kubectl apply -f ingress.yaml
```

Проверяем статус Ingress ресурса:
```bash
kubectl get ingress app-ingress
```
Вывод:
```console
NAME          CLASS   HOSTS   ADDRESS        PORTS   AGE
app-ingress   nginx   *       192.168.49.2   80      1m
```

### 2.4. Проверка доступности через Ingress

1. **Запрос к пути `/` (Frontend / Nginx):**
```bash
curl -s -I http://192.168.49.2/
```
Вывод ответа Frontend:
```console
HTTP/1.1 200 OK
Date: Mon, 27 Jul 2026 18:29:37 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
Last-Modified: Tue, 16 Apr 2024 14:29:59 GMT
ETag: "661e8b67-267"
Accept-Ranges: bytes
```

2. **Запрос к пути `/api` (Backend / Multitool):**
```bash
curl -s http://192.168.49.2/api
```
Вывод ответа Backend:
```console
WBITT Network MultiTool (with NGINX) - backend-app-6cc6fcb7fc-47d9l - 10.244.0.31 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
```

---

## Заключение
1. Настроен `Deployment` с 3 репликами мульти-контейнерного приложения.
2. Создан `Service` типа `ClusterIP`, открывающий порты `9001` (nginx) и `9002` (multitool) внутри кластера.
3. Настроен `Service` типа `NodePort`, открывающий внешний порт `30080`.
4. Включен Ingress-контроллер в Minikube, развернуты приложения `frontend` и `backend` с соответствующими сервисами `ClusterIP`.
5. Создан ресурс `Ingress`, успешно настроена маршрутизация запросов к `/` (frontend) и `/api` (backend).
