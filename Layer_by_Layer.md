# Layer by Layer
## Цели
узнать о слоях образа
## Требуется
Убедиться что докер установлен на вашей системе и сервис запущен.
```
rpm -qa | grep docker
systemctl status docker
```
## Иснтрукции
1. Написать докерфайл.
```
FROM ubuntu
EXPOSE 212
ENV foo=bar
WORKDIR /tmp
RUN dd if=/dev/zero of=some_file bs=1024 count=0 seek=1024
RUN dd if=/dev/zero of=some_file bs=1024 count=0 seek=1024
RUN dd if=/dev/zero of=some_file bs=1024 count=0 seek=1024
```
2. Собрать образ используя написанный докерфайл 
```
docker image build -t super_cool_app:latest
```
3. Использование каких инструкций создает новые слои и какие добавляли метаданные образа?
```
FROM, RUN -> new layer
EXPOSE, ENV, WORKDIR -> metadata
```
4. Как можно убедиться в ответе на последний вопрос?

Когда мы запускаем docker image history super_cool_app он покажет вашу каждую инструкцию и её размер. Обычно инструкции по созданию новых слоев имеют ненулевой размре, но это не что на что можно положится само по себе, поскольку есть команды запуска которые имеют нулеовй размер в выводе истории образов докер (например ls -l)

Вы так же можете использовать docker image isnpect super_cool_app и посмотреть вывод в разделе "RootFS" количество слоев и соответствующих инструкциями, которые должны создавать новые слои.

5.  Как мы можем уменьшить размер созданного образа?

Объединим RUN инструкции водин слой.
```
RUN dd if=/dev/zero of=some_file bs=1024 count=0 seek=1024 && dd if=/dev/zero of=some_file bs=1024 count=0 seek=1024 && dd if=/dev/zero of=some_file bs=1024 count=0 seek=1024
```
Это изменит размер.
