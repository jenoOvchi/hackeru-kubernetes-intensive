# Lab 5: Volumes

## Topic 1: Use Volumes

Создаём образ приложения, хранящего состояние:
```bash
mkdir fortune
cd fortune
vi fortuneloop.sh
```

```yaml
#!/bin/bash
trap "exit" SIGINT
mkdir /var/htdocs
while :
do
  echo $(date) Writing fortune to /var/htdocs/index.thml
  /usr/games/fortune > /var/htdocs/index.html
  sleep 10
done
```

```bash
vi Dockerfile
```

```yaml
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT /bin/fortuneloop.sh
```

```bash
chmod 777 fortuneloop.sh
docker build -t jenoovchi/fortune .
docker push jenoovchi/fortune
cd ..
```

Создаём описание приложения, хранящего состояние:
```bash
vi fortune-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: jenoovchi/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

Создаём приложение, хранящее состояние:
```bash
k create -f fortune-pod.yaml
```

Изучаем созданное приложение:
```bash
k get po
k describe po fortune
```

Тестируем созданное приложение:
```bash
k port-forward fortune 8080:80
curl http://localhost:8080
```

Настраиваем хранение данных в оперативной памяти:
```bash
vi fortune-pod.yaml
```

```yaml
...
  volumes:
  - name: html
    emptyDir:
      medium: Memory
```

Пересоздаём приложение с изменённым типом хранилища:
```bash
k delete po fortune
k create -f fortune-pod.yaml
```

Изучаем созданное приложение:
```bash
k get po
k describe po fortune
```

Создаём описание приложения с спользованием репозитория GitHub https://github.com/jenoOvchi/appjs-website-example в качестве источника данных для тома:
```bash
vi gitrepo-volume-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-volume-pod
spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    gitRepo:
      repository: https://github.com/jenoOvchi/appjs-website-example.git
      revision: master
      directory: .
```

Создаём описанное приложение:
```bash
k create -f gitrepo-volume-pod.yaml
```

Изучаем созданное приложение:
```bash
k get po
k describe po gitrepo-volume-pod
```

Тестируем созданное приложение:
```bash
k port-forward gitrepo-volume-pod 8080:80
curl http://localhost:8080
```

Изучаем модуль fluentd на наличие постоянных томов:
```bash
k get pods --namespace kube-system
k describe po $(k get po --namespace kube-system | grep fluentd | grep -v scaler | head -n 1 | awk '{print $1}') --namespace kube-system
```

Удаляем лишние модули:
```bash
k delete po --all
```

Создаём диск в облаке для размещения постоянного тома:
```bash
gcloud container clusters list
gcloud compute disks create --size=1GiB --zone=$(gcloud container clusters list | grep jeno | awk '{print $2}') mongodb
```

Создаём описание базы данных с спользованием диска в облаке Google в качестве тома:
```bash
vi mongodb-pod-gcepd.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    gcePersistentDisk:
      pdName: mongodb
      fsType: ext4
```

Создаём описанную базу данных:
```bash
k create -f mongodb-pod-gcepd.yaml
```

Изучаем созданную базу данных:
```bash
k get po
k describe po mongodb
```

Сохраняем данные в созданной базе:
```bash
k exec -it mongodb mongo
> use mystore
> db.foo.insert({name: 'foo'})
> db.foo.find()
> exit
```

Удаляем созданный экземпляр базы данных:
```bash
k delete pod mongodb
```

Пересоздаём базу данных и проверяем, что данные сохранились:
```bash
k create -f mongodb-pod-gcepd.yaml
k get po
k exec -it mongodb mongo
> use mystore
> db.foo.find()
> exit
```

Создаём описание базы данных с спользованием диска в облаке Amazon в качестве тома:
```bash
vi mongodb-pod-aws.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb-aws
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    awsElasticBlockStore:
      volumeID: my-volume
      fsType: ext4
```

Создаём описание базы данных с спользованием директории на NFS сервере в качестве тома:
```bash
vi mongodb-pod-nfs.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb-aws
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    nfs:
      server: 1.2.3.4
      path: /some/path
```

Удаляем созданные модули:
```bash
k delete po --all
```

## Topic 2: Use PersistentVolume and PersistentVolumeClaims

Создаём описание постоянного тома:
```bash
vi mongodb-pv-gcepd.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    pdName: mongodb
    fsType: ext4
```

Создаём постоянный том:
```bash
k create -f mongodb-pv-gcepd.yaml
```

Изучаем созданный том:
```bash
k get pv
k describe pv mongodb-pv
```

Создаём описание заявки на постоянный том:
```bash
vi mongodb-pvc-gcepd.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```

Создаём заявку на постоянный том:
```bash
k create -f mongodb-pvc-gcepd.yaml
```

Изучаем созданную заявку на том и её связь с томом:
```bash
k get pvc
k describe pvc mongodb-pvc
k get pv
```

Создаём описание базы данных с спользованием заявки на постоянный том:
```bash
vi mongodb-pod-pvc.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

Создаём базу данных с спользованием заявки на постоянный том:
```bash
k create -f mongodb-pod-pvc.yaml
```

Изучаем созданную базу данных:
```bash
k get po mongodb
k describe po mongodb
```

Проверяем сохранность данных в томе:
```bash
k exec -it mongodb mongo
> use mystore
> db.foo.find()
> exit
```

Удаляем созданную базу данных и заявку на постоянный том:
```bash
k delete pod mongodb
k delete pvc mongodb-pvc
```

Пробуем создать заявку на постоянный том заново:
```bash
k create -f mongodb-pvc-gcepd.yaml
```

Изучаем созданную заявку на постоянный том и том предыдущей заявки:
```bash
k get pvc
k get pv
```

Удаляем заявку на постоянный том:
```bash
k delete pvc mongodb-pvc
```

## Topic 3: Use DinamicProvisioning

Созваём описание класса хранилища для динамического предоставления томов:
```bash
vi storageclass-fast-gcepd.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zone: europe-north1-a
```

Создаём класс хранилища для динамического предоставления томов:
```bash
k create -f storageclass-fast-gcepd.yaml
```

Изучаем созданный класс хранилища:
```bash
k describe sc fast
```

Создаём описание заявки на постоянный том с запросом на динамическое предоставление тома:
```bash
vi mongodb-pvc-dp.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
```

Создаём заявку на постоянный том с запросом на динамическое предоставление тома:
```bash
k create -f mongodb-pvc-dp.yaml
```

Изучаем созданную заявку и том:
```bash
k get pvc mongodb-pvc
k describe pvc mongodb-pvc
k get pv
gcloud compute disks list
```

Изучаем созданные классы хранилища:
```bash
k get sc
k get sc standard -o yaml
```

Создаём описание заявки на постоянный том с использованием класса хранилища по умолчанию:
```bash
vi mongodb-pvc-nostorageclass.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc2
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
```

Создаём заявку на постоянный том с использованием класса хранилища по умолчанию:
```bash
k create -f mongodb-pvc-nostorageclass.yaml
```

Изучаем созданную заявку и том:
```bash
k get pvc mongodb-pvc2
k describe pvc mongodb-pvc2
k get pv $(k get pvc | grep mongodb-pvc2 | head -n 1 | awk '{print $3}')
gcloud compute disks list
```

Удаляем созданные модули:
```bash
k delete po --all
```