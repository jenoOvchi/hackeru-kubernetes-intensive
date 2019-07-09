# Lab 8: Deployments

## Topic 1: Use Deployments

Создадим образ приложения 1 версии:
```bash
mkdir v1
cd v1
vi app.js
```

```javascript
const http = require('http');
const os = require('os');

console.log("Appjs server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("This is v1 running in pod " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

```bash
vi Dockerfile
```

```bash
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

```bash
docker build -t jenoovchi/appjs:v1 .
docker push jenoovchi/appjs:v1
cd ..
```

Создадим описание контроллера репликации и сервиса для приложения 1 версии:
```bash
vi appjs-rc-and-service-v1.yaml
```

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: appjs-v1
spec:
  replicas: 3
  template:
    metadata:
      name: appjs
      labels:
        app: appjs
    spec:
      containers:
      - image: jenoovchi/appjs:v1
        name: nodejs
---
apiVersion: v1
kind: Service
metadata:
  name: appjs
spec:
  type: LoadBalancer
  selector:
    app: appjs
  ports:
  - port: 80
    targetPort: 8080
```

Создадим контроллер репликации и сервис для приложения 1 версии:
```bash
k create -f appjs-rc-and-service-v1.yaml
```

Изучим созданный сервис:
```bash
k get svc appjs
```

Запустим скрипт вызова развёрнутого приложения:
```bash
while true; do curl http://$(k get svc appjs | grep appjs | awk '{print $4}'); done
```

Создадим образ приложения 2 версии:
```bash
mkdir v2
cd v2
vi app.js
```

```javascript
const http = require('http');
const os = require('os');

console.log("Appjs server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("This is v2 running in pod " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

```bash
vi Dockerfile
```

```bash
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

```bash
docker build -t jenoovchi/appjs:v2 .
docker push jenoovchi/appjs:v2
cd ..
```

Запустим процесс обновления приложения с 1 версии на 2:
```bash
k rolling-update appjs-v1 appjs-v2 --image=jenoovchi/appjs:v2
```

Изучим созданные контроллеры репликации и модули:
```bash
k describe rc appjs-v2
k describe rc appjs-v1
k get po --show-labels
```

Сэмулируем обновление приложения до 3 версии и повысим уровень логирования:
```bash
k rolling-update appjs-v2 appjs-v3 --image=jenoovchi/appjs:v1 --v 6
```

Удалим созданные контроллеры репликации:
```bash
k delete rc --all
```

Создадим описание развёртывания 1 версии приложения:
```bash
vi appjs-deployment-v1.yaml
```

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: appjs
spec:
  replicas: 3
  template:
    metadata:
      name: appjs
      labels:
        app: appjs
    spec:
      containers:
      - image: jenoovchi/appjs:v1
        name: nodejs
```

Создадим развёртывание 1 версии приложения с включенной опцией записи:
```bash
k create -f appjs-deployment-v1.yaml --record
```

Изучим статус развёртывания:
```bash
k rollout status deployment appjs
k get po
k get replicasets
k get deployments
k describe deployment appjs
```

Замедлим развёртывание, добавив простую проверку на готовность приложения:
```bash
k patch deployment appjs -p '{"spec": {"minReadySeconds": 10}}'
```

Изменим версию образа в развёртывании на вторую:
```bash
k set image deployment appjs nodejs=jenoovchi/appjs:v2
```

Изучим статус развёртывания:
```bash
k get rs
```

Создадим образ приложения 3 версии, возвращающего ошибку после 5 запроса:
```bash
mkdir v3
cd v3
vi app.js
```

```javascript
const http = require('http');
const os = require('os');

var requestCount = 0;

console.log("Appjs server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  if (++requestCount >= 5) {
    response.writeHead(500);
    response.end("Some internal error has occurred! This is pod " + os.hostname() + "\n");
    return;
  }
  response.writeHead(200);
  response.end("This is v3 running in pod " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

```bash
vi Dockerfile
```

```bash
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

```bash
docker build -t jenoovchi/appjs:v3 .
docker push jenoovchi/appjs:v3
cd ..
```

Изменим версию образа в развёртывании на третью:
```bash
k set image deployment appjs nodejs=jenoovchi/appjs:v3
```

Изучим статус развёртывания:
```bash
k rollout status deployment appjs
```

Откатим развёртывание к предыдущей версии:
```bash
k rollout undo deployment appjs
```

Изучим историю развёртываний:
```bash
k rollout history deployment appjs
```

Откатим развёртывание к первой версии:
```bash
k rollout undo deployment appjs --to-revision=1
```

Создадим описание стратегии развёртывания скользящего обновления:
```bash
vi rollingUpdate
```

```yaml
...
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
...
```

Создадим образ приложения 4 версии (без ошибки):
```bash
mkdir v4
cd v4
vi app.js
```

```javascript
const http = require('http');
const os = require('os');

console.log("Appjs server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("This is v4 running in pod " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

```bash
vi Dockerfile
```

```bash
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

```bash
docker build -t jenoovchi/appjs:v4 .
docker push jenoovchi/appjs:v4
cd ..
```

Изменим версию образа в развёртывании на четвёртую:
```bash
k set image deployment appjs nodejs=jenoovchi/appjs:v4
```

Остановим развёртывание:
```bash
k rollout pause deployment appjs
```

Изучим статус развёртывания:
```bash
k get rs
k rollout status deployment appjs
```

Продолжим развёртывание:
```bash
k rollout resume deployment appjs
```

Создадим описание развёртывания 3 версии приложения с проверкой готовности:
```bash
vi appjs-deployment-v3-with-readinesscheck.yaml
```

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: appjs
spec:
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      name: appjs
      labels:
        app: appjs
    spec:
      containers:
      - image: jenoovchi/appjs:v3
        name: nodejs
        readinessProbe:
          periodSeconds: 1
          httpGet:
            path: /
            port: 8080
```

Изменим текущее созданное развёртывания на новое:
```bash
k apply -f appjs-deployment-v3-with-readinesscheck.yaml
```

Изучим статус развёртывания:
```bash
k rollout status deployment appjs
k get po
k describe deploy appjs
```

Откатим развёртывание к предыдущей версии:
```bash
k rollout undo deployment appjs
```

Удалим созданные развёртывания и сервисы:
```bash
k delete deployments --all
k delete service --all
```