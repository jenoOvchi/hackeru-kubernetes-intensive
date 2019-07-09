# Lab 4: Ingresses

## Topic 1: Use Ingress

### (Опционально) Устанавливаем Ingress контроллер в Minikube
```bash
minikube addons list
minikube addons enable ingress
k get po --all-namespaces
```

Создаём описание внешней точки доступа:
```bash
vi appjs-ingress.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: appjs
spec:
  rules:
  - host: appjs.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: appjs-nodeport
          servicePort: 80
```

Создаём внешнюю точку доступа:
```bash
k create -f appjs-ingress.yaml
```

Изучаем внешнюю точку доступа:
```bash
k get ingresses 
k describe ingresses appjs
```

Добавляем внешнюю точку доступа в список хостов ОС:
```bash
k get ingresses | grep appjs | awk '{print $3}'
sudo vi /etc/hosts (for Windows C:\windows\system32\drivers\etc\hosts)
```

```yaml
...
(k get ingresses | grep appjs | awk '{print $3}') appjs.example.com
...
```

Проверяем доступность приложения по внешней точке доступа:
```bash
curl http://appjs.example.com
```

Запускаем дополнительное приложение для демонстрации возможностей внешних точек доступа:
```bash
k run nginx --image=maninzoo/example-nginx2 --generator=run/v1 --port=80
```

Изучаем созданное приложение:
```bash
k get rc nginx
k describe rc nginx
```

Создаём описание сервиса для развёрнутого приложения:
```bash
vi nginx.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
  selector:
    run: nginx
```

Создаём сервис для развёрнутого приложения:
```bash
k create -f nginx.yaml
```

Изучаем созданный сервис:
```bash
k get svc nginx
k describe svc nginx
```
Проверяем доступность приложения по созданному сервису:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl http://$(k get svc | grep nginx | awk '{print $3}')
```

Создаём описание сервиса для развёрнутого приложения с доступом на порте узла:
```bash
vi nginx-nodeport.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30124
  selector:
    run: nginx
```

Создаём сервис для развёрнутого приложения с доступом на порте узла:
```bash
k create -f nginx-nodeport.yaml
```

Изучаем созданный сервис:
```bash
k get svc nginx-nodeport
k describe svc nginx-nodeport
```

Добавляем правило в фаервол Google Cloud Platform для предоставления доступа к сервису:
```bash
gcloud compute firewall-rules create nginx-svc-rule --allow=tcp:30124
```

Получаем адрес узла:
```bash
k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'
```

Проверяем доступность приложения по созданному сервису (можно в браузере):
```bash
curl http://$(k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}' | awk '{print $1}'):30124
```

Создаём описание внешней точки доступа для созданны сервисов:
```bash
vi appjs-nginx-ingress.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: appjs-nginx
spec:
  rules:
  - host: appjs-nginx.example.com
    http:
      paths:
      - path: /appjs
        backend:
          serviceName: appjs-nodeport
          servicePort: 80
      - path: /nginx
        backend:
          serviceName: nginx-nodeport
          servicePort: 80
```

Создаём внешнюю точку доступа для созданны сервисов:
```bash
k apply -f appjs-nginx-ingress.yaml
```

Изучаем внешнюю точку доступа для созданны сервисов:
```bash
k get ingresses
k describe ingresses appjs-nginx
```

Добавляем внешнюю точку доступа в список хостов ОС:
```bash
sudo vi /etc/hosts (for Windows C:\windows\system32\drivers\etc\hosts)
```

```yaml
...
(k get ingresses | grep appjs-nginx | awk '{print $3}') appjs-nginx.example.com
...
```

Проверяем доступность приложений по внешней точке доступа (можно в браузере):
```bash
curl http://appjs-nginx.example.com/appjs
curl http://appjs-nginx.example.com/nginx
```

Создаём описание внешней точки доступа для созданны сервисов с разными путями:
```bash
vi appjs-nginx-ingress-hosts.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: appjs-nginx-hosts
spec:
  rules:
  - host: appjs-new.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: appjs-nodeport
          servicePort: 80
  - host: nginx-new.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-nodeport
          servicePort: 80
```

Создаём внешнюю точку доступа для созданны сервисов с разными путями:
```bash
k apply -f appjs-nginx-ingress-hosts.yaml
```

Изучаем внешнюю точку доступа:
```bash
k get ingresses
k describe ingresses appjs-nginx-hosts
```

Добавляем внешнюю точку доступа в список хостов ОС:
```bash
vi /etc/hosts (for Windows C:\windows\system32\drivers\etc\hosts)
```

```yaml
(k get ingresses | grep appjs-new | awk '{print $3}') appjs-new.example.com
(k get ingresses | grep nginx-new | awk '{print $3}') nginx-new.example.com
```

Проверяем доступность приложений по внешним точкам доступа (можно в браузере):
```bash
curl http://appjs-new.example.com
curl http://nginx-new.example.com
```

Устанавливаем openssl:
```bash
brew install openssl
```

Создаём ключ и сертификат для нашей точки доступа:
```bash
openssl -h
openssl genrsa -out tls.key 2048
openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj '/CN=appjs.example.com'
```

Создаём секрет с ключом и сертификатом для нашей точки доступа:
```bash
k create secret tls tls-secret --cert=tls.cert --key=tls.key
```

Изучаем созданный секрет:
```bash
k get secrets
k describe secret tls-secret
```

Создаём описание внешней точки доступа с использованием TLS:
```bash
vi appjs-ingress-tls.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: appjs
spec:
  tls:
  - hosts:
    - appjs.example.com
    secretName: tls-secret
  rules:
  - host: appjs.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: appjs-nodeport
          servicePort: 80
```

Создаём внешнюю точку доступа с использованием TLS:
```bash
k apply -f appjs-ingress-tls.yaml
```

Изучаем созданную точку доступа:
```bash
k get ingresses
k describe ingress appjs
```

Добавляем внешнюю точку доступа в список хостов ОС:
```bash
vi /etc/hosts (for Windows C:\windows\system32\drivers\etc\hosts)
```

```yaml
(k get ingresses | grep appjs | awk '{print $3}') appjs.example.com
```

Проверяем доступность приложения по внешней точке доступа (можно в браузере):
```bash
curl -k -v https://appjs.example.com
```

Создаём ключ и сертификат для подписи:
```bash
openssl req -x509 -sha256 -newkey 2048 -keyout ca.key -out ca.crt -days 356 -nodes -subj '/CN=Hackeru Cert Authority'
```

Создаём ключ и сертификат для сервера, подписанный созданным сертификатом:
```bash
openssl req -new -newkey 2048 -keyout server.key -out server.csr -nodes -subj '/CN=appjs.example.com'
openssl x509 -req -sha256 -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt
```

Создаём ключ и сертификат для сервера, подписанный созданным сертификатом:
```bash
openssl req -new -newkey 2048 -keyout client.key -out client.csr -nodes -subj '/CN=Hackeru'
openssl x509 -req -sha256 -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 02 -out client.crt
```

Создаём секрет с ключом и сертификатом для нашей точки доступа:
```bash
k create secret generic my-certs --from-file=tls.crt=server.crt --from-file=tls.key=server.key --from-file=ca.crt=ca.crt
```

Изучаем созданный секрет:
```bash
k get secret my-certs
k describe secret my-certs
```

Создаём описание приложения для демонстрации двусторонней аутентификации:
```bash
vi echoserver.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: echoserver
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
      - name: echoserver
        image: gcr.io/kubernetes-e2e-test-images/echoserver:2.1
        ports:
        - containerPort: 8080
```

Создаём приложение для демонстрации двусторонней аутентификации:
```bash
k apply -f echoserver.yaml
```

Изучаем развёрнутое приложение:
```bash
k get deploy
k get pods
```

Создаём описание сервиса для нашего приложения:
```bash
vi echoserver-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echoserver-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  type: NodePort
  selector:
    app: echoserver
```

Создаём сервис для нашего приложения:
```bash
k apply -f echoserver-svc.yaml
```

Изучаем созданный сервис:
```bash
k get svc
```

Создаём описание внешней точки доступа для нашего приложения:
```bash
vi echoserver-ingress.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-verify-client: \"on\"
    nginx.ingress.kubernetes.io/auth-tls-secret: \"default/my-certs\"
  name: echoserver-ingress
  namespace: default
spec:
  rules:
  - host: appjs.example.com
    http:
      paths:
      - backend:
          serviceName: echoserver-svc
          servicePort: 80
        path: /
  tls:
  - hosts:
    - appjs.example.com
    secretName: my-certs
```

Создаём внешнюю точку доступа для нашего приложения:
```bash
k apply -f echoserver-ingress.yaml
```

Изучаем созданную внешнюю точку доступа:
```bash
k get ingresses
```

Добавляем внешнюю точку доступа в список хостов ОС (если её нет):
```bash
vi /etc/hosts (for Windows C:\windows\system32\drivers\etc\hosts)
```

```yaml
(k get ingresses | grep appjs | awk '{print $3}') appjs.example.com
```

Тестируем двустороннюю аутентификацию:
```bash
curl https://appjs.example.com/ -k
curl https://appjs.example.com/ --cert client.crt --key client.key -k
```

## Topic 2: Use ReadynessProbe

Изменяем описание контроллера репликации:
```bash
k edit rc appjs
```

```yaml
apiVersion: v1
kind: ReplicationController
...
spec:
  ...
  template:
    ...
    containers:
      - name: appjs
        image: jenoovchi/appjs
        readinessProbe:
          exec:
            command:
            - ls
            - /var/ready
```

Удаляем модули для обновления конфигурации:
```bash
k delete po --all
```

Проверяем конфигурацию созданных модулей:
```bash
k get po
k describe rc appjs
```

Переводим один из созданных модулей в состояние готовности:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- touch /var/ready
```

Проверяем готовность модулей:
```bash
k get po $(k get po | grep appjs | head -n 1 | awk '{print $1}')
k describe po $(k get po | grep appjs | head -n 1 | awk '{print $1}')
```

Проверяем список конечных точек сервиса:
```bash
k get endpoints appjs
```
Проверяем доступность приложения:
```bash
curl http://$(k get ingresses appjs | grep appjs | awk '{print $2}')
curl http://$(k get ingresses appjs | grep appjs | awk '{print $2}')
curl http://$(k get ingresses appjs | grep appjs | awk '{print $2}')
```

## Topic 3: Use HeadlessServices

Создаём описание сервиса без единого IP адреса:
```bash
vi appjs-svc-headless.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs-headless
spec:
  clusterIP: None
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: appjs
```

Создаём сервис без единого IP адреса:
```bash
k create -f appjs-svc-headless.yaml
```

Изучаем созданны сервис:
```bash
k get svc
k describe svc appjs-headless
```

Переводим ещё один из созданных модулей в состояние готовности:
```bash
k exec $(k get po | grep -v 1/1 | grep appjs | head -n 1 | awk '{print $1}') -- touch /var/ready
```

Изучаем конечные точки созданного сервиса:
```bash
k get endpoints appjs-headless
```

Запускаем модуль для анализа сети:
```bash
k run dnsutils --image=tutum/dnsutils --generator=run-pod/v1 --command -- sleep infinity
```

Изучаем сервисы с помощью сетевой утилиты:
```bash
k exec dnsutils nslookup appjs-headless
k get svc
k exec dnsutils nslookup appjs
```

Создаём описание сервиса без единого IP адреса, не исключающего неготовые модули:
```bash
vi appjs-svc-headless-all.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs-headless-all
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  clusterIP: None
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: appjs
```

Создаём сервис без единого IP адреса, не исключающий неготовые модули:
```bash
k create -f appjs-svc-headless-all.yaml
```

Изучаем созданный сервис:
```bash
k get endpoints appjs-headless-all
k exec dnsutils nslookup appjs-headless-all
```

Удаляем созданные модули:
```bash
k delete po --all
```