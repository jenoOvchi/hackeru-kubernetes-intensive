# Lab 10: Resources

## Topic 1: Use Requests and Limits

Создадим описание модуля с запросом ресурсов:
```bash
vi requests-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: requests-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      requests:
        cpu: 200m
        memory: 10Mi
```

Создадим модуль с запросом ресурсов:
```bash
k create -f requests-pod.yaml
```

Изучим созданный модуль:
```bash
k get po requests-pod
k describe po requests-pod
```

Изучим потребление CPU и RAM в созданном модуле:
```bash
k exec -it requests-pod top
```

Изучим доступные ресурсы на узлах:
```bash
k describe nodes
```

Создадим модуль, не помещающийся ни на одном узле:
```bash
k run request-pod-3 --image=busybox --restart Never --requests='cpu=2,memory=20Mi' -- dd if=/dev/zero of /dev/null
```

Изучим созданный модуль:
```bash
k get po requests-pod-3
k describe po requests-pod-3
```

Создадим описание модуля с ограничением выделения ресурсов:
```bash
vi limited-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      limits:
        cpu: 1
        memory: 20Mi
```

Создадим модуль с ограничением выделения ресурсов:
```bash
k create -f limited-pod.yaml
```

Изучим созданный модуль:
```bash
k get po limited-pod
k describe po limited-pod
```

Изучим потребление CPU и RAM в созданном модуле:
```bash
k exec -it limited-pod top
```

Создадим развёртывание с запросом и лимитом ресурсов:
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
        resources:
          requests:
            cpu: 200m
            memory: 10Mi
          limits:
            cpu: 1
            memory: 20Mi
```

Создаём средство автоматического масштабирования для развёртывания:
```bash
k autoscale deployment appjs --cpu-percent=30 --min=1 --max=5
```

Генерируем нагрузку:
```bash
k run -it --rm --restart=Never loadgenerator --image=busybox -- sh -c "while true; do wget -O - -q http://appjs.default; done"
```

Наблюдаем за изменением количества экземпляров:
```bash
k get po -w
```

Создаём описание ограничения на ресурсы для пространство имён default:
```bash
vi limits.yaml
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example
spec:
  limits:
  - type: Pod
    min:
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 10Mi
    default:
      cpu: 200m
      memory: 100Mi
    min:
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
    maxLimitRequestRatio:
      cpu: 4
      memory: 10
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 10Gi
```

Создаём ограничение на ресурсы для пространство имён default:
```bash
k create -f limits.yaml
```

Создаём описание лимитов на ресурсы пространства имён default:
```bash
vi cpu-and-mem.yaml
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-and-mem
spec:
  hard:
    requests.cpu: 400m
    requests.memory: 200Mi
    limits.cpu: 600m
    limits.memory: 500Mi
```

Создаём описание квоты на ресурсы:
```bash
vi objects.yaml
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: objects
spec:
  hard:
    pods: 10
    replicationcontrollers: 5
    secrets: 10
    configmaps: 10
    persistentvolumeclaims: 5
    services: 5
    services.loadbalancers: 1
    services.nodeports: 2
    ssd.storageclass.storage.k8s.io/persistentvolumeclaims: 2
```

Создаём квоту на ресурсы:
```bash
k create -f objects.yaml
```