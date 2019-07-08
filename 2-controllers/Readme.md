# Lab 2: Controllers
---

## Topic 1: Use ReplicationControllers

Создаём описание контроллера репликации:
```bash
vi appjs-rc.yaml
```

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: appjs
spec:
  replicas: 3
  selector:
    app: appjs
  template:
    metadata:
      labels:
        app: appjs
    spec:
      containers:
      - image: jenoovchi/appjs
        name: appjs
        ports:
        - containerPort: 8080
```

Создаём контроллер репликации:
```bash
k create -f appjs-rc.yaml
```

Изучаем созданный контроллер репликации:
```bash
k get rc
k describe rc appjs
```

Проверяем восстановление модулей после сбоев:
```bash
k get pods
k delete pod $(k get pods | grep appjs | head -n 1 | awk '{print $1}')
k get pods
```

Проверяем восстановление модулей после сбоев узлов:
```bash
gcloud compute ssh $(k get node | grep jeno | head -n 1 | awk '{print $1}')
sudo ifconfig eth0 down
k get node
k get pods
```

Восстанавливаем работоспособность узла:
```bash
gcloud compute instances reset $(k get node | grep NotReady | awk '{print $1}')
```

Демонстрируем принципы работы контроллера репликации при использовании меток:
```bash
k label pod $(k get pods | grep appjs | head -n 1 | awk '{print $1}') type=special
k get pods --show-labels
k label pod $(k get pods | grep appjs | head -n 1 | awk '{print $1}') app=foo --overwrite
k get pods -L app
k edit rc appjs
#add another label to template
k get pods
```

Демонстрируем работу контроллера репликации при изменении количества реплик:
```bash
k edit rc appjs
#change replicas count to 10
k get rc
k scale rc appjs --replicas=3
k get pods
```

Удаляем контроллер репликации без удаления управляемых модулей:
```bash
k delete rc appjs --cascade=false
k get pods
```

## Topic 2: Use ReplicaSets

Создаём описание набора реплик:
```bash
vi appjs-replicaset.yaml
```

```yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: appjs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: appjs
  template:
    metadata:
      labels:
        app: appjs
    spec:
      containers:
      - image: jenoovchi/appjs
        name: appjs
        ports:
        - containerPort: 8080
```

Создаём набор реплик:
```bash
k create -f appjs-replicaset.yaml
```

Изучаем созданный набор реплик:
```bash
k get rs
k describe rs
```

Удаляем набор реплик без удаления управляемых модулей:
```bash
k delete rs appjs --cascade=false
```

Создаём описание набора реплик с селектором на основе выражения:
```bash
vi appjs-replicaset-matchexpressions.yaml
```

```yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: appjs
spec:
  replicas: 3
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
          - appjs
  template:
    metadata:
      labels:
        app: appjs
    spec:
      containers:
      - image: jenoovchi/appjs
        name: appjs
        ports:
        - containerPort: 8080
```

Создаём набор реплик с селектором на основе выражения:
```bash
k create -f appjs-replicaset-matchexpressions.yaml
```

Изучаем созданный набор реплик:
```bash
k get pods
k get rs appjs
k describe rs appjs
```

Удаляем созданный набор реплик:
```bash
k delete rs appjs
```

## Topic 3: Use DemonSets

Создаём описание набора реплик с селектором на основе выражения:
```bash
vi ssd-monitor-demonset.yaml
```

```yaml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - image: luksa/ssd-monitor
        name: main
```

Создаём набор демонов:
```bash
k create -f ssd-monitor-demonset.yaml
```

Изучаем созданный набор демонов:
```bash
k get ds
k get po
k get node
```

Добавляем метку набора демонов одному из узлов:
```bash
k label node $(k get node | grep jeno | head -n 1 | awk '{print $1}') disk=ssd
k get po
```

Меняем созданную метку на другую:
```bash
k label node $(k get node | grep jeno | head -n 1 | awk '{print $1}') disk=hdd --overwrite
k get po
```

Удаляем набор демонов:
```bash
k delete ds ssd-monitor
```

## Topic 4: Use Jobs

Создаём описание задачи:
```bash
vi job.yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - image: luksa/batch-job
        name: main
```

Создаём задачу:
```bash
k create -f job.yaml
```

Изучаем созданную задачу:
```bash
k get jobs
k describe jobs
k get po
k get po -a
k logs $(k get po -a | grep job | head -n 1 | awk '{print $1}')
k get job
```

Создаём описание задачи с множественным запуском:
```bash
vi multi-completion-batch-job.yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - image: luksa/batch-job
        name: main
```

Создаём задачу с множественным запуском:
```bash
k create -f multi-completion-batch-job.yaml
```

Изучаем созданную задачу:
```bash
k get jobs
k get po
k get jobs
```

Создаём описание задачи с множественным запуском и параллельным выполнением:
```bash
vi multi-completion-parallel-batch-job.yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-parallel-batch-job
spec:
  completions: 5
  parallelism: 2
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - image: luksa/batch-job
        name: main
```

Создаём задачу с множественным запуском и параллельным выполнением:
```bash
k create -f multi-completion-parallel-batch-job.yaml
```

Изучаем созданную задачу:
```bash
k get jobs
k get po
```

Увеличиваем количество параллельно выполняющихся реплик созданной задачи:
```bash
k scale job multi-completion-parallel-batch-job --replicas 3
k get po
```

Создаём описание задачи ограничением на длительность выполнения и количество неудачных запусков:
```bash
vi multi-completion-backoff-parallel-batch-job.yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-backoff-batch-job
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 3
  activeDeadlineSeconds: 20
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - image: luksa/batch-job
        name: main
```

Создаём задачу ограничением на длительность выполнения и количество неудачных запусков:
```bash
k create -f multi-completion-backoff-parallel-batch-job.yaml
```

Изучаем созданную задачу:
```bash
k get jobs
k get po
k get jobs
```

## Topic 5: Use CronJobs

Создаём описание задачи, запускаемой по расписанию:
```bash
vi cronjob.yaml
```

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
            restartPolicy: OnFailure
            containers:
            - image: luksa/batch-job
              name: main
```

Создаём задачу, запускаемую по расписанию:
```bash
k create -f cronjob.yaml
```

Проверяем созданную задачу:
```bash
k get jobs
k get po
```

Создаём описание задачи, запускаемой по расписанию с ограничением по точности запуска:
```bash
vi cronjob-with-deadline.yaml
```

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes-with-deadline
spec:
  schedule: "0,15,30,45 * * * *"
  startingDeadlineSeconds: 15
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
            restartPolicy: OnFailure
            containers:
            - image: luksa/batch-job
              name: main
```

Создаём задачу, запускаемую по расписанию с ограничением по точности запуска:
```bash
k create -f cronjob-with-deadline.yaml
```

Проверяем созданную задачу:
```bash
k get jobs
k get po
```

Удаляем созданные задачи:
```bash
k delete cronjob $(k get cronjob | grep job | awk '{print $1}')
```