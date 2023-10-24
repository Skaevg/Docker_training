# Dockerfile
это текстовый документ в который мы определяем этапы сборки приложения.

```
# syntax=docker/dockerfile:1
FROM goland:1.21-alpine
WORKDIR /src
COPY . .
RUN go mod download
RUN go build -o /bin/client /cmd/client
RUN go build -o /bin/server ./comd/server
ENTRYPOINT [ "/bin/server" ]
```
1. syntax=docker/dockerfile:1  это директива  парсера Dockerfile, указывает какоую версию синтаксиса Dockerfile использовать
2. FROM инструкция использовать образ Goland, версии 1.21
3. WORKDIR Создает рабочий каталог внутри контейнера
4. COPY Копирует файлы в костексте сборки в рабочий каталог контейнера.
5. RUN Исполнение команды ГО
6. ENTRYPOINT Настройки контейнера как исполняемого файла.


# Добавление этапов

Используя многоступенчатые сборки можно использовать разные базовые образы для сред сборки и выполнения. Мы можем скопировать артефакты сборки со стадии сборки на стадию выполнения.

Добавим ещё один этап, использующий минимальный образ scratch в качестве основы.

```
# syntax=docker/dockerfile:1
FROM goland:1.21-alpine
WORKDIR /src
COPY go mod go.sum .
RUN go mod download
COPY . .
RUN go build -o /bin/client /cmd/client
RUN go build -o /bin/server ./comd/server

FROM scratch
COPY --from=0 /bin/client /bin/server /bin
ENTRYPOINT [ "/bin/server" ]
```
Таким образом образ будет весить всего 8.45мб вместо 150мб.
Потому что содержит только двоичные файлы и ничего больше.

# Парралелизм

Теперь можно увелить скорость сборки за  счет Multi-stage build и парралелизма. сейчас сборка создает двоичные файлы один за другим. Но нет причин по которым нам нужно создавать клиент преед созданием сервера или наоборот. 

Можно разделить этапы создания двоичного файла на отдельные этапы и в финале скопируем двоичные файлы с каждого соответствующего этапа сборки. Сегментируя эти сборки на отдельные этапы. 
Этапы сборки каждого двоичного файла требует как иснтрументы компиляции GO так и зависимости приложения. Определим эти ощие шаги как базовый этап многократного использования. Сделаем это присвоив образу имя stage_name чтобы ссылаться на этот образ в FROM в других этапах. (FROM stage_name)

Так же можно присвоить имена этапам создания двоичных файлов и ссылаться на имя в COPY --from=stage_name инстркуции при копировании двоичных файлов в scratch образ.

```
# syntax=docker/dockerfile:1
FROM goland:1.21-alpine AS base
WORKDIR /src
COPY go.mod go.sum .
RUN Go mod download
COPY . .

FROM base as build-client
RUN go build -o /bin/client /cmd/client

FROM bas as build-server
RUN go build -o /bin/server ./comd/server

FROM scratch
COPY --from=build-client /bin/client /bin/
COPY --from=buid-server /bin/server /bin/
ENTRYPOINT [ "/bin/server" ]
```
 Теперь вместо того чтобы начать собирать двоичные файлы один за другим build-client и build-server выполняются одновременно.

# Стройте цели 

Неправильно использовать клиентское и серверное приложение в одном образе. Мы может в одном докерфайле создать два образа.
Для этого используем флаг --target.

```
# syntax=docker/dockerfile:1
FROM goland:1.21-alpine AS base
WORKDIR /src
COPY go.mod go.sum .
RUN Go mod download
COPY . .

FROM base as build-client
RUN go build -o /bin/client /cmd/client

FROM bas as build-server
RUN go build -o /bin/server ./comd/server

FROM scratch as client
COPY --from=build-client /bin/client /bin/
ENTRYPOINT [ "/bin/client" ]

FROM scratch as server
COPY --from=buid-server /bin/server /bin/
ENTRYPOINT [ "/bin/server" ]
```
И теперт собираем клиент и сервер отдельными тегами.
```
$ docker build --tag=buildme-client --target=client .
$ docker build --tag=buildme-server --target=server .
$ docker images buildme
REPOSITORY      TAG     IMAGE ID        CREATED         SIZE
buildme-client  latest  657493vnj39     20 seconds ago  4.25MB
buildme-server  latest  151493vnj39     5 seconds ago   4.25MB
```

Цели(target) также позволяют избежать необходимости собирать оба двоичных файла. При выборе сборки докер будет выполнять этапы ведущие к выбранной сборке и пропускать ненужные.


# Mount

Монтирование кэша позволяет указать постоянный кэш пакетов, который будет использован во время собрки. Постоянный кэш позволяет ускорить сборку, особенно этапы связанные с установкой пакетов с помощью мэнеджера пакетов. Наличие постоянного кэша означает, что даже после перестройки слоя мы грузим только новые или изменненые пакеты.

Монтирование кеша созадется с использованием флага --mount вместе с RUN инструкцией в Dockerfile

Чтобы использовать монтирование кэша --mount=type=cache,target="path" где путь это расположение каталога кэша который мы хотим смонтировать в контейнер.

## Добавление монтирование кэша
Путь используемый для монтирования кэша зависит от используемого менеджера пакетов.

```
# syntax=docker/dockerfile:1
FROM goland:1.21-alpine AS base
WORKDIR /src
COPY go.mod go.sum .
RUN --mount=type=cache,target=/go/pkg/mod/ \
    go mod download -x
COPY . .

FROM base as build-client
RUN --mount=type=cahe,target=/go/pkg/mod/ \
    go build -o /bin/client ./cmd/client

FROM bas as build-server
RUN --mount=type=cache,target=/go/pkg/mod/ \
    go build -o /bin/server ./cmd/server

FROM scratch as client
COPY --from=build-client /bin/client /bin/
ENTRYPOINT [ "/bin/client" ]

FROM scratch as server
COPY --from=buid-server /bin/server /bin/
ENTRYPOINT [ "/bin/server" ]
```
Флаг -x добавленный к  go mod download команде, отображает процесс загрузки и позволит увидеть как на след. шаге будет использоваться монтирование кэша.

# Восстановление образа

Прежде чем пересобирать образ, очистите кеш сборки. 
```
$ docker builder prune -af
```
Теперь восстановим образ, но теперь вместе с флагом --progress=plain перенаправляя выходные данные в файл журнала.

```
$ docker build --target=client --progress=plain . 2> log1.txt
```
 
Теперь чтобы увидеть, что используется монтирование кеша, изменим версию одного из модулей GO,
Обновите версию пакета chi который использует сервер приложения
```
$ docker run -v $PWD:$PWD -w $PWD goland:1.21-alpine \
    go get github.com/go-chi/chi/v5@v5.0.8
```
Теперь запустим сборку снова.

```
$ docker build --target=client --progress=plain . 2> log2.txt
```
Теперь можно увидеть, что chi был загружен только тот пакет который изменен.


## Продолжим оптимизацию add bind mounts

Есть ещё пара оптимизаций которые мы можем реализовать для улучшения Dockerfile. 
Сейчас он использует COPY для извлечения файлов go.mod &  go.sum перед загрузкой модулей.
Вместо копирования этих файлов мы можем использовать --mount=type=bind(привязка).
Bind делает файлы доступными для контейнера непосредственно с хоста. Это изменение полностью устранит необходимость в дополнительных COPY инструкциях и слоях.

```
# syntax=docker/dockerfile:1
FROM goland:1.21-alpine AS base
WORKDIR /src
RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,source=go.sum,target=go.sum \
    --mount=type=bind,source=go.mod,target=go.mod \
    go mod download -x

FROM base as build-client
RUN --mount=type=cahe,target=/go/pkg/mod/ \
    --mount=type=bind,target=. \
    go build -o /bin/client ./cmd/client

FROM bas as build-server
RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,target=. \
    go build -o /bin/server ./cmd/server

FROM scratch as client
COPY --from=build-client /bin/client /bin/
ENTRYPOINT [ "/bin/client" ]

FROM scratch as server
COPY --from=buid-server /bin/server /bin/
ENTRYPOINT [ "/bin/server" ]
```
# Аргументы сборки

Можно передавать аргументы сборки во время сборки и установить значения по умолчанию, которое docker будет использовать в качестве запасного варианта.

```
# syntax=docker/dockerfile:1
ARG GO_VERSION=1.21
FROM goland:${GO_VERSION}-alpine AS base
WORKDIR /src
RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,source=go.sum,target=go.sum \
    --mount=type=bind,source=go.mod,target=go.mod \
    go mod download -x

FROM base as build-client
RUN --mount=type=cahe,target=/go/pkg/mod/ \
    --mount=type=bind,target=. \
    go build -o /bin/client ./cmd/client

FROM bas as build-server
RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,target=. \
    go build -o /bin/server ./cmd/server

FROM scratch as client
COPY --from=build-client /bin/client /bin/
ENTRYPOINT [ "/bin/client" ]

FROM scratch as server
COPY --from=buid-server /bin/server /bin/
ENTRYPOINT [ "/bin/server" ]
```

Ключевое ARG интерполируется в название изображения в FROM инструкции
Теперь при сборке можно задать другую версию через --build-arg
```
$ docker build --build-arg="GO_VERSION=1.19" .
```

## Ввод значений

Можно использовать аргументы сборки для изменений значений в исходном коде ваше программы во время сборки.
Это удобно для динамического ввода информации, избегая жестко запрограммированных значений.

Серверная часть приложения содержит условный оператор для печати версии приложения, если версия указана:
```
// cmd/server/main.go
var version string

func main() {
    if version != "" {
        log.Printf("Version: %s", version)
    }
}
```
Можно объявить значение строки версии непосредственно в коде. Но обновление версии для соответствия релизной версии приложения потребует обновление кода перед каждым выпуском. Это черевато ошибками.
Лучше - передать строку версии в качестве аргумента build и внедрить аргумент собрки в код.

```
# syntax=docker/dockerfile:1
ARG GO_VERSION=1.21
FROM goland:${GO_VERSION}-alpine AS base
WORKDIR /src
RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,source=go.sum,target=go.sum \
    --mount=type=bind,source=go.mod,target=go.mod \
    go mod download -x

FROM base as build-client
RUN --mount=type=cahe,target=/go/pkg/mod/ \
    --mount=type=bind,target=. \
    go build -o /bin/client ./cmd/client

FROM bas as build-server
ARG APP_VERSION="v0.0.0+unknown"
RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,target=. \
    go build -ldflags "-X main.version=$APP_VERSION" -o /bin/server ./cmd/server

FROM scratch as client
COPY --from=build-client /bin/client /bin/
ENTRYPOINT [ "/bin/client" ]

FROM scratch as server
COPY --from=buid-server /bin/server /bin/
ENTRYPOINT [ "/bin/server" ]
```

# Экспорт файлов

По умолчанию выходной формат docker build - образ контейнера. Которое автоматом помещается в локальное хранилище образов.
Но мы можем использовать local экспортер. Он сохраняет файловую систему билда контейнера в указанном каталоге на хосте.

Чтобы использовать local exporter, передайте --output параметр команде docker build. Флаг --output принимает один аргумент: место назначения на хосте .

Следующая команда экспортирует файлы из server целевого объекта в текущий рабочий каталог файловой системы хоста:
```
$ docker build --output=. --target=server .
```
Запуск это команды создает двоичный файл с раширением ./bin/server 
Он создает в bin/ каталоге, поскольку там находится файл внутри контейнера сборки.

```
$ ls -l ./bin
total 14576
-rwxr-xr-x 1 user user 56141724 Apr 6 09:25 server
```

Данные двоичные файлы не требуют среду выполнения контейнера, такой как daemon docker.

Если наша хостовая операционная система - Linux, мы можем запустить эти файлы. Для запуска на Mac или Windows требуется кросс-компиляция.

# Тест

Раздел посвещен тестированию линтингу (Linting - автоматизированный процесс анализа кода и поиска потенциальных ошибок нарушения стиля и антишаблоны, возможно даже исправление ошибок автоматически. Проверяя код на соответствие стандартам определенных правил)

Для данного примера на GO мы добавим шаг сборки golanci-lint.

## Запуск тестов

golangci-lint доступен в виде образа на Docker Hub. Прежде чем добавить шаг проверки в файл Dockerfile можно опробовать его с помощью docker run команды.
```
$ docker run -v $PWD:/test -w /test \
  golangci/gokangci-lint golangci-lint run
```
Можно увидеть, что golangci-lint работает: он находит проблему в коде, где отсутствует проверка ошибок.
```
cmd/server/main.go:23:10: Error return value of 'w.Write' is not chechked (errcheck) w.Write([]byte(translated))
```
Теперь можно добавить это шаг в dockerfile

```
# syntax=docker/dockerfile:1
ARG GO_VERSION=1.21
ARG GOLANGCI_LINT_VERSION=v1.52
FROM goland:${GO_VERSION}-alpine AS base
WORKDIR /src
RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,source=go.sum,target=go.sum \
    --mount=type=bind,source=go.mod,target=go.mod \
    go mod download -x

FROM base as build-client
RUN --mount=type=cahe,target=/go/pkg/mod/ \
    --mount=type=bind,target=. \
    go build -o /bin/client ./cmd/client

FROM bas as build-server
ARG APP_VERSION="v0.0.0+unknown"
RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,target=. \
    go build -ldflags "-X main.version=$APP_VERSION" -o /bin/server ./cmd/server

FROM scratch as client
COPY --from=build-client /bin/client /bin/
ENTRYPOINT [ "/bin/client" ]

FROM scratch as server
COPY --from=buid-server /bin/server /bin/
ENTRYPOINT [ "/bin/server" ]

FROM scratch AS binaries
COPY --from=build-client /bin/client /
COPY --from=build-server /bin/server /

FROM golangci/golangci-lint:${GOLANGCI_LINT_VERSION} as lint
WORKDIR /test
RUN --mount=type=bind, target=. \
    golanci-lint run
```

Добавленный lint этап использует golangci/golangci-lint образ из Docker hub для вызова golangci-lint run команды с привязкой для контекста сборки.

Этап проверки не зависит от других этапов Dockerfile Та ким образом, запуск обычной сборки не приведет к выполнению линт теста. Для линт теста кода тредуется указать этап lint:
```
$ docker build --target=lint .
```
## Экспорт результатов теста

Важно уметь экспортировать результаты тестов в отчет о тестировании.

Экспорт результатов теста ничем не отличается от экспорта файлов, как было приведено ранее:
1. Сохраните результаты теста в файл.
2. СОздайте новый этап в вашем Dockerfile, используя scratch базовый образ.
3. Экспортируйте с помощью localэ экспортера.

# Мультиплатформенность

Самый простой вариант началь разработку для нескольких платформ - использовать эмуляцию. С помощью эмуляции вы можете создавать свое приложение для нескольких архитектру без необходимости внесения каких-либо изменений в Dockerfile. Все, что потребуется это передать флаг --platform команде билд, указав ОС и архитектуру для который мы собираем.
```
$ docker build --target=server --platform=linux/arm/v7
```
Можно использовать эмуляцию для одновременного создания результатов разных платформ. Однако build driver по умолчанию не поддерживает одновременные многоплатформенные сборки. Итак, сначала нужно переключится на другой сборщик, который использует драйвер, поддерживающий одновременные многоплатформенные сборки.

ЧТобы переключиться на использование другого драйвера, понадобиться Docker Buildx. Buildx - клиент сборки нового поколения, который обеспечивает взаимодействие с пользователем с расширенным фукнционалом.

### Настройка сборки
Buildx поставляется в копмлекте с Docker Desktop и вы можно вызвать через docker buildx.

```
$ docker buildx version
github.com/docker/buildx v0.10.74
```
Затем создайте build использующий расширение docker-container. Выполнив следующую команду:
```
$ docker buildx create --driver=docker-container --name=container
```
Это создаст новый билд с именем container. Просмотр доступных билдов docker buildx ls
```
$ docker buildx ls
NAME/NODE           DRIVER/ENDPOINT             STATUS
container           docker-container            
    container_0     unix:///var/run/docker.sock inactive
default *           docker                      
    default         default                     running
desktop-linux       docker          
    desktop-linux   desktop-linux               running
```
Статус нового контейнера не активен и это нормально, потому что мы еще не начали его использовать.

### Сборка с использованием эмуляции.

Чтобы запустить многоплатформенные сборки с помощью Buildx, вызовите docker buildx build и передайте ей те же аргументы, что и обычному билд ранее, только добавив:
- --builder=container выбрать новый билд
- --platform=linux/amd64,linux/arm/v7,linux/arm64/v8 создание нескольких архитектру одновременно.

```
$ docker buildx build \
    --target=binaries \
    --output=bin \
    --builder=container \
    --platform=linux/amd64, linux/arm64,linux/arm/v7 .
```
Данная команда использует эмуляцию для запуска одно и той же сборки 4 раза, по одному для каждой платформы. Результаты можно увидеть в bin каталог.
```
bin
|--linux_amd64
|  |--client
|  |--server
|
|--linux_arm64
|  |--client
|  |--server
|
|--linux_arm_v7
   |--client
   |--server
```

Однако у запуска многоплатформенных сборок с использованием эмуляции есть несколько недостатков:

- При попытке выполнения выше команд, можно заметить как много времени заняло данное действие. Эмуляци может быть намного медленнее, чем собственно выполнение задач, интенсивно использующих ЦП.
- Эмуляция работает только в случаях, если архитектруа поддерживается используемым базовым образом. В примере был использован Alpine Linux golang. Это значит, что такоим образом мы можем создавать образы Linux для огранниченого набора архитектур ЦП без необходисти изменения базового образа.

Альтернативой в этом вопросе рассматривается кросс-компиляция. Кросс-компиляция делает многоплатформенные сборки гораздо быстрее и универсальнее.

## Сборка с использованием кросс-компиляции

Использование кросс-компиляции означает использование возможностей компилятора для сборки нескольких платформ без необходимости эмуляции.

Первое, что надо сделать - закрепить сборщик, чтобы он использовал собственную архитектуру узла в качестве платформы сборки. Это делают для предотвращения эмуляции.

## Аргументы сборки платформы

Этот подход предполагает использование нескольких заранее определенных аргументов сборки, к которым у вас есть доступ в сборках Docker: BUILDPLATFORM и TARGETPLATFORM (плюс производные, типа  TARGETOS). Эти аргументы сборки отражают значения, которые вы передаете флагу --platform

Например, если вы вызываете сборку с помощью --platform=linux/amd64, аргументы сборки разрешаются следующим образом:

- TARGETPLATFORM=Linux/amd64
- TARGETOS=linux
- TARGETARCH=amd64

Когда мы передаем более одного значения флагу платформы, этапы сборки использующие предварительно определенные аргументы платформы, автоматически раветвляются для каждой платформы. В этом отличие от сборок, выполняемых в режиме эмуляции, где весь конвейер сборки выполняется на каждой платформе.

```
$ docker buildx build \
    --platform=linux/x86_64,linux/arm64

                     -> STAGE2(x86)
    STAGE0 -> STAGE1                  -> STAGE3 -> 
                     -> STAGE2(arm64)
```