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

Создаём ограничения на ресурсы для пространство имён default:
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