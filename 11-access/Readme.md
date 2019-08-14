# Lab 11: Access

## Topic 1: Use ServiceAccounts

Получение скписка служебных записей:
```bash
k get sa
```

Создание служебной записи:
```bash
k create serviceaccount foo
```

Вывод описание созданной служебной записи:
```bash
k describe sa foo
```

Вывод описание созданной служебной записи:
```bash
k describe secret $(k get secrets | grep foo | awk '{print $1}')
```

Разрешить модулям, использующим учётную запись, монтировать только те секреты, которые указаны в служебной записи:
```bash
k annotate sa foo kubernetes.io/enforce-mountable-secrets="true"
```

Создадим описание служебной записи для выгрузки образов:
```bash
vi sa-image-pull-secrets.yaml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
imagePullSecrets:
- name: my-dockerhub-secret
```

Создадим служебную запись для выгрузки образов:
```bash
k create -f sa-image-pull-secrets.yaml
```

Вывод описания служебной записи для выгрузки образов:
```bash
k describe sa my-service-account
```

Создадим описание модуля, использующего нестандартную служебную учётную запись:
```bash
vi curl-custom-sa.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-custom-sa
spec:
  serviceAccountName: foo
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```

Создадим модуль, использующий нестандартную служебную учётную запись:
```bash
k create -f curl-custom-sa.yaml
```

Изучим созданный модуль:
```bash
k describe po curl-custom-sa
```

Проверим, что в модуль смонтирован токен, принадлежащий служебной учётной записи:
```bash
k exec -it curl-custom-sa -c main cat /var/run/secrets/kubernetes.io/serviceaccount/token
k describe secret $(k get secrets | grep foo | awk '{print $1}')
```

Проверим, что смонтированный токен даёт возможность взаимодействовать с API Kubernetes:
```bash
k exec -it curl-custom-sa -c main curl localhost:8001/api/v1/pods
```

## Topic 2: Use Roles and RoleBindings

Удалим привязку к кластерной роли, созданную ранее:
```bash
k delete clusterrolebinding permissive-binding
```

Создадим несколько пространств имён и запустим в них тестовые модули:
```bash
k create ns foo
k run test --image=luksa/kubectl-proxy -n foo
k create ns bar
k run test --image=luksa/kubectl-proxy -n bar
```

Проверим готовность созданных модулей:
```bash
k get po -n foo
k get po -n bar
```

Откройте 2 терминала и перейдите в интерактивный режим каждого из модулей:
```bash
k exec -it $(k get po -n foo | grep test | awk '{print $1}') -n foo sh
k exec -it $(k get po -n bar | grep test | awk '{print $1}') -n bar sh
```

Проверим, что включено управление ролевым доступом:
```bash
curl localhost:8001/api/v1/namespaces/foo/services
curl localhost:8001/api/v1/namespaces/bar/services
```

Создадим описание роли, разрешающей чтение и получение списка ресурсов типа "служба":
```bash
vi service-reader.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]
  resources: ["services"]
```

Создадим в пространстве имён foo роль, разрешающую чтение и получение списка ресурсов типа "служба":
```bash
k create -f service-reader.yaml -n foo
(если не работает - k create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=your.email@address.com)
```

Изучим созданную роль:
```bash
k describe role service-reader -n foo
```

Создадим в пространстве имён bar роль, разрешающую чтение и получение списка ресурсов типа "служба":
```bash
k create role service-reader --verb=get --verb=list --resource=services -n bar
```

Создадим в пространстве имён foo привязку роли к аккаунту по умолчанию, разрешающую чтение и получение списка ресурсов типа "служба":
```bash
k create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo
```

Изучим созданную привязку роли:
```bash
k get rolebinding test -n foo -o yaml
```

Проверим, что в пространстве имён foo получение списка служб стало доступным для аккаунта по умолчанию:
```bash
k exec -it $(k get po -n foo | grep test | awk '{print $1}') -n foo sh
curl localhost:8001/api/v1/namespaces/foo/services
```

Добавим в созданную привязку роли разрешение чтения и получения списка ресурсов типа "служба" в пространстве имён bar:
```bash
k edit rolebinding test -n foo
```

```yaml
...
subjects:
- kind: ServiceAccount
  name: default
  namespace: bar
```

Во втором терминале проверим, что чтение и получение списка ресурсов типа "служба" в пространстве имён foo доступно из под учётной записи по умолчанию пространства имён bar:
```bash
curl localhost:8001/api/v1/namespaces/foo/services
```

## Topic 2: Use ClusterRoles and ClusterRoleBindings

Создадим роль уровня кластера для получения и вывода списком постоянных томов:
```bash
k create clusterrole pv-reader --verb=get,list --resource=persistentvolumes
```

Изучим созданную роль уровня кластера:
```bash
k get clusterrole pv-reader -o yaml
```

В первом терминале проверим, что чтение и получение списка ресурсов типа "постоянный том" в пространстве имён foo не доступно из под учётной записи по умолчанию:
```bash
curl localhost:8001/api/v1/persistentvolumes
```

Создадим в пространстве имён foo привязку роли к аккаунту по умолчанию, разрешающую чтение и получение списка ресурсов типа "постоянный том", используя для привязки кластерную роль:
```bash
k create rolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default -n foo
```

В первом терминале проверим, что чтение и получение списка ресурсов типа "постоянный том" не доступно из под учётной записи по умолчанию в пространстве имён foo:
```bash
curl localhost:8001/api/v1/persistentvolumes
```

Изучим созданную привязку роли:
```bash
k get rolebindings pv-test -n foo -o yaml
```

Удалим созданную привязку роли:
```bash
k delete rolebinding pv-test -n foo
```

Создадим привязку кластерной роли к аккаунту по умолчанию в пространстве имён foo, разрешающую чтение и получение списка ресурсов типа "постоянный том" во всем кластере:
```bash
k create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default
```

Изучим созданную привязку кластерной роли:
```bash
k get clusterrolebindings pv-test -o yaml
```

В первом терминале проверим, что чтение и получение списка ресурсов типа "постоянный том" в пространстве имён foo доступно из под учётной записи по умолчанию:
```bash
curl localhost:8001/api/v1/persistentvolumes
```

Изучим системную кластерную роль, дающую минимальный доступ к APi Kubernetes, создаваемую по умолчанию:
```bash
k get clusterrole system:discovery -o yaml
```

Изучим пирвязку описанной выше кластерных ролей, создаваемую по умолчанию для всех пользователей:
```bash
k get clusterrolebinding system:discovery -o yaml
```

Продемонстрируем работу данной привязки кластерной роли, вызвав API Kubernetes, на которое распространяется данная привязка, неаутентифицированным пользователем:
```bash
(опционально minikube start)
curl https://$(minikube ip):8443/healthz -k
(опционально minikube stop)
```

Изучим системную кластерную роль, создаваемую по умолчанию и дающую возможность просматривать ресурсы пространств имён:
```bash
k get clusterrole view -o yaml
```

В первом терминале проверим, что просмотр ресурсов типа "модуль" на уровне кластера и в пространстве имён foo не доступен из под учётной записи пространства имён foo по умолчанию:
```bash
curl localhost:8001/api/v1/pods
curl localhost:8001/api/v1/namespaces/foo/pods
```

Создадим привязку кластерной роли к аккаунту по умолчанию в пространстве имён foo, разрешающую просмотр ресурсов типа "модуль" во всем кластере:
```bash
k create clusterrolebinding view-test --clusterrole=view --serviceaccount=foo:default
```

Изучим созданную привязку кластерной роли:
```bash
k get clusterrolebinding view-test -o yaml
```

В первом терминале проверим, что просмотр ресурсов типа "модуль" из под учётной записи пространства имён foo по умолчанию доступен в пространствах имён foo и bar, а также на уровне кластера:
```bash
curl localhost:8001/api/v1/namespaces/foo/pods
curl localhost:8001/api/v1/namespaces/bar/pods
curl localhost:8001/api/v1/pods
```

Удалим созданную привязку кластерной роли к аккаунту по умолчанию в пространстве имён foo:
```bash
k delete clusterrolebinding view-test
```

Создадим аналогичную простую привязку аккаунту по умолчанию в пространстве имён foo к той же кластерной роли:
```bash
k create rolebinding view-test --clusterrole=view --serviceaccount=foo:default -n foo
```

Изучим созданную привязку роли:
```bash
k get rolebinding view-test -o yaml -n foo
```

В первом терминале проверим, что просмотр ресурсов типа "модуль" из под учётной записи пространства имён foo по умолчанию доступен в пространстве имён foo, но не доступен в пространстве имён bar и на уровне кластера:
```bash
curl localhost:8001/api/v1/namespaces/foo/pods
curl localhost:8001/api/v1/namespaces/bar/pods
curl localhost:8001/api/v1/pods
```

Изучим привязки кластерных ролей, создаваемые по умолчанию:
```bash
k get clusterrolebindings
```

Изучим кластерные роли, создаваемые по умолчанию:
```bash
k get clusterroles
```