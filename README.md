# Docker_training
# Docker in practioce - ivan Mill, Eidar Hobson

## Часть 1. Основы Docker
Docker -  открытая платформа для разработки, доставки и эксплуатации прожений.
Позволяет в своем ядре запускать практически любое приложение, безопасно изолированное в контейнере.

### 1.2.2 Dockerfile
Пример докер файла для сборки образа todoapps
```
FROM node  # базовый образ
LABEL maintainer ian.mill@gmail.com # объявляем автора
RUN git clpne -q https://github.com/docker-in-practice/todo.git # клонирует код приложения 
WORKDIR todo # перемещаем в новый католог 
RUN npm install > /dev/null # Запкуск установки менеджера пакетов
EXPOSE 8000 # Указываем, что контенейр по умолчанию будет слушать этот порт
CMD ["npm", "start"] # Выполнение команды при запуске
```
Определяем образ, в примере используем образ Node.js.
1.2.3 Собираем образ Docker

Команда сборки образа 
```
docker build . 
```
`Сборка образа из текущей папки без аргументов`

смена имени контейнера
```
docker tag 66c76cea05bb todoapp
```
`обращение к докеру через подкомандку tag, далее вводим идентификатор созданого образа и имя тега.`

### 1.2.5 Слои Docker
Docker использует слои для хранения информации о контейнере. Каждый слой содержит результат оперции, выполненной над предыдущем слоем. Используя кеширование, чтобы повторно не выполнять генерацию слоя.

# Часть 2. Постигаем Docker.

## 2.1 Архитектура Docker

Хост-компьютер включает в себя докер-клиент и демон докер. Может обращаться к частному реестру докер либо общедоступному DockerHub. Все запросы происходят через протокол HTTP используя стандартные типы, такие как GET, POST DELETE.

Докер-клиент чтобы получить информацию или дать инструкции обращается к серверу докер-демону, который возращает ответы от клиента по HTTP. Он принимает запросы от любого авторизированного клиента.

Частный реест Docker - сервис который хранит образы Докер. 

## 2.2 Демон Docker

Демон - это процесс который выполняется в фоновом режиме, а не под  непосредственным контролем пользователя. Сервер - процесс который принимает запросы от клиента и осуществляет действие необходимые для выполнения запросов. Демон может являтся Сервером.

Сервер Docker можно сделать общедоступным. с открытым TCP-адресом.

По умолчанию демон-докер обращается через сокет /var/run/docker.sock к клиенту и обратно.
Открыв общедоступный сокет к нему может обращаться любой кто может подключиться к хосту (что очень не безопасно) Например Jenkins Сервер.

Чтобы открыть доступ надо...
```
 systemctl stop docker
 # остановить сервис докер
 ps - ef | grep -E 'docker(d| -d| daemon)\b' | grep -v | grep
 # Проверка, если сработало не будет вывода.
 sudo docker daemon -H tcp://0.0.0.0:2375
 # задаем демону в качествер хост-сервера порт 2375 на локалхосте.
 docker -H tcp://localhsot:2375 <subcommand>
 # подключение снаружи к докеру.
```
Можно экспортировать переменную среды DOCKER_HOST (*ЭТО НЕ СРАБОТАЕТ ЕСЛИ ДЛЯ ЗАПУСКА DOCKER НУЖЕН SUDO*)
```
export DOCKER_HOST=tcp://<your host`s ip>:2375
docker <subcommnd>
```
ВНИМАНИЕ! Если указать 0.0.0.0 это даст доступ пользователям на всх сетевых интерфейсах (как публичных так и частных), что считается не безопасным.

Альтернатива это монтирование сокета Docker в качестве тома с использованием утилиты socat для переброса траффика из внешнего порта 
```
docker run -p 2375:2375 -v/var/run/docker.sock:/var/run/docker.sock sequenceid/socat
```

Запуск контейнера в фоновом режиме как служба, используется флаг -d (detach)
```
docker run -d -i -p 1234:1234 --name daemon ubuntu:14.04 nc -l 1234
```
Флаг -d в качестве демона. Флаг -i дает взаимодействие с нашим сеансом Telnet. -p объявляет порт 1234 (внутренний:внешний). --name присвоение имени контейнеру.
в конце запускаем простой прослушивающий эхо-сервер на порту 1234 с помощью netcat (nc)

Флаг docker run--restart= (имеет следующие варианты)
* no - Не перезапускать при выходе контейнера
* always - Всегда перезапускать при выходе контейнера
* unless-stopped - Всегда перезагружать, но помнить о явной остановке.
* on-failure[:max-retry] - Перезапускать только в случае сбоя

`docker run -d --restart=always ubuntu echo done`
Запуская контейнер в качестве демона (-d) и всегда перезапуская его по завершению (--restart=always) она выдает команду (echo), которая завершается выходя из контейнера.

```
dockers ps # выводит список запущенных контейнеров
```
* Когда был создан (CREATED)
* Текущее состояние контейнера (STATUS)
* Код выхода предыдущего запуска контейнера. 0 == успех
* Название контейнера

Unless-stopped похож на always - оба прекратят перезапуск, если выполнить docker stop, но unless-stopped  запомнит  состояние при повторно запуске (возможно перезагрузке компьютера), в то время как always просто вернет контейнер.
On-failure перезапускается только в случае когда контейнер вернет ненулевой код завершения (что значит сбой) из своего основного процесса:
--restart=on-failure:10 устанавливаиет кол-во попыток перезапуска и выхода когда оно будет превышено.

Docker 
переопределение места хранения данных -g
Будет создан новый набор папок и файлов необходимых docker`у.
```
dockerd -g /home/sokceruser/mydocker
```
### Метод 4. Использоваине socat для мониторинга трафика Docker API.

Для диагностики проблем полезно просмотреть поток данных поступающих к демону Docker и от него.

```
socat -v UNIX-LISTEN:/tmp/dockerapi.sock,fork UNIX-CONNECT:/var/run/docker.sock &
```
-v делает читаемым с указанием потока данных.
Часть UNIX-LISTEN говорит socat  прослушивать сокет unix. fork гарантирует, что socat не завершит работу после первого заппроса, а UNIX-CONNECT сообщает socat  подключиться к UNIX сокету docker. & указывает что команда выполняется в фоновом режиме.

Socat - он может обрабатывать множетсво различных протоколов. Так же можем заставить показывать внешний порт с помощью `TCP-LISTEN: 2375` так же работает для прослушивания докер, без остановки контейнеров.

docker attach  для подключения к терминалу, который запущен с помощью docker run, для сотрудничества напрямую.

### Докер в браузере.

```
git clone https://github/aidanhs/Docker-Terminal.git
cd Docker-Terminal
# Запуск скрипта 
python2 -m SimpleHTTPServer 8000
```

### Использование портов для подключения к контейнеру.

Флаг -p чтобы задать порт в своей хост-машине.

```
docker pull tutum/wordpress  
docker run -d -p 10001:80 --name blog1 tutum/wordpress
docker ps | grep blog
docker run -d -p 10002:80 --name blog2 tutum/wordpress
```
теперь на http://localhost:10001 и  http://localhost:10002 доступен веб-интерфейс.

### Создание сетей между контейнерами.
```
$ docker network create my_network (объявление сети)
$ docker network connect my_network blog1 (подключение к сети)
$ docker run -it --network my_network ubuntu:16.04 bash
(запуск контейнера с подключением к сети)
```

### Общение между контейнерами без использования сети.

```
$ docker run --name wp-mysql -e MYSQL_ROOT_PASSWORD=yoursecretpassword -d mysql
$ docker run --name wordpress --link wp-mysql:mysql -p 10003:80 -d wordpress
```
задаем контейнеру Mysql имя wp-mysql  чтобы иметь возможность обратиться к нему позже. указываем переменную среды чтобы контейнер инициализировал базу данных (-e MYSQL_ROOT_PASSWORD=yoursecretpassword) Запуск обоих контейнеров в качестве демонов. 
Образу Wordpress соеденяем с wp-mysql (--link wp-mysql:mysql) ссылка будет направление на сервер MySql, так же используем проброс портов -p 10003:80
Флаг --link устанавливает хост-файл контейнера, чтобы в нашем случае wordpress мог обращаться mysql и он будет ссылаться на любой контейнер wp-mysql что дает возможность заменить контейнер.

**Важно в данном случае соблюдать правильный порядок запуска контейнеров, имеет место просброс для имен контейнеров, которые уже существуют.**

# Глава 3. Докер как легкая виртуальная машина.

Docker - не является VM-технологией. Он не моделирует аппаратное обеспечение компьютер и не включает в себя опер. систему. По умолчанию контейнер докер не ограничен конкретными аппаратными ограничениями. Докер вирутализирует среду в которой запускают службы, а не компьютер. Докер не может с легкостью запускать ПО Windows (либо ПО для других опер. систем Unix)

> Команда ADD распоковывает TAR-файлы(а также файлы в формате gzip и другие подобные типы файлов). Образ scratch - это псевдообраз с нулевым байтом, который вы можете собрать поверх. Обычно он применяется в случаях, когда вы хотите скопировать(или добавить) полную файловую систему, используя файл Dockerfile
```
$ VMDISK="$HOME/VirtualBox VMs/myvm/myvm.vdi"
$ sudo modprobe nbd
$ sudo qemu-nbd -c /dev/nbd0 -r $VMDISK3((CO1-3))
$ ls /dev/nbd0p*
 /dev/nbd0p1 /dev/nbd0p2
$ sudo mount /dev/nbd0p2 /mnt
$ sudo tar cf img.tar -C /mnt .
$ sudo umount /mnt && sudo qume-nbd -d /dev/nbd0
```
> Для выбора раздела для монтирования выполните sudo cfdisk/dev/nbd0, чтобы увидеть, что доступно. Обратите внимание, что, если вы где-либо видите LVM, то у вашего диска нетривиальная схема разметки.

### Извлечение раздела.
```
$ sudo mount -o loop partition.dump /mnt
$ sudo tar cf $(pwd)/img.tar -C /mnt
$ sudo umount /mnt
```
### Извлечение файловой системы раб. вирт машины.
```
$ cd /
$ sudo tar cf /img.tar --exclude=/img.tar --one-file-system /
```
### Добавление архива в образ Docker
```
FROM scratch
ADD img.tar /
```
> В Docker есть альтернатива команде ADD в виде команды docker import, можно использовать в cat img.tar | docker import - new_image_name. Но сборка поверх образа с доп инструкциями потребует создания файла dockerfile, поэтому может быть проще идти через add.

Dockerfile монолитного приложения.
```
FROM ubuntu:14.04
RUN apt-get update && apt-get install postgresql npm nginx
WORKDIR /opt
COPY db /opt/db
RUN service postgresql start && \
    cat db/schema.sql | psql && \
    service postgresql stop
COPY app /opt/app
RUN cd app && npm install
RUN cd app && ./minify_static.sh
COPY conf /opt/conf
RUN cp conf/mysite /etc/nginx/sites-available/ && \
    cd /etc/nginx/sites-enabled && \
    ln -s ../sites-available/mysite
```
Dockerfile postgres
```
FROM ubuntu:14.04
RUN pat-get update && apt-get install postgresql
WORKDIR /opt
COPY db /opt/db
RUN service postgresql start && \
    cat db/schema.sql | psql && \
    service postgresql stop
```
Dockerfile nodejs
```
FROM ubuntu:14.04
RUN apt-get update && apt-get install nodejs npm
WORKDIR /opt
COPY app /opt/app
RUN cd app && npm install
RUN cd app && ./minify_static.sh
```
Dockerfile nginx
```
FROM ubuntu:14.04
RUN apt-get update && apt-get isntall nginx
WORKDIR /opt
COPY conf /opt/conf
RUN cp conf/mysite /etc/nginx/sites-available/ && \
    cd /etc/nginx/sites-enabled && \
    ln -s ../sites-available/mysite
```
Dockerfile Supervisor
```
FROM ubuntu:14.04
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update && apt-get install -y python-pip apache2 tomcat7
RUN pip install supervisor
RUN mkdir -p /var/lock/apache2
RUN mkdir -p /var/run/apache2
RUN mkdir -p /var/log/tomcat
RUN echo_supervisord_conf > /etc/supervisord.conf
ADD ./supervidord_add.conf /tmp/supervisord_add.conf
RUN cat /tmp/supervisord_add.conf >> /etc/supervisord_add.conf
RUN rm /tmp/supervisord_add.conf
CMD ["supervisord", "-c"]
```
Файл к докфайлу супервизор supervisord_add.conf
```
[supervisord]
nodaemon=true

# apache
[program:apache2]
command=/bin/bash -c "source /etc/apache2/envvars && exec /usr/sbin/apache2 -DFOREGROUND"

# tomcat
[program:tomcat]
command=service start tomcat
redirect_stderr=true
stdout_logfile=/var/log/tomcat/supervisor.log
stderr_logfile=/var/log/tomcat/supervisor.errpr_log
```

### Копирование общедоступного образа и помещение его в учетную запись Docker hub
```
docker pull debian:wheezy # тянем образ
docker tag debian:whezzy adev/debian:mywheezy1 # помечаем своим именеме и тегом
docker push adev/debian:mywheezy1 # заливает вновь созданный тег
```

