# Configure your Desktop
---

## Install VisualStudioCode (опционально)
Скачиваем: https://code.visualstudio.com
1. Запускаем файл "Docker for Windows Installer.exe"
2. Устанавливаем VisualStudioCode
3. Запускаем ярлык VisualStudioCode на рабочем столе

## Install Docker

### Windows

Скачиваем https://download.docker.com/win/stable/Docker%20for%20Windows%20Installer.exe
1. Запускаем файл "Docker for Windows Installer.exe"
2. Устанавливаем Docker
3. Запускаем ярлык Docker на рабочем столе

### OS X

Скачиваем https://download.docker.com/mac/stable/Docker.dmg
1. Запускаем файл Docker.dmg
2. Устанавливаем Docker
3. Запускаем ярлык Docker на панели быстрого запуска

## Test Docker and create test app

Запускаем PowerShell и проверяем работоспособность Docker:
```bash
docker run busybox echo "Hello World"
```
### (Опционально) На Windows
Устанавливаем Vim (либо везде по тексту меняем vi на notepad):
1. Скачиваем Vim: ftp://ftp.vim.org/pub/vim/pc/gvim74.exe
2. Устанавливаем Vim из скачанного файла
3. Создаём профиль PowerShell:
```bash
$profile
new-item -path $profile -itemtype file -force
notepad $profile
set-alias vim "C:/Program Files (x86)/Vim/vim74/.\vim.exe"
set-alias vi "C:/Program Files (x86)/Vim/vim74/.\vim.exe"
Save and exit
```
4. Перезапускаем PowerShell

Создаём тестовое приложение:
```bash
mkdir Kubernetes
cd Kubernetes
mkdir appjs
cd appjs
vi app.js
```

```javascript
const http = require('http');
const os = require('os');

console.log("Appjs server starting...");

var handler = function(request, response) {
   console.log("Received request from " + request.connection.remoteAddress);
   response.writeHead(200);
   response.end("You've hit " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

Создаём файл сборки образа Docker:
```bash
vi Dockerfile
```

```bash
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

Создаём образ приложения:
```bash
docker build -t appjs .
```

Проверяем созданные образ:
```bash
docker images
```

Запускаем созданное приложение:
```bash
docker run --name appjs-container -p 8080:8080 -d appjs
```

Проверяем созданное приложение:
```bash
docker ps
curl http://localhost:8080
docker inspect appjs-container
docker exec -it appjs-container bash
ls /
ps aux
exit
ps aux | grep app.js
```

Останавливаем запущенное приложение:
```bash
docker stop appjs-container
```

Загружаем созданное приложение на Docker Hub:
http://hub.docker.com -> Registration => login/password
```bash
docker tag appjs ${login}/appjs
docker login -> login/password
docker push ${login}/appjs
```

Запускаем созданное приложение с Docker Hub:
```bash
docker run -p 8080:8080 -d ${login}/appjs
curl http://localhost:8080
```

## Install Minikube

### Windows
1. Устанавливаем VirtualBox (https://download.virtualbox.org/virtualbox/6.0.8/VirtualBox-6.0.8-130520-Win.exe)
2. Устанавливаем Minikube (https://github.com/kubernetes/minikube/releases/latest/download/minikube-installer.exe)
3. Отключаем Hyper-V (если включен): https://yandex.ru/znatoki/question/computers/kak_otkliuchit_hyper_v_v_windows_10_0ee3d668/?utm_source=yandex&utm_medium=wizard#5fb4c48c-73a1-4057-9962-795a9b54735d (если Docker ругается - игнорируем)
4. Устанавливаем MobaXterm: https://download.mobatek.net/1112019010310554/MobaXterm_Portable_v11.1.zip


### OS X
1. Устанавливаем Homebrew (https://brew.sh/index_ru)
2. Устанавливаем VirtualBox (https://download.virtualbox.org/virtualbox/6.0.8/VirtualBox-6.0.8-130520-OSX.dmg)
3. Устанавливаем Minikube:
```bash
brew update
sysctl -a | grep machdep.cpu.features
brew cask install minikube
```

## Test Minikube

Запускаем Minikube:
```bash
minikube start
```

Запрашиваем информацию о состоянии Minikube:
```bash
kubectl cluster-info
kubectl get pod -n kube-system
```

## Configure Kubectl (for Windows)

Узнаём адрес виртуальной машины Minikube:
```bash
minikube status
```

Создаём SSH сессию в MobaXterm:
host: из команды minikube status
login: docker
password: tcuser

Устанавливаем Kubectl:
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/bin/kubectl
```

Создаём файл конфигурации для Kubectl:
```bash
mkdir .kube
cd .kube
cp /C/Users/{your-user}/.kube/config ./config
vi config
```
Замените в путях бэкслэш (\) на обычный (/) и удалите двоеточие (:).

Проверяем работу Kubectl:
```bash
kubectl cluster-info
kubectl get pod -n kube-system
```

## Test Minikube (2)

Запускаем тестовое приложение:
```bash
kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.10 --port=8080
```

Делаем приложение доступным извне:
```bash
kubectl expose deployment hello-minikube --type=NodePort
```

Получаем информацию о запущенном приложении:
```bash
kubectl get pod
minikube service hello-minikube --url
```

Тестируем запущенное приложение:
```bash
minikube service hello-minikube --url | xargs curl
```

Запускаем информационную панель Minikube:
```bash
minikube dashboard
```

Удаляем созданные ресурсы и останавливаем Minikube:
```bash
kubectl delete services hello-minikube
kubectl delete deployment hello-minikube
minikube stop
```

## Create Kubernetes Cluster on Google Cloud Platform

1. Открываем в браузере https://cloud.google.com/kubernetes-engine/docs/quickstart
2. Открываем URL "Kubernetes Engine page"
3. Подтверждаем пользовательское соглашение
4. Нажимаем "Создать"
5. Вводим "Название проекта" и нажимаем "Создать"
6. Нажимаем "Активировать пробный период"
7. Выбираем "Страна" и подтверждаем пользовательское соглашение
8. Заполняем информацию о карте. В пробный период (до 1 года) деньги сниматься не будут
9. Нажимаем "Включить оплату"
10. Нажимаем кнопку "..." и выбираем "Файлы для установки"
11. Нажимаем на ссылку "Cloud SDK"
12. Следуем инструкциям для вашей операционной системы
13. Запускаем в консоли gcloud init
14. Авторизуемся в аккаунте Google для использования SDK
15. Выбираем проект
17. Выбирае мрегион (50 для выбора ближайшей Финляндии)

Создаём псевдонимы:
```bash
echo "alias gcloud='/Users/jeno/Downloads/google-cloud-sdk/bin/gcloud'" >> ~/.bash_profile
echo "alias k=kubectl" >> ~/.bash_profile
source ~/.bash_profile
```

Устанавливаем kubectl:
```bash
gcloud components install kubectl
```

Устнавливаем средства дополнения команд:
```bash
brew install bash-completion
echo '[[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"' >> ~/.bash_profile
source ~/.bash_profile
source <(kubectl completion bash)
source <(kubectl completion bash | sed s/kubectl/k/g)
```

## Create Kubernetes Cluster

Создаём тестовый кластер:
```bash
gcloud container clusters create jenotest --num-nodes 3 --machine-type g1-small
```

Проверяем список узлов:
```bash
kubectl get nodes
kubectl describe node gke-jenotestnew-default-pool-b89f9404-f66k
```

Заходим на один из узлов по ssh:
```bash
gcloud compute ssh gke-jenotestnew-default-pool-b89f9404-f66k
Enter passphrase
ps aux
exit
```

## Install Dashboard
Создаём ресурсы информационной панели:
```bash
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

Запускаем прокси для доступа к информационной панели:
```bash
kubectl proxy
```

Получаем авторизационные данные:
```bash
kubectl create serviceaccount dashboard -n default
kubectl create clusterrolebinding dashboard-admin -n default --clusterrole=cluster-admin --serviceaccount=default:dashboard
kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

1. Открываем информационную панель в браузере: http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
2. Выбираем опцию  "Token" копируем её из терминала и вставляем в браузер
