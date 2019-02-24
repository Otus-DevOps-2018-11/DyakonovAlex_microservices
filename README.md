# DyakonovAlex_microservices
DyakonovAlex microservices repository

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
