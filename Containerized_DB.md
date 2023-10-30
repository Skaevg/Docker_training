# Контейнеризация БД
1. Запусти контейнер с бд любого типа на твой выбор (Mysql, PostgreSQL, Mongo)
2. Убедись что контейнр запущен
3. Получи доступ к контейнеру и создай новую таблицу (или коллекцию, в зависимости от БД которую мы выбрали) для студентов
4. Вставь ряд(строку) (или документ) студента
5. Убедись, что ряд(документ) добавлен.

# Solution
```
# Run the container
podman run --name mysql -e MYSQL_USER=mario -e MYSQL_PASSWORD=tooManyMushrooms -e MYSQL_DATABASE=university -e MYSQL_ROOT_PASSWORD=MushroomsPizza -d mysql

# Verify it's running
podman ps

# Add student row to the database
podman exec -it mysql /bin/bash
mysql -u root
use university;
CREATE TABLE Students (id int NOT NULL, name varchar(255) DEFAULT NULL, PRIMARY KEY (id));
insert into Projects (id, name) values (1,'Luigi');
select * from Students;
```