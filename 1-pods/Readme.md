# Lab 2: Pods
---

## Topic 1: Use Pods

Запускаем приложение из собранного образа Docker:
```bash
k run appjs --image=jenoovchi/appjs --port=8080 --generator=run/v1
```

Изучаем созданный ресурс модуль:
```bash
k get pods
k describe pod
```

Предоставляем доступ к приложению в рамках кластера Kubernetes:
```bash
k expose rc appjs --type=LoadBalancer --name appjs-http
```

Изучаем созданный ресурс сервис:
```bash
k get services
k get svc
```

Тестируем работу развёрнутого приложения:
```bash
curl http://35.228.90.221:8080
```

Изучаем созданный ресурс контроллер репликации:
```bash
k get replicationcontrollers
k api-resources
```

Масштабирование количества запущенных реплик:
```bash
k scale rc appjs --replicas=3
```

Проверка успешности масштабирования:
```bash
k get rc
k get pods
```

Тестируем работу смасштабированного приложения:
```bash
curl http://35.228.90.221:8080
```

Изучаем созданные модули:
```bash
k get pods -o wide
k describe pod $(k get po | grep appjs | head -n 1 | awk '{print $1}')
k get po $(k get po | grep appjs | head -n 1 | awk '{print $1}') -o yaml
k explain pods
k explain pods.spec
```

Создаём описание приложения:
```bash
vi appjs-manual.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: appjs-manual
spec:
  containers:
  - image: jenoovchi/appjs
    name: appjs
    ports:
    - containerPort: 8080
```

Создаём модуль приложения:
```bash
k create -f appjs-manual.yaml
```

Изучаем созданный модуль:
```bash
k get po appjs-manual -o yaml
k get po appjs-manual -o json
k get pods
```

Получаем доступ к логам приложения напрямую из Docker:
```bash
k describe pod $(k get po | grep appjs-manual | head -n 1 | awk '{print $1}')
gcloud compute ssh $(k describe pod $(k get po | grep appjs | head -n 1 | awk '{print $1}') | grep Node | head -n 1 | awk '{print $2}' | sed 's/\/.*//')
docker ps
docker ps | grep appjs
docker logs $(docker ps | grep appjs | awk '{print $2}')
exit
```

Получаем доступ к логам приложения с помощью Kubectl:
```bash
k logs appjs-manual
k logs appjs-manual -c appjs
```

Тестируем приложение с помощью перенаправления порта контейнера на порт локальной машины:
```bash
k port-forward appjs-manual 8888:8080
curl http://localhost:8888
```

## Topic 2: Use Labels

Создаём описание приложения c использованием меток:
```bash
vi appjs-manual-with-labels.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: appjs-manual-v2
  labels:
    creation_method: manual
    env: prod
spec:
  containers:
  - image: jenoovchi/appjs
    name: appjs
    ports:
    - containerPort: 8080
```

Создаём модуль приложения с метками:
```bash
k create -f appjs-manual-with-labels.yaml
```

Изучаем созданный модуль:
```bash
k get po --show-labels
k get po -L creation_method,env
```

Добавляем модулю дополнительные метки:
```bash
k label po appjs-manual creation_method=manual
k label po appjs-manual-v2 env=debud --overwrite
```

Изучаем изменённый модуль:
```bash
k get po -L creation_method,env
```

## Topic 3: Use Selectors

Сортируем модули с помощью меток:
```bash
k get po -l creation_method=manual
k get po -l env
k get po -l '!env'
k get po -l creation_method!=manual
k get po -l 'env in (prod,devel)'
k get po -l 'env notin (prod,devel)'
```

Добавляем узлу дополнительную метки:
```bash
k get nodes
k label node $(k get nodes | grep gke | head -n 1 | awk '{print $1}') gpu=true
```

Находим нужный узел с помощью созданной метки:
```bash
k get nodes -l gpu=true
```

Создаём описание приложения c использованием селектора узлов:
```bash
vi appjs-gpu.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: appjs-gpu
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - image: jenoovchi/appjs
    name: appjs
```

Создаём модуль приложения с селектором узлов:
```bash
k create -f appjs-gpu.yaml
```

Изучаем созданный модуль:
```bash
k describe pod appjs-gpu
```

## Topic 4: Use Annotations

Изучаем один из созданных модулей на наличие аннотаций:
```bash
k get po $(k get po | grep appjs | head -n 1 | awk '{print $1}') -o yaml
```

Добавляем модулю appjs-manual дополнительную аннотацию:
```bash
k annotate pod appjs-manual mycompany.com/someannotation="foo bar"
```

Изучаем аннотированный модуль:
```bash
k describe pod appjs-manual
```

## Topic 5: Use Namespaces

Выведем текущий список пространств имён:
```bash
k get ns
```

Выведем список модулей пространства имён kube-system:
```bash
k get po --namespace kube-system
```

Создаём описание пространства имён:
```bash
vi custom-namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```

Создаём новое пространство имён:
```bash
k create -f custom-namespace.yaml
```

Выведем обновлённый список пространств имён:
```bash
k get ns
```

Создаём новое пространство имён с помощью консольной команды:
```bash
k create namespace custom-namespace-v2
```

Выведем обновлённый список пространств имён:
```bash
k get ns
```

Создаём модуль приложения в новом пространстве имён:
```bash
k create -f appjs-manual.yaml -n custom-namespace
```

Выведем текущий список модулей в созданном пространстве имён:
```bash
k get po -n custom-namespace
```

Создаём псевдоним для быстрой смены пространства имён:
```bash
alias kcd='kubectl config set-context $(kubectl config current-context) --namespace '
```

Удаляем созданные ресурсы:
```bash
k delete po appjs-gpu
k delete po -l creation_method=manual
k delete po -l rel=canary
k delete ns custom-namespace
k get pods
k delete po --all
k get pods
k delete all --all
```

## Topic 6: Use LivenessProbe

Изменяем код приложения так, чтобы после 5 запроса оно возвращало ошибку сервера:
```bash
vi app.js
```

```javascript
const http = require('http');
const os = require('os');

console.log("Appjs server starting...");

var requestCount = 0;

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  requestCount++;
  if (requestCount > 5) {
    response.writeHead(500);
    response.end("I'm not well. Please restart me!");
    return;
  }
  response.writeHead(200);
  response.end("You've hit " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

Создаём (если его нет) описание образа приложения:
```bash
vi Dockerfile
```

```bash
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

Собираем и публикуем версию приложения с ошибкой:
```bash
docker build -t jenoovchi/appjs:unhealthy .
docker push jenoovchi/appjs:unhealthy
```

Создаём описание модуля приложения с ошибкой и проверкой живости:
```bash
vi appjs-liveness-probe.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: appjs-liveness
spec:
  containers:
  - image: jenoovchi/appjs:unhealthy
    name: appjs
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```

Создаём модуль приложения с ошибкой и проверкой живости:
```bash
k create -f appjs-liveness-probe.yaml
```

Изучение модуля приложения с ошибкой и проверкой живости:
```bash
k get po appjs-liveness
k logs appjs-liveness
k logs appjs-liveness --previous
k describe po appjs-liveness
```

Создаём описание модуля приложения с ошибкой, проверкой живости и начальной задержкой:
```bash
vi appjs-liveness-probe-initial-delay.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: appjs-liveness-initial-delay
spec:
  containers:
  - image: jenoovchi/appjs:unhealthy
    name: appjs
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 15
```

Создаём модуль приложения с ошибкой, проверкой живости и начальной задержкой:
```bash
k create -f appjs-liveness-probe-initial-delay.yaml
```

Изучение созданного модуля приложения:
```bash
k get po appjs-liveness-initial-delay
```

Удаление созданных модулей:
```bash
k delete pods --all
```