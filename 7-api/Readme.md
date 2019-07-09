# Lab 7:API

## Topic 1: Use Downward API

Создадим описание модуля с передачей информации о модуле в переменные окружения:
```bash
vi downward-api-env.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:
          resource: requests.cpu
          divisor: 1m
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```

Создаём модуль с передачей информации о модуле в переменные окружения:
```bash
k create -f downward-api-env.yaml
```

Изучаем созданный модуль:
```bash
k get po 
k describe po downward
```

Тестируем передачу информации о модуле в переменные окружения:
```bash
k exec downward env
k exec downward env | grep NODE
```

Создадим описание модуля с передачей информации о модуле во внутренний том контейнера:
```bash
vi downward-api-volume.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-volume
  labels:
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      - path: "containerMemoryLimitByte"
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1
```

Создаём модуль с передачей информации о модуле во внутренний том контейнера:
```bash
k create -f downward-api-volume.yaml
```

Изучаем созданный модуль:
```bash
k get po 
k describe po downward-volume
```

Тестируем передачу информации о модуле во внутренний том контейнера:
```bash
k exec downward-volume -- ls -lL /etc/downward
k exec downward-volume cat /etc/downward/annotations
```


Удаляем созданные модули:
```bash
k delete po --all
```

## Topic 2: Use Kubernetes API

Выведем информацию о кластере:
```bash
k cluster-info
```

Попробуем получить информацию с помощью API:
```bash
curl https://35.228.37.100 -k
```

Включим прокси и попробуем получить информацию с помощью проксированного API:
```bash
k proxy
curl localhost:8001
```

Создадим задачу о которой попробуем получить информацию:
```bash
k create -f job.yaml #If not exist
curl localhost:8001/apis/batch
curl localhost:8001/apis/batch/v1
curl localhost:8001/apis/batch/v1/jobs
curl localhost:8001/apis/batch/v1/namespaces/default/jobs/batch-job
```

В тестовых целях создадим связь ролей группы служебных аккаунтов с ролью администратора кластера (для демонстрации получения информации о кластере из модуля):
```bash
k create clusterrolebinding permissive-binding --clusterrole=cluster-admin --group=system:serviceaccounts
```

Создадим описание модуля с утилитой curl:
```bash
vi curl.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - image: tutum/curl
    name: main
    command: ["sleep", "9999999"]
```

Создадим модуль с утилитой curl:
```bash
k create -f curl.yaml
```

Изучаем созданный модуль:
```bash
k get po
k describe po curl
```

Подключимся к модулю curl и изучим доступной API:
```bash
k exec -it curl bash
> env | grep KUBERNETES_SERVICE
> curl https://kubernetes
> ls /var/run/secrets/kubernetes.io/serviceaccount
> curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes
> export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
> curl https://kubernetes
> TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
> NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
> curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/$NS/pods
> exit
```

Создадим описание модуля curl с контейнером-послом:
```bash
vi curl-with-ambassador.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-with-ambassador
spec:
  containers:
  - image: tutum/curl
    name: main
    command: ["sleep", "9999999"]
  - image: luksa/kubectl-proxy:1.6.2
    name: ambassador
```

Создадим модулm curl с контейнером-послом:
```bash
k create -f curl-with-ambassador.yaml
```

Изучаем созданный модуль:
```bash
k get po
k describe po curl-with-ambassador
```

Подключимся к модулю curl и изучим доступной API:
```bash
k exec -it curl-with-ambassador -c main bash
> curl localhost:8001
> NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
> curl localhost:8001/api/v1/namespaces/$NS/pods
> exit
```

Ещё раз выведем информацию о кластере и откроем API в браузере:
```bash
k cluster-info
https://35.228.37.100/openapi/v2
```

Удалим все созданные сущности:
```bash
k delete all --all
k delete ingresses --all
k delete service --all
gcloud compute addresses delete static-address
```