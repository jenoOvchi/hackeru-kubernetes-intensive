# Lab 9: Pods

## Topic 1: Use StatefulSets

Создадим образ приложения, хранящего состояние:
```bash
mkdir appjs-pet-image
cd appjs-pet-image
vi app.js
```

```javascript
const http = require('http');
const os = require('os');
const fs = require('fs');

const dataFile = "/var/data/appjs.txt";

function fileExists(file) {
  try {
    fs.statSync(file);
    return true;
  } catch (e) {
    return false;
  }
}

var handler = function(request, response) {
  if (request.method == 'POST') {
    var file = fs.createWriteStream(dataFile);
    file.on('open', function (fd) {
      request.pipe(file);
      console.log("New data has been received and stored.");
      response.writeHead(200);
      response.end("Data stored on pod " + os.hostname() + "\n");
    });
  } else {
    var data = fileExists(dataFile) ? fs.readFileSync(dataFile, 'utf8') : "No data posted yet";
    response.writeHead(200);
    response.write("You've hit " + os.hostname() + "\n");
    response.end("Data stored on this pod: " + data + "\n");
  }
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
docker build -t jenoovchi/appjs-pet .
docker push jenoovchi/appjs-pet
cd ..
```

Создадим постоянные тома для экземпляров нашего приложения:
```bash
gcloud compute disks create --size=1GiB --zone=$(gcloud container clusters list | grep jeno | awk '{print $2}') pv-a
gcloud compute disks create --size=1GiB --zone=$(gcloud container clusters list | grep jeno | awk '{print $2}') pv-b
gcloud compute disks create --size=1GiB --zone=$(gcloud container clusters list | grep jeno | awk '{print $2}') pv-c
```

Проверим созданные диски:
```bash
gcloud compute disks list
```

Создадим описание постоянных томов в виде списка:
```bash
vi persistent-volumes-gcepd.yaml
```

```yaml
kind: List
apiVersion: v1
items:
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-a
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    gcePersistentDisk:
      pdName: pv-a
      fsType: ext4
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-b
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    gcePersistentDisk:
      pdName: pv-b
      fsType: ext4
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-c
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    gcePersistentDisk:
      pdName: pv-c
      fsType: ext4
```

Создадим постоянные тома:
```bash
k create -f persistent-volumes-gcepd.yaml
```

Изучим созданные постоянные тома:
```bash
k get pv
k describe pv pv-a
```

Создадим описание Headless сервиса для идентификации экземпляров приложения между собой:
```bash
vi appjs-service-headless.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs
spec:
  clusterIP: None
  selector:
    app: appjs
  ports:
  - name: http
    port: 80
```

Создадим Headless сервис для идентификации экземпляров приложения между собой:
```bash
k create -f appjs-service-headless.yaml
```

Изучим созданный сервис:
```bash
k get svc
k describe svc appjs
```

Создадим описание StatefulSet для нашего приложения:
```bash
vi appjs-statefulset.yaml
```

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: appjs
spec:
  serviceName: appjs
  replicas: 2
  template:
    metadata:
      labels:
        app: appjs
    spec:
      containers:
      - name: appjs
        image: jenoovchi/appjs-pet
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /var/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
```

Создадим StatefulSet для нашего приложения:
```bash
k create -f appjs-statefulset.yaml
```

Изучим процесс развёртывания:
```bash
k get po
k get po
```

Изучим создаваемые модули:
```bash
k get po appjs-0 -o yaml
```

Изучим заявки на постоянные тома:
```bash
k get pvc
k get pvc data-appjs-0 -o yaml
```

Изучим созданный StatefulSet:
```bash
k get statefulset
k describe statefulset appjs
```

Протестируем работу созданного StatefulSet:
```bash
k proxy
curl localhost:8001/api/v1/namespaces/default/pods/appjs-0/proxy/
curl -X POST -d "Hey there! This greeting was submitted to appjs-0." localhost:8001/api/v1/namespaces/default/pods/appjs-0/proxy/
curl localhost:8001/api/v1/namespaces/default/pods/appjs-0/proxy/
curl localhost:8001/api/v1/namespaces/default/pods/appjs-1/proxy/
```

Удалим один из модулей StatefulSet:
```bash
k delete po appjs-0
```

Изучим процесс восстановления модуля:
```bash
k get po
k get po
k get po
```

Проверяем доступность данных после сбоя:
```bash
curl localhost:8001/api/v1/namespaces/default/pods/appjs-0/proxy/
```

Создадим описание общей точки доступа к модулям StatefulSet:
```bash
vi appjs-service-public.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs-public
spec:
  selector:
    app: appjs
  ports:
  - port: 80
    targetPort: 8080
```

Создадим общую точку доступа к модулям StatefulSet:
```bash
k create -f appjs-service-public.yaml
```

Протестируем доступ к StatefulSet с помощью общей точки:
```bash
curl localhost:8001/api/v1/namespaces/default/services/appjs-public/proxy/
```

Изучим DNS запись общей точки доступа:
```bash
k run -it srvlookup --image=tutum/dnsutils --rm --restart=Never -- dig SRV appjs.default.svc.cluster.local
```

Создадим образ приложения, хранящего состояние и получающего данные со всех запущенных экземпляров:
```bash
mkdir appjs-pet-peers-image
cd appjs-pet-peers-image
vi app.js
```

```javascript
const http = require('http');
const os = require('os');
const fs = require('fs');
const dns = require('dns');

const dataFile = "/var/data/appjs.txt";
const serviceName = "appjs.default.svc.cluster.local";
const port = 8080;


function fileExists(file) {
  try {
    fs.statSync(file);
    return true;
  } catch (e) {
    return false;
  }
}

function httpGet(reqOptions, callback) {
  return http.get(reqOptions, function(response) {
    var body = '';
    response.on('data', function(d) { body += d; });
    response.on('end', function() { callback(body); });
  }).on('error', function(e) {
    callback("Error: " + e.message);
  });
}

var handler = function(request, response) {
  if (request.method == 'POST') {
    var file = fs.createWriteStream(dataFile);
    file.on('open', function (fd) {
      request.pipe(file);
      response.writeHead(200);
      response.end("Data stored on pod " + os.hostname() + "\n");
    });
  } else {
    response.writeHead(200);
    if (request.url == '/data') {
      var data = fileExists(dataFile) ? fs.readFileSync(dataFile, 'utf8') : "No data posted yet";
      response.end(data);
    } else {
      response.write("You've hit " + os.hostname() + "\n");
      response.write("Data stored in the cluster:\n");
      dns.resolveSrv(serviceName, function (err, addresses) {
        if (err) {
          response.end("Could not look up DNS SRV records: " + err);
          return;
        }
        var numResponses = 0;
        if (addresses.length == 0) {
          response.end("No peers discovered.");
        } else {
          addresses.forEach(function (item) {
            var requestOptions = {
              host: item.name,
              port: port,
              path: '/data'
            };
            httpGet(requestOptions, function (returnedData) {
              numResponses++;
              response.write("- " + item.name + ": " + returnedData + "\n");
              if (numResponses == addresses.length) {
                response.end();
              }
            });
          });
        }
      });
    }
  }
};

var www = http.createServer(handler);
www.listen(port);
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
docker build -t jenoovchi/appjs-pet-peers .
docker push jenoovchi/appjs-pet-peers
cd ..
```

Изменим описание созданного StatefulSet:
```bash
k edit statefulset appjs
```

```yaml
...
  replicas: 3
...
        image: jenoovchi/appjs-pet-peers
...
```

Изучим созданные модули:
```bash
k get po
```

Удалим старые модули и проверим, что создались модули из новой версии шаблона:
```bash
k delete po appjs-0 appjs-1
k get po
k describe po appjs-1
```

Протестируем работу новой версии приложения:
```bash
curl -X POST -d "The sun is shining" localhost:8001/api/v1/namespaces/default/services/appjs-public/proxy/
curl -X POST -d "The weather is sweet" localhost:8001/api/v1/namespaces/default/services/appjs-public/proxy/
curl localhost:8001/api/v1/namespaces/default/services/appjs-public/proxy/
```

Сэмулируем сбой одного из узлов:
```bash
gcloud compute ssh $(k get node | grep jeno | head -n 1 | awk '{print $1}')
sudo ifconfig eth0 down
```

Проверим статус узлов и экземпляров нашего приложения:
```bash
k get node
k get po
```

Изучим модули в неизвестном состоянии:
```bash
k describe po $(k get po | grep Unknown | head -n 1 | awk '{print $1}')
```

Удалим модуль в неизвестном состоянии:
```bash
k delete po $(k get po | grep Unknown | head -n 1 | awk '{print $1}')
```

Проверим экземпляров нашего приложения:
```bash
k get po
```

Действительно удалим модуль в неизвестном состоянии:
```bash
k delete po $(k get po | grep Unknown | head -n 1 | awk '{print $1}') --force --grace-period 0
```

Проверим экземпляров нашего приложения:
```bash
k get po
```

Проверим доступность данных:
```bash
curl localhost:8001/api/v1/namespaces/default/services/appjs-public/proxy/
```

Перезагрузим узел:
```bash
gcloud compute instances reset $(k get node | grep NotReady | awk '{print $1}')
```