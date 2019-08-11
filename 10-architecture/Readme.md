# Lab 10: Architecture

## Topic 1: etcd

Проверка статуса компонентов плоскости управления:
```bash
k get componentstatuses
```

Вывод компонентов плоскости управления Kubernetes, работающих как модули:
```bash
k get po -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system
```

Запускаем minikube (опционально) для демонстрации хранения конфигураций в Kubernetes:
```bash
minikube start
```

Создаём модуль для демонстрационных целей:
```bash
k run appjs --image=jenoovchi/appjs --port=8080 --generator=run/v1
```

Проверим, что созданный модуль запущен:
```bash
k get po
```

Переходим в контейнер etcd для изучения содержимого хранилища конфигураций Kubernetes:
```bash
k exec $(k get po -n kube-system | grep etcd | awk '{print $1}') -n kube-system -it sh
```

Конфигурируем переменные etcdctl:
```bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=/var/lib/minikube/certs/etcd/ca.crt
export ETCDCTL_CERT=/var/lib/minikube/certs/etcd/server.crt
export ETCDCTL_KEY=/var/lib/minikube/certs/etcd/server.key
```

Демонстрация содержимого хранилища конфигураций Kubernetes:
```bash
etcdctl get /registry --prefix --keys-only
```

Демонстрация списка пространств имён, в которых хранятся модули Kubernetes:
```bash
etcdctl get /registry/pods --prefix --keys-only
```

Демонстрация списка конфигураций модулей Kubernetes в пространстве имён default:
```bash
etcdctl get /registry/pods/default --prefix --keys-only
```

Демонстрация описания модуля appjs, развёрнутого в пространстве имён default:
```bash
etcdctl get $(etcdctl get /registry/pods/default --prefix --keys-only | grep appjs)
```

## Topic 2: API Server

Наблюдение за созданием и удалением модуля со стороны сервера API:
```bash
k delete po $(k get po | grep appjs | awk '{print $1}') | k get pods --watch
```

Наблюдение за созданием и удалением модуля со стороны сервера API в виде yaml описания:
```bash
k delete po $(k get po | grep appjs | awk '{print $1}') | k get pods -o yaml --watch
```

## Topic 3: Addons

Просмотр развёрнутых надстроек:
```bash
k get deploy -n kube-system
```

## Topic 4: Events

Просмотр событий создаваемого и удаляемого модуля:
```bash
k delete po $(k get po | grep appjs | awk '{print $1}') | k get events --watch
```

## Topic 4: Pods

Создание модуля Nginx:
```bash
k run nginx --image=nginx
```

Соединение с узлом для изучения работы модулей:
```bash
gcloud compute ssh $(k describe pod $(k get po | grep nginx | head -n 1 | awk '{print $1}') | grep Node | head -n 1 | awk '{print $2}' | sed 's/\/.*//')
or
minikube ssh
```

Находим контейнеры, ассоциированные с Nginx:
```bash
docker ps | grep nginx
```

## Topic 4: Leader ellection

Демонстрация механизма выбора лидера при создании конечных точек:
```bash
k get endpoints kube-scheduler -n kube-system -o yaml
```

Удаление всех модулей:
```bash
k delete rc --all
k delete deployment --all
```