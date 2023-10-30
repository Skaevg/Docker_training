## Требования
Иметь локально образы, либо запустить docker pull nginx:alpine
## Цели
1. Запустить контейнер использующие веб-сервер образ (nginx, httpd, ...)
- Задайте контейнер порт 80: локальный порт 80
- Запустите его в -d (detached) открепленном(фоновом) режиме 
- Имя задайте nginx_container
2. Убедитесь что сервер запущен и работет 
3. Создайте HTML файл со следующим содержимым и скопируйте его в контейнер по пути, по которому он будет доступен как индексный файл.
```
<html>
<head>
<title>It's a me</title>
</head>
<body>
<h1>Mario</h1>
</body>
```
4. Создайте образ запущенного контейнера и назови "nginx_mario"
5. Тэгай контенер как "mario"
6. Удали оригинальный контейнер (container_nginx) и убедись что он удален.
7. Создай новый контейнер из созданного образа (также как исходный контейнер)
8. Выполни curl 127.0.0.1:80. Что ты увидишь?
9. Выполни docker/podman diff в новом образе. Объясни вывод.

# Solution
```
# Run the container
podman run --name nginx_container -d -p 80:80 nginx:alpine

# Verify web server is running
curl 127.0.0.1:80
    #  <!DOCTYPE html>
    #  <html>
    #  <head>
    #  <title>Welcome to nginx!</title>

# Create index.html file
cat <<EOT >>index.html
<html>
<head>
<title>It's a me</title>
</head>
<body>
<h1>Mario</h1>
</body>
EOT

# Copy index.html to the container
podman cp index.html nginx_container:/usr/share/nginx/html/index.html

# Create a new image out of the running container
podman commit nginx_container nginx_mario

# Tag the image
podman image ls
# localhost/nginx_mario     latest   dc7ed2343521   52 seconds ago   25 MB
podman tag dc7ed2343521 mario

# Remove the container
podman stop nginx_container
podman rm nginx_container
podman ps -a # no container 'nginx_container'

# Create a container out of the image 
podman run -d -p 80:80 nginx_mario

# Check the container created from the new image
curl 127.0.0.1:80
#<html>
#<head>
#<title>It's a me</title>
#</head>
#<body>
#<h1>Mario</h1>
#</body>

# Run diff
podman diff nginx_mario

C /etc
C /etc/nginx/conf.d
C /etc/nginx/conf.d/default.conf
A /run/nginx.pid
C /usr/share/nginx/html
C /usr/share/nginx/html/index.html
C /var/cache/nginx
C /var
C /var/cache
A /var/cache/nginx/client_temp
A /var/cache/nginx/fastcgi_temp
A /var/cache/nginx/proxy_temp
A /var/cache/nginx/scgi_temp
A /var/cache/nginx/uwsgi_temp

# We've set new index.html which explains why it's changed (C)
# We also created the image while the web server is running, which explains all the files created under /vars
```