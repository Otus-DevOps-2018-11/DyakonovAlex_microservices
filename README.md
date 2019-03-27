# DyakonovAlex_microservices

DyakonovAlex microservices repository

## Homework 18 Мониторинг приложения и инфраструктуры

### Подготовка окружения

```bash
export GOOGLE_PROJECT=docker-232609

# Создать докер хост
docker-machine create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-1 \
    --google-zone europe-west1-b \
    docker-host

# Настроить докер клиент на удаленный докер демон
eval $(docker-machine env docker-host)

# Переключение на локальный докер
# eval $(docker-machine env --unset)

docker-machine ip docker-host #35.241.242.123

# docker-machine rm docker-host

```

### Мониторинг Docker контейнеров  

- Запуск мониторинга вынесен в отдельный compose-файл. В makefile внесены нужные изменения.

```bash
docker-compose up -d
docker-compose -f docker-compose-monitoring.yml up -d
```

- Добавлен запуск cAdvisor. Открыт порт для его веб-интерфейса. Проверено, что метрики собираются.

```bash
gcloud compute firewall-rules create cadvisor-default --allow tcp:8080

```

### Визуализация метрик  

- Добавлен запуск Grafana. Открыт порт.  

```bash
gcloud compute firewall-rules create grafana-default --allow tcp:3000

```

- Запущен новый сервис Grafana

```bash
docker-compose -f docker-compose-monitoring.yml up -d grafana

```

- Скачан и импортирован дашборд

### Сбор метрик приложения  

- Добвлена информация о post сервисе в конфигурацию Prometheus
- Пересобран образ Prometheus с обновленной конфигурацией

```bash
docker build -t $USER_NAME/prometheus .
```

- Пересоздана Docker инфраструктура мониторинга

```bash
docker-compose -f docker-compose-monitoring.yml down
docker-compose -f docker-compose-monitoring.yml up -d
```

- Создана панель для метрик приложения.  
- Использована для первого графика ```rate(ui_request_count[1m])```
- Использована функция ```rate(ui_request_count{http_status=~"^[45].*"}[1m])``` для второго графика
- Ознакомлен с гистограммами ```ui_request_response_time_bucket{path="/"}```
- Добавлена панель с перцентилем ```histogram_quantile(0.95, sum(rate(ui_request_response_time_bucket[5m])) by (le))```
- Панель экспортирована в файл.  

### Сбор метрик бизнес логики  

- Добавлена и экспортирован панель Business_Logic_Monitoring ```rate(post_count[1h])``` ```rate(comment_count[1h])```

### Алертинг  

- Добавлен alertmanager и конфиги для него.  
- Добавлен запуск alertmanager. Открыт порт.  

```bash
gcloud compute firewall-rules create alertmanager-default --allow tcp:9093

```

- Собран образ alertmanager

```bash
docker build -t $USER_NAME/alertmanager .
```

- Создан файл alerts.yml в директории prometheus
- Добавлена операция копирования данного файла в Dockerfile
- Добавлена информация о правилах, в конфиг Prometheus
- Пересобран образ Prometheus
- Пересоздана Docker инфраструктура мониторинга
- Проверена работа алерта
- Образы загружены в [docker registry](https://hub.docker.com/u/happydyakonov)

## Homework 17 Введение в мониторинг. Модели и принципы работы систем мониторинга  

### Подготовка окружения

- Создано правило фаервола для Prometheus и Puma, Docker хост в GCE и настроено локальное окружение на работу с ним

```bash
gcloud compute firewall-rules create prometheus-default --allow tcp:9090
gcloud compute firewall-rules create puma-default --allow tcp:9292

export GOOGLE_PROJECT=docker-232609

# create docker host
docker-machine create --driver google \
--google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
--google-machine-type n1-standard-1 \
--google-zone europe-west1-b \
docker-host

# configure local env
eval $(docker-machine env docker-host)

docker run --rm -p 9090:9090 -d --name prometheus  prom/prometheus
docker-machine ip docker-host #34.76.29.42
docker stop prometheus
```

- Запущен контейнер prometheus

```bash
docker-machine ssh docker-host
docker run --rm -p 9090:9090 -d --name prometheus  prom/prometheus:v2.1.0

```

- Переупорядочена структура директорий
- Создана директория monitoring, добавлен dockerfile для создания настроенного образа prometheus, собран образ, созданы образы приложений

```bash
export USER_NAME=happydyakonov
docker build -t $USER_NAME/prometheus .
for i in ui post-py comment; do cd src/$i; bash docker_build.sh; cd -; done
```

- Добавлена секция запуска prometheus в docker-compose.  
- Добавлена секция networks в определение сервиса prometheus в docker-compose
- Запущены сервисы с помощью docker-compose  
- Проверено, что указанные в конфигурации endpoints в состоянии UP  
- Протестировано реагирование графиков на отключение сервисов  
- Добавлен запуск node exporter в docker-compose для сбора информации о хосте  
- Проверен мониторинг
- Образы загружены в [docker registry](https://hub.docker.com/u/happydyakonov)  


## Homework 16 Устройство Gitlab CI. Построение процесса непрерывной поставки

### Создана виртуальная машина

```bash
docker-machine create --driver google \
--google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
--google-machine-type n1-standard-1 \
--google-zone europe-west1-b \
--google-disk-size 70 \
--google-project docker-232609 \
--google-tags http-server,https-server \
gitlab-host

eval $(docker-machine env gitlab-host)
```

### Подготавливаем окружение

```bash
docker-machine ssh gitlab-host
sudo mkdir -p /srv/gitlab/config /srv/gitlab/data /srv/gitlab/logs
cd /srv/gitlab
sudo touch docker-compose.yml
sudo nano docker-compose.yml
    web:
    image: 'gitlab/gitlab-ce:latest'
    restart: always
    hostname: 'gitlab.example.com'
    environment:
        GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://35.205.91.233'
    ports:
        - '80:80'
        - '443:443'
        - '2222:22'
    volumes:
        - '/srv/gitlab/config:/etc/gitlab'
        - '/srv/gitlab/logs:/var/log/gitlab'
        - '/srv/gitlab/data:/var/opt/gitlab'

sudo apt install docker-compose
sudo docker-compose up -d
```

- Создана группа, проект и загружено содержимое репозиторя microservices  
- Добавлен файл .gitlab-ci.yml  
- Токен m_8zx_2uMiM17sizxR5v
- Запущен и зарегистрирован gitlab-runner

```bash

sudo docker run -d --name gitlab-runner --restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:latest

sudo docker exec -it gitlab-runner gitlab-runner register --run-untagged --locked=false
```

- Паплайн запустился
- Добавлен исходный код reddit в репозиторий
- Изменено описание пайплайна в .gitlab-ci.yml
- Добавлен simpletest.rb в папку reddit
- Добавлен gem 'rack-test'
- Добавлено dev окружение, результат виден в Opeations → Environments
- Добавлены stage и production окружения
- Добавлена директива, которая не позволяет выкатить на staging и production код,не помеченный с помощью тэга в git (only: - /^\d+.\d+.\d+/)
- Добавлен job, который определяет динамическое окружение для каждой ветки в репозитории, кроме ветки master

## Homework 15

### Работа с сетью в docker

#### None network driver

- ```docker run -ti --rm --network none joffotron/docker-net-tools -c ifconfig```

#### Host network driver

- ```docker run -ti --rm --network host joffotron/docker-net-tools -c ifconfig```
- ```docker-machine ssh docker-host ifconfig```
- Результаты одинаковы, так как используется сеть хоста.
- Запущен несколько раз ```docker run --network host -d nginx```.
В ```docker ps``` видно, что остался запущен только один контейнер. Это происходит из-за того, что используется один интерфейс и порт уже занят.
- ```docker kill $(docker ps -q)```  
- ```docker-machine ssh docker-host 'sudo ln -s /var/run/docker/netns /var/run/netns'```
- ```docker-machine ssh docker-host 'sudo ip netns'```  
- Повторены запуски контейнеров с использованием драйверов ```none``` и ```host``` и просмотрено, как меняется список namespace-ов

#### Bridge network driver  

- ```docker network create reddit --driver bridge```
- Запущены контейнеры приложения
- Проверено, что приложение функционирует некорректно. Созданы новые контейнеры с присвоением сетевых псевдонимов  
- Приложение функционирует корректно

##### Запуск проекта в 2-х bridge сетях

- Созданы docker-сети
- Запущены контейнеры
- Убедились, что приложени работает некорректно, так как Docker при инициализации контейнера может подключить к нему только 1 сеть  
- Подключены дополнительные сети для контейнеров post и comment  
- Приложение работает корректно  
- Произведена установка пакета bridge-utils на docker-host
- Выполнена команда ```docker network ls``` и найдены ID сетей, созданных в рамках проекта
- Выполнено на docker-host ```ifconfig | grep br``` и найдены bridge-интерфейсы для каждой из сетей  
- Просмотрена информация о интерфейсе с помощью команды ```brctl show br-24373a954c6f```
- Выполнено ```sudo iptables -nL -t nat```
- Выполнено ```ps ax | grep docker-proxy```. Проверено, что docker-proxy слушает порт 9292  

### Использование docker-compose

- Установлен docker-compose ```pip install docker-compose```
- Добавлен файл docker-compose.yml
- Собраны образы и запущены контейнеры

```bash
export USERNAME=happydyakonov
docker-compose up -d
docker-compose ps
```

- Проверено, что приложение работает.

#### Задания для самостоятельной работы  

- Изменён ```docker-compose.yml``` под кейс с множеством сетей, сетевых алиасов
- Параметризированы с помощью переменных окружений:
  - порт публикации сервиса ui
  - версии сервисов
- Параметризованные параметры записаны в отдельный файл ```.env```  
- Проверено, что без использования команд ```source``` и ```export``` ```docker-compose``` подхватывает переменные из этого файла  
- Базовое имя проекта, по умолчанию, образуется на основе имени директории из которой производится запуск  
  Способы изменения:
  - запустить ```docker-compose up -d -p new_project_name```  
  - задать в переменной окружения ```COMPOSE_PROJECT_NAME```  

## Homework 14

### Сделано

- Скачан архив с исходниками
- Для каждого из трех компонентов создан Dockerfile
- Скачен последний образ MongoDB
- Собраны образы с нашими сервисами
- Создана bridge-сеть для контейнеров
- Запущены контейнеры в этой сети
- Добавлены сетевые алиасы контейнерам
- Улучшен образ сервиса ui
- Создан Docker volume и подключен к контейнеру с MongoDB
- Проверено, что пост сохраняется после перезапуска контейнеров


## Homework 13

### Сделано

- Создан новый проект docker
- Сконфигурирован gcloud для нового проекта docker
- Получен файл с аутентификационными данными gcloud auth application-default login
- Запущен докер хост в GCP
- Добавлен Dockerfile - текстовое описание нашего образа
- Добавлен mongod.conf - подготовленный конфиг для mongodb
- Добавлен db_config - содержит переменную окружения со ссылкой на mongodb
- Добавлен start.sh - скрипт запуска приложения
- Собран образ
- Запущен контейнер
- Настроен firewall
- Настроена работа с Docker hub
- Образ загружен на Docker hub


## Homework 12

- Настроена интеграция с travis-ci по аналогии с репозиторием infra
- Установлен docker
- Запущен docker run hello-world
- Запущен docker run -it ubuntu:16.04 /bin/bash 
- Найден старый контейнер docker ps -a
- Подключился к старому контейнеру docker start 26b4b001a56d && docker attach 26b4b001a56d
- Создан образ на основе этого контейнера docker commit 26b4b001a56d dyakonovalex/ubuntu-tmp-file
- Сохранен вывод команды docker images в файл docker-monolith/docker-1.log
