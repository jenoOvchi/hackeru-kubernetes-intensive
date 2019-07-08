# Lab 3: Services

## Topic 1: Use Services

Создаём контроллер репликации:
```bash
k create -f appjs-rc.yaml
```

Создаём описание сервиса:
```bash
vi appjs-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: appjs
```

Создаём сервис:
```bash
k create -f appjs-svc.yaml
```

Изучаем созданный сервис:
```bash
k get svc
k describe svc appjs
```

Проверяем доступность нашего приложения, вызывая его из контейнера одного из модулей:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep appjs | awk '{print $3}')
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep appjs | awk '{print $3}')
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep appjs | awk '{print $3}')
```

Демонстрируем необходимость двух дефисов:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') - curl -s http://$(k get svc | grep appjs | awk '{print $3}')
```

Создаём описание сервиса с функцией сохранения пользовательских сессий:
```bash
vi appjs-svc-with-sa.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs-with-sa
spec:
  sessionAffinity: ClientIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: appjs
```

Создаём сервис с функцией сохранения пользовательских сессий:
```bash
k create -f appjs-svc-with-sa.yaml
```

Изучаем созданный сервис:
```bash
k get svc
k describe svc appjs-with-sa
```

Проверяем работу функциии сохранения пользовательских сессий:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep appjs-with-sa | awk '{print $3}')
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep appjs-with-sa | awk '{print $3}')
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep appjs-with-sa | awk '{print $3}')
```

Создаём дополнительное приложение для демонстрации работы нескольких портов в одном сервисе:
```bash
k run tomcat-mysql --image=aallam/tomcat-mysql --port=3306 --port=8080 --env="MYSQL_PASSWORD=root" --generator=run/v1
```

Открываем дополнительный порт у контейнера:
```bash
k edit rc tomcat-mysql
```

```yaml
...
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 3306
          protocol: TCP
...
```

Изучаем созданный контроллер репликации:
```bash
k describe rc tomcat-mysql
```

Создаём описание сервиса с двумя открытыми портами:
```bash
vi tomcat-mysql-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-mysql-svc
spec:
  ports:
  - name: tomcat
    port: 80
    targetPort: 8080
  - name: mysql
    port: 3306
    targetPort: 3306
  selector:
    run: tomcat-mysql
```

Создаём сервис с двумя открытыми портами:
```bash
k create -f tomcat-mysql-svc.yaml
```

Изучаем созданный сервис:
```bash
k get svc
k describe svc tomcat-mysql-svc
```

Проверяем доступность приложений на обоих портах:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep tomcat-mysql | awk '{print $3}')
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep tomcat-mysql | awk '{print $3}'):3306
```

Создаём описание приложения с двумя открытыми проименованными портами:
```bash
vi tomcat-mysql-v2.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tomcat-mysql-svc-v2
  labels:
    app: tomcat-mysql-svc-v2
spec:
  containers:
  - image: aallam/tomcat-mysql
    name: main
    env:
    - name: MYSQL_PASSWORD
      value: root
    ports:
    - name: tomcat
      containerPort: 8080
    - name: mysql
      containerPort: 3306
```

Создаём приложение с двумя открытыми проименованными портами:
```bash
k create -f tomcat-mysql-v2.yaml
```

Изучаем созданное приложение:
```bash
k get pods
```

Создаём описание сервиса с двумя открытыми портами, выбранными по именам:
```bash
vi tomcat-mysql-by-name.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-mysql-svc-by-name
spec:
  ports:
  - name: tomcat
    port: 80
    targetPort: tomcat
  - name: mysql
    port: 3306
    targetPort: mysql
  selector:
    app: tomcat-mysql-svc-v2
```

Создаём сервис с двумя открытыми портами, выбранными по именам:
```bash
k create -f tomcat-mysql-by-name.yaml
```

Изучаем созданный сервис:
```bash
k get svc
k describe svc tomcat-mysql-svc-by-name
```

Проверяем доступность приложений на обоих портах:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep tomcat-mysql-svc-by-name | awk '{print $3}')
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') -- curl -s http://$(k get svc | grep tomcat-mysql-svc-by-name | awk '{print $3}'):3306
```

Удаляем созданные приложения без контроллера репликации:
```bash
k delete po --all
```

Изучаем переменные окружения модуля:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') env
```

Находим модуль с сервером доменных имён Kubernetes:
```bash
k get po -n kube-system | grep dns
```

Проверяем доступность приложений по разным доменным именам:
```bash
k exec -it $(k get po | grep appjs | head -n 1 | awk '{print $1}') bash
curl http://appjs.default.svc.cluster.local
curl http://appjs.default
curl http://appjs
```

Изучаем файл разрешения доменных имён и пингуем наше приложение:
```bash
cat /etc/resolv.conf
ping appjs
```

Удаляем контроллер репликации:
```bash
k delete rc tomcat-mysql
```

## Topic 2: Use Endpoints

Изучаем сервис и его конечные точки:
```bash
k describe svc appjs
k get endpoints appjs
```

Создаём описание внешнего сервиса:
```bash
vi external-service.yaml
```


```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```

Создаём внешний сервис:
```bash
k create -f external-service.yaml
```

Изучаем внешний сервис:
```bash
k get svc
k describe svc external-service
```

Создаём описание внешних конечных точек:
```bash
vi external-service-endpoints.yaml
```

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:
    - ip: 11.11.11.11
    - ip: 22.22.22.22
    ports:
    - port: 80
```

Создаём внешние конечные точки:
```bash
k create -f external-service-endpoints.yaml
```
Изучаем внешние конечные точки:
```bash
k get endpoints
k describe svc external-service
```

Удаляем модули для обновления переменных окружения:
```bash
k delete po --all
```

Проверяем переменные окружения модуля:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') env
```

Создаём описание сервиса внешней базы данных (описана по следующей ссылке: https://rnacentral.org/help/public-database):
```bash
vi external-service-externalname.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service-externalname
spec:
  type: ExternalName
  externalName: hh-pgsql-public.ebi.ac.uk
  ports:
  - port: 5432
```

Создаём сервис внешней базы данных:
```bash
k create -f external-service-externalname.yaml
```

Изучаем сервис внешней базы данных:
```bash
k get svc
k describe svc external-service-externalname
```

Удаляем модули для обновления переменных окружения:
```bash
k delete po --all
```

Проверяем переменные окружения модуля:
```bash
k exec $(k get po | grep appjs | head -n 1 | awk '{print $1}') env
```

Создаём модуль для тестирования подключения к внешней базе данных:
```bash
k run psql --image=crosbymichael/psql --generator=run/v1 --command -- sleep infinity
```

Тестируем подключения к внешней базе данных:
```bash
k exec $(k get po | grep psql | head -n 1 | awk '{print $1}') -it bash
psql -h external-service-externalname -d pfmegrnargs -U reader
password: -> NWDMCE5xdipIjRrp
\l
\q
exit
```

Разворачиваем аналогичную базу данных внутри кластера Kubernetes:
```bash
k run postgres --image=postgres:10.6 --generator=run/v1 --port=5432 --env="POSTGRES_USER=reader" --env="POSTGRES_PASSWORD=NWDMCE5xdipIjRrp" --env="POSTGRES_DB=pfmegrnargs"
```

Изучаем модуль созданной базы данных:
```bash
k describe pod $(k get po | grep postgres | head -n 1 | awk '{print $1}')
```

Изменяем сервис для коннекта к созданной базе данных:
```bash
k edit svc external-service-externalname
```

```yaml
...
  selector:
    run: postgres
  type: ClusterIP
...
```

Изучаем изменённый сервис:
```bash
k get svc external-service-externalname
k describe pod $(k get po | grep postgres | head -n 1 | awk '{print $1}')
k describe svc external-service-externalname
```

Проверяем доступ к созданной базе данных:
```bash
k exec $(k get po | grep psql | head -n 1 | awk '{print $1}') -it bash
psql -h external-service-externalname -d pfmegrnargs -U reader
password: -> NWDMCE5xdipIjRrp
\l
\q
exit
```

Удаляем созданные контроллеры репликации:
```bash
k delete rc postgres
k delete rc psql
```

## Topic 3: Use NodePort

Создаём описание сервиса, опубликованного на порте узла:
```bash
vi appjs-svc-nodeport.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app: appjs
```

Создаём сервис, опубликованный на порте узла:
```bash
k create -f appjs-svc-nodeport.yaml
```

Изучаем созданный сервис:
```bash
k get svc appjs-nodeport
k describe svc appjs-nodeport
```

Добавляем правило в фаервол Google Cloud Platform для предоставления доступа к сервису:
```bash
gcloud compute firewall-rules create appjs-svc-rule --allow=tcp:30123
```

Определяем внешний адрес узла:
```bash
k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'
```

Проверяем доступность приложений на порте узла:
```bash
curl http://$(k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}' | awk '{print $1}'):30123
curl http://$(k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address }' | awk '{print $2}'):30123
```

Удаляем созданное правило в фаерволе Google Cloud Platform:
```bash
gcloud compute firewall-rules delete appjs-svc-rule
```

Создаём описание сервиса с внешней балансировкой нагрузки:
```bash
vi appjs-svc-loadbalancer.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: appjs
```

Создаём сервис с внешней балансировкой нагрузки:
```bash
k create -f appjs-svc-loadbalancer.yaml
```

Изучаем созданный сервис:
```bash
k get svc appjs-loadbalancer
k describe svc appjs-loadbalancer
```

Проверяем доступность приложения на внешней балансировке (также можно открыть в браузере):
```bash
curl http://$(k get svc appjs-loadbalancer | grep appjs | awk '{print $4}')
```

Создаём описание сервиса с внешней балансировкой нагрузки и политикой вызова локальных модулей:
```bash
vi appjs-svc-loadbalancer-local.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appjs-loadbalancer-local
spec:
  externalTrafficPolicy: Local
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: appjs
```

Создаём сервис с внешней балансировкой нагрузки и политикой вызова локальных модулей:
```bash
k create -f appjs-svc-loadbalancer-local.yaml
```

Изучаем созданный сервис:
```bash
k get svc appjs-loadbalancer-local
k describe svc appjs-loadbalancer-local
```

Проверяем доступность приложения на внешней балансировке с политикой вызова локальных модулей:
```bash
curl http://$(k get svc appjs-loadbalancer-local | grep appjs | awk '{print $4}')
```
