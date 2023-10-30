# Контейнеризация базы данных с постоянным хранилищем

1. Запусти контенер с бд любого типа.
2. Используй точку монтирования mount на своей хост машине для бд, вместо того что бы использовать хранилище контейнера
3. Объясни когда используешь хост-хранилище вместо контейнер-хранилища почему это лучший выбор.
4. Проверь работу контейнера.
# Solutin
```
# Create the directory for the DB on host
mkdir -pv ~/local/mysql
sudo semanage fcontext -a -t container_file_t '/home/USERNAME/local/mysql(/.*)?'
sudo restorecon -R /home/USERNAME/local/mysql

# Run the container
podman run --name mysql -e MYSQL_USER=mario -e MYSQL_PASSWORD=tooManyMushrooms -e MYSQL_DATABASE=university -e MYSQL_ROOT_PASSWORD=MushroomsPizza -d mysql -v /home/USERNAME/local/mysql:/var/lib/mysql/db

# Verify it's running
podman ps
```
Лучше использовать хранилище хоста потому, что в контейнере есть веротность удаления( или хранилище будет освобождено ), а данные БД будут все еще доступны.  