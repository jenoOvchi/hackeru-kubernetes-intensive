# Lab 6: Configuration

## Topic 1: Use Arguments

Создадим образ приложения, использующего аргументы для конфигурации:
```bash
mkdir fortune-args
cd fortune-args
vi fortuneloop.sh
```

```bash
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1
echo Configured to generate new fortune every $INTERVAL seconds
mkdir /var/htdocs
while :
do
  echo $(date) Writing fortune to /var/htdocs/index.thml
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```

```bash
vi Dockerfile
```

```yaml
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]
```

```bash
chmod 777 fortuneloop.sh
docker build -t jenoovchi/fortune:args .
docker run -it jenoovchi/fortune:args
docker run -it jenoovchi/fortune:args 15
docker push jenoovchi/fortune:args
cd ..
```

Создадим описание модуля, переписывающего команду и её аргументы:
```bash
vi pod-args-example.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
  - image: some/image
    command: ["/bin/command"]
    args: ["arg1", "arg2", "arg3"]
```

Создадим описание модуля с переданным аргументом команды запуска:
```bash
vi fortune-pod-args.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
  - image: jenoovchi/fortune:args
    args: ["2"]
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

Создадим модуль с переданным аргументом команды запуска:
```bash
k create -f fortune-pod-args.yaml
```

Изучим созданный модуль:
```bash
k get po
k describe po fortune2s
```

Протестируем созданный модуль:
```bash
k port-forward fortune2s 8080:80
curl http://localhost:8080
```

Удалим созданные модули:
```bash
k delete po --all
```

## Topic 2: Use Environment Variables

Создадим образ приложения, использующего переменные окружения для конфигурации:
```bash
mkdir fortune-env
cd fortune-env
cp ../fortune-args/* .
vi fortuneloop.sh
```

```bash
#!/bin/bash
trap "exit" SIGINT
echo Configured to generate new fortune every $INTERVAL seconds
mkdir /var/htdocs
while :
do
  echo $(date) Writing fortune to /var/htdocs/index.thml
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```

```bash
chmod 777 fortuneloop.sh
docker build -t jenoovchi/fortune:env .
docker run -it -e "INTERVAL=30" jenoovchi/fortune:env
docker push jenoovchi/fortune:env
cd ..
```

Создадим описание модуля, использующего переменные окружения для конфигурации:
```bash
vi fortune-pod-env.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune30s
spec:
  containers:
  - image: jenoovchi/fortune:env
    env:
    - name: INTERVAL
      value: "30"
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

Создадим модуль, использующий переменные окружения для конфигурации:
```bash
k create -f fortune-pod-env.yaml
```

Изучим созданный модуль:
```bash
k get po
k describe po fortune30s
```

Протестируем созданный модуль:
```bash
k port-forward fortune30s 8080:80
curl http://localhost:8080
```

Продемонстрируем описание модуля с вложенным использованием переменных:
```bash
vi env-reuse.yaml
```

```yaml
...
    env:
    - name: FIRST_VAR
      value: "foo"
    - name: SECOND_VAR
      value: "$(FIRST_VAR)bar"
...
```

Удалим созданные модули:
```bash
k delete po --all
```

## Topic 3: Use ConfigMaps

Создадим справочник конфигурации:
```bash
k create configmap fortune-config --from-literal=sleep-interval=25
```

Изучим созданный справочник конфигурации:
```bash
k get configmaps
k describe configmap fortune-config
```

Создадим составной справочник конфигурации:
```bash
k create configmap myconfigmap --from-literal=foo=bar --from-literal=bar=baz --from-literal=one=two
```

Изучим созданный справочник конфигурации:
```bash
k get configmaps
k describe configmap myconfigmap
k get configmap myconfigmap -o yaml
```

Создадим описание справочника конфигурации:
```bash
vi fortune-config.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortune-config-2
data:
  sleep-interval: "25"
```

Создадим справочник конфигурации из описания:
```bash
k create -f fortune-config.yaml
```

Изучим созданный справочник конфигурации:
```bash
k get configmaps
k describe configmap fortune-config-2
```

Создадим файл конфигурации Nginx:
```bash
vi nginx.conf
```

```yaml
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 15;
    types_hash_max_size 2048;
    server_tokens off;
    
    include /etc/nginx/mime.types;
    default_type text/javascript;

    access_log off;
    error_log /var/log/nginx/error.log;
    
    gzip on;
    gzip_min_length 100;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    
    client_max_body_size 8M;
    
    server {
        listen 80;
        root /var/www;
    }
}
```

Создадим справочник конфигурации из файла:
```bash
k create configmap my-config --from-file=nginx.conf
```

Изучим созданный справочник конфигурации:
```bash
k describe configmap my-config
```

Создадим справочник конфигурации из файла с изменённым именем файла:
```bash
k create configmap my-config-2 --from-file=customkey=nginx.conf
```

Изучим созданный справочник конфигурации:
```bash
k describe configmap my-config-2
```

Создадим справочник конфигурации из директории:
```bash
k create configmap my-config-3 --from-file=./fortune
```

Изучим созданный справочник конфигурации:
```bash
k describe configmap my-config-3
```

Создадим справочник конфигурации, комбинируя все вышеприведённые способы:
```bash
k create configmap my-config-4 --from-file=nginx.conf --from-file=bar=tls.cert --from-file=./fortune --from-literal=some=thing
```

Изучим созданный справочник конфигурации:
```bash
k describe configmap my-config-4
```

Создадим описание модуля, использующего справочника конфигурации для заполнения значений переменных:
```bash
vi fortune-pod-env-configmap.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: jenoovchi/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
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

Создадим модуль, использующий справочник конфигурации для заполнения значений переменных:
```bash
k create -f fortune-pod-env-configmap.yaml
```

Изучим созданный модуль:
```bash
k get po
k describe po fortune-env-from-configmap
```

Создадим описание модуля, использующего префиксы для задания переменных окружения:
```bash
vi pod-env-prefix-example.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
  - image: some/image
    envFrom:
    - prefix: CONFIG_
      configMapRef:
        name: my-config-map
```

Создадим описание модуля, использующего значения скравочников конфигурации для задания аргументов команды:
```bash
vi fortune-pod-args-configmap.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-args-from-configmap
spec:
  containers:
  - image: jenoovchi/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
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

Создадим модуль, использующий значения скравочников конфигурации для задания аргументов команды:
```bash
k create -f fortune-pod-args-configmap.yaml
```

Изучим созданный модуль:
```bash
k get po
k describe po fortune-args-from-configmap
```

Удалим созданный справочник конфигурации:
```bash
k delete configmap fortune-config
```

Создадим директорию и в ней файлы для справочника конфигурации с целью использования сжатия gzip:
```bash
mkdir configmap-files
cd configmap-files
vi my-nginx-config.conf
```

```yaml
server {
  listen 80;
  server_name www.appjs-example.com;

  gzip on;
  gzip_types text/plain application/xml;

    
  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    }
}
```

```bash
vi sleep-interval
```

```yaml
25
```

```bash
cd ..
```

Создадим справочник конфигурации из директории:
```bash
k create configmap fortune-config --from-file=configmap-files
```

Изучим созданный справочник конфигурации:
```bash
k get configmap fortune-config -o yaml
```

Создадим описание модуля, использующего скравочник конфигурации как постоянный том:
```bash
vi fortune-pod-configmapvolume.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
spec:
  containers:
  - image: jenoovchi/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
```

Создадим модуль, использующий скравочник конфигурации как постоянный том:
```bash
k create -f fortune-pod-configmapvolume.yaml
```

Изучим созданный модуль:
```bash
k get po
k describe po fortune-configmap-volume
```

Протестируем созданный модуль:
```bash
k port-forward fortune-configmap-volume 8080:80
curl -H "Accept-Encoding: gzip" -I localhost:8080
k exec fortune-configmap-volume -c web-server ls /etc/nginx/conf.d
```

Создадим описание модуля, тонко настраивающего скравочник конфигурации как постоянный том:
```bash
vi fortune-pod-configmap-volume-with-items.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume-with-items
spec:
  containers:
  - image: jenoovchi/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:
      - key: my-nginx-config.conf
        path: gzip.conf
```

Создадим модуль, тонко настраивающий скравочник конфигурации как постоянный том:
```bash
k create -f fortune-pod-configmap-volume-with-items.yaml
```

Изучим созданный модуль:
```bash
k get po
k describe po fortune-configmap-volume-with-items
```

Протестируем созданный модуль:
```bash
k exec fortune-configmap-volume-with-items -c web-server ls /etc/nginx/conf.d
```

Удалим созданные модули:
```bash
k delete po --all
```

Создадим описание модуля, некорректно переписывающего конкретный файл:
```bash
vi fortune-pod-configmap-volume-without-subpath.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume-without-subpath
spec:
  containers:
  - image: jenoovchi/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    - name: config
      mountPath: /etc/nginx
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
```

Создадим модуль, некорректно переписывающий конкретный файл:
```bash
k create -f fortune-pod-configmap-volume-without-subpath.yaml
```

Изучим созданный модуль:
```bash
k get po
k logs fortune-configmap-volume-without-subpath -c web-server
k describe po fortune-configmap-volume-without-subpath
```

Создадим описание модуля, корректно переписывающего конкретный файл:
```bash
vi fortune-pod-configmap-volume-with-subpath.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume-with-subpath
spec:
  containers:
  - image: jenoovchi/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    - name: config
      mountPath: /etc/nginx/myconfig.conf
      subPath: myconfig.conf
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:
      - key: my-nginx-config.conf
        path: myconfig.conf
```

Создадим модуль, корректно переписывающего конкретный файл:
```bash
k create -f fortune-pod-configmap-volume-with-subpath.yaml
```

Изучим созданный модуль:
```bash
k get po
k describe po fortune-configmap-volume-with-subpath
```

Протестируем созданный модуль:
```bash
k exec fortune-configmap-volume-with-subpath -c web-server ls /etc/nginx
k exec fortune-configmap-volume-defaultmode -c web-server cat /etc/nginx/myconfig.conf
```

Создадим описание модуля с указанием прав доступа к его файлам:
```bash
vi fortune-pod-configmap-volume-defaultmode.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume-defaultmode
spec:
  containers:
  - image: jenoovchi/fortune:args
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      defaultMode: 0400
```

Создадим модуль с указанием прав доступа к его файлам:
```bash
k create -f fortune-pod-configmap-volume-defaultmode.yaml
```

Изучим созданный модуль:
```bash
k get po
k describe po fortune-configmap-volume-defaultmode
```
Может не работать - есть Issue на багтрекере Google Cloud Platform.

Изменим файл конфигурации Nginx:
```bash
k edit configmap fortune-config
```

```yaml
...
  gzip off;
...
```

Протестируем обновление справочника конфигураций:
```bash
k exec fortune-configmap-volume-defaultmode -c web-server cat /etc/nginx/conf.d/my-nginx-config.conf
k exec fortune-configmap-volume-defaultmode -c web-server -- nginx -s reload
k port-forward fortune-configmap-volume-defaultmode 8080:80
curl -I localhost:8080
k exec -it fortune-configmap-volume-defaultmode -c web-server -- ls -lA /etc/nginx/conf.d
```

Удалить созданные модули:
```bash
k delete po --all
```


## Topic 4: Use Secrets

Изучим созданные секреты:
```bash
k get secrets
k describe po $(k get po | grep appjs | head -n 1 | awk '{print $1}')
k describe secrets $(k get secrets | grep default | head -n 1 | awk '{print $1}')
k describe po $(k get po | grep appjs | head -n 1 | awk '{print $1}') | grep token
```

Изучим монтирование стандартных секретов Kubernetes в каждый из подов:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') ls /var/run/secrets/kubernetes.io/serviceaccount/
```

Создадим файлы для генерации секрета:
```bash
openssl genrsa -out https.key 2048
openssl req -new -x509 -key https.key -out https.cert -days 365 -subj '/CN=www.appjs-example.com'
echo bar > foo
```

Сгенерируем секрет на основе созданных файлов:
```bash
k create secret generic fortune-https --from-file=https.key --from-file=https.cert --from-file=foo
```

Изучим созданный секрет:
```bash
k get secret fortune-https -o yaml
k get configmap fortune-config -o yaml
k describe secret fortune-https
```

Создадим секрет без использования кодирования в base64 для одного из элементов:
```bash
vi secret-stringdata.yaml
```

```yaml
kind: Secret
apiVersion: v1
stringData:
  foo: plain text
data:
  https.cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURDekNDQWZPZ0F3...
  https.key: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklpSjkuZXlKcGMzTWlPaU...
```

Изменяем справочник конфигураций нашего приложения для взаимодействия с использованием SSL:
```bash
k edit configmap fortune-config
```

```yaml
...
data:
  my-nginx-config.conf: |
    server {
      listen 80;
      listen 443 ssl;
      server_name www.appjs-example.com;
      ssl_certificate certs/https.cert;
      ssl_certificate_key certs/https.key;
      ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers HIGH:!aNULL:!MD5;
            
      location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        }
    }
  sleep-interval: |
...
```

Создаём описание модуля для взаимодействия с использованием SSL:
```bash
vi fortune-pod-https.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  containers:
  - image: jenoovchi/fortune:env
    name: html-generator
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:
      - key: my-nginx-config.conf
        path: https.conf
  - name: certs
    secret:
      secretName: fortune-https
```

Создаём модуль для взаимодействия с использованием SSL:
```bash
k create -f fortune-pod-https.yaml
```

Изучим созданный модуль:
```bash
k get po 
k describe po fortune-https
```

Протестируем созданный модуль:
```bash
k port-forward fortune-https 8443:443
curl https://localhost:8443 -k
curl https://localhost:8443 -k -v
k exec fortune-https -c web-server -- mount | grep certs
```

Покажем использование секретов для заполнения переменных окружения:
```bash
vi env-secret.yaml
```

```yaml
...
env:
- name: FOO_SECRET
  valueFrom:
    secretKeyRef:
      name: fortune-https
      key: foo
...
```

Создание секрета для доступа в приватное хранилище образов Docker:
```bash
k create secret docker-registry mydockerhubsecret --docker-username=myusername --docker-password=mypassword --docker-email=my.email@provider.com
```

Изучим созданный секрет:
```bash
k describe secret mydockerhubsecret
```

Создадим описания модуля с указанием секрета для загрузки образов из приватного хранилища образов Docker:
```bash
vi pod-with-private-image.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:
  - name: mydockerhubsecret
  containers:
  - image: username/private:tag
    name: main
```

Удаляем созданные модули:
```bash
k delete po --all
```