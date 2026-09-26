# Протокол Parmigiano

[English](README.md)

Protobuf-протокол, пакеты чата в реальном времени, присутствие и обновление клиента.

Этот репозиторий — контракт обмена для Parmigiano. Это не приложение. Клиенты и серверы подключают схему, генерируют код и обмениваются protobuf-пакетами с префиксом длины. Опубликованный модуль: [buf.build/parmigiano/protocol](https://buf.build/parmigiano/protocol).

Смысл каждого поля и сообщения описан в [proto/parmigiano/README.md](proto/parmigiano/README.md).

## Как устроен пакет

По соединению за раз идёт один пакет:

```text
[ uint32 length ][ байты protobuf ]
```

`length` — беззнаковое 32-битное число в порядке big-endian. Оно считает только байты protobuf. Сами 4 байта префикса в длину не входят.

Клиент сериализует `ClientMessage`. Сервер сериализует `ServerMessage`. В каждом сообщении заполнена ровно одна ветка `payload`. `ClientMessage.user_id` — отправитель, внутри payload он не повторяется.

Имя пакета: `parmigiano.v1`.

| Файл | Кто отправляет | Что внутри |
| --- | --- | --- |
| `proto/parmigiano/v1/common.proto` | обе стороны | Общие типы: вид содержимого, файлы, каталоги |
| `proto/parmigiano/v1/client.proto` | клиент | Подключение, действия в чате, присутствие, проверка обновления |
| `proto/parmigiano/v1/server.proto` | сервер | События чата, публичные ключи, разрыв соединения, манифест обновления |

## Состав репозитория

```text
proto/parmigiano/v1/          схема
proto/parmigiano/README.md    описание сообщений на английском и русском
buf.yaml                      модуль Buf buf.build/parmigiano/protocol
buf.gen.examples/             примеры шаблонов buf generate
.github/workflows/check.yml   lint и проверка совместимости на pull request
.github/workflows/publish.yml отправка в Buf Schema Registry
```

Сгенерированный код пишется в `gen/` и в git не коммитится.

## Что установить

Установите [Buf CLI](https://buf.build/docs/installation/). Проверка:

```bash
buf --version
```

Buf нужен, чтобы проверять схему и генерировать код. Библиотека времени выполнения ставится отдельно и зависит от языка. В примерах ниже указано, какая библиотека нужна каждому плагину.

Клонировать репозиторий нужно тем, кто меняет схему или генерирует код из локальных файлов. Разработчик приложения может зависеть от модуля в Buf Schema Registry после публикации и не клонировать этот репозиторий.

## Как менять схему

1. Правьте `.proto` в `proto/parmigiano/v1/`.
2. Отступ — 4 пробела. `buf format` переписывает файлы на 2 пробела, в этом репозитории его не запускайте.
3. Если изменился смысл поля или сообщения, обновите [proto/parmigiano/README.md](proto/parmigiano/README.md) на обоих языках.
4. Из корня репозитория проверьте схему:

```bash
buf lint
buf build
```

Правила, из-за которых старый и новый код продолжают понимать друг друга:

- Номера полей — это контракт на проводе. Номер существующего поля не меняют и не занимают заново.
- Чтобы убрать поле или значение enum, удалите его и добавьте `reserved` на номер и имя.
- Новое поле получает следующий свободный номер.
- Новое действие клиента или сервера — это новое сообщение и новая ветка в `oneof payload`.
- Первое значение enum равно `0`, а имя заканчивается на `_UNSPECIFIED`.
- Остальные имена значений начинаются с имени enum в `UPPER_SNAKE_CASE`. У `DeleteReason` префикс `DELETE_REASON_`. Два enum в одном пакете не могут иметь одинаковое имя значения.
- `oneof` несёт один payload. Второе тело в то же сообщение не кладут.

Совместимые добавления — новые поля, новые ветки `oneof` и новые значения enum. Несовместимые изменения — переименованные типы, смена типа поля и удалённые номера. Несовместимую схему кладите в новый пакет, например `parmigiano.v2`, а не переписывайте `v1`.

## Pull request

Откройте pull request в `main`. Workflow Check запускает `buf lint` и `buf breaking` против `main`. Он ничего не публикует.

`buf breaking` использует правила `FILE` из `buf.yaml`. Он отклоняет удаления и смену типов, которые сломают схему, уже лежащую в `main`.

Если поломка намеренная, поставьте на pull request метку `buf skip breaking`. Lint при этом всё равно выполняется. Метку ставьте только когда совместимость нужно сломать сознательно. Для настоящей поломки провода лучше новый пакет.

Прямой push в `main` Check не запускает. Проверка идёт через pull request.

## Релизы и теги

Слияние в `main` запускает Publish. Этот workflow выполняет `buf push` и обновляет модуль в Buf Schema Registry. Повторно lint и breaking он не гоняет: они уже прошли на pull request.

Publish также запускается, когда вы пушите тег вида `v*.*.*`:

```bash
git tag v1.2.0
git push origin v1.2.0
```

Версии — по [semantic versioning](https://semver.org/):

| Тег | Когда |
| --- | --- |
| `v1.2.3` | Документация или другое изменение без эффекта на провод |
| `v1.3.0` | Совместимое добавление: новые поля, сообщения или значения enum |
| `v2.0.0` | Несовместимое изменение, вместе с новым пакетом protobuf |

Тег фиксирует релиз. Метка модуля `main` в BSR сдвигается на каждом слиянии. Потребителям стоит закреплять именно тег версии.

Публикация использует секрет GitHub Actions `BUF_TOKEN`. Это API-токен Buf с правом `module.push` на `parmigiano/protocol`. Его создают в настройках аккаунта Buf и сохраняют в секретах репозитория под этим именем. `GITHUB_TOKEN` выдаёт сам GitHub Actions, отдельный секрет для него не нужен.

## Как подключить схему

В репозитории приложения код не генерируют. После Publish Buf Schema Registry собирает пакет для каждого языка, который он поддерживает. Пакет ставят менеджером пакетов этого языка.

Модуль: `buf.build/parmigiano/protocol`. Готовые фрагменты для текущего коммита лежат на вкладке [SDKs](https://buf.build/parmigiano/protocol/sdks). Команды ниже ставят последний коммит метки по умолчанию. Для релиза версию копируют с этой страницы.

Обзор: [buf.build/docs/bsr/generated-sdks/overview](https://buf.build/docs/bsr/generated-sdks/overview/).

| Язык | Установка | Документация |
| --- | --- | --- |
| Go | `go get buf.build/gen/go/parmigiano/protocol/protocolbuffers/go@latest` | [Go](https://buf.build/docs/bsr/generated-sdks/go/) |
| JavaScript и TypeScript | `npm install @buf/parmigiano_protocol.bufbuild_es` | [npm](https://buf.build/docs/bsr/generated-sdks/npm/) |
| Java | `build.buf.gen:parmigiano_protocol_protocolbuffers_java` | [Maven](https://buf.build/docs/bsr/generated-sdks/maven/) |
| Kotlin | `build.buf.gen:parmigiano_protocol_protocolbuffers_kotlin` | [Maven](https://buf.build/docs/bsr/generated-sdks/maven/) |
| Swift | пакет `buf.parmigiano_protocol_apple_swift` | [Swift](https://buf.build/docs/bsr/generated-sdks/swift/) |
| Python | `pip install parmigiano-protocol-protocolbuffers-python` | [Python](https://buf.build/docs/bsr/generated-sdks/python/) |
| Rust | `cargo add --registry buf parmigiano_protocol_community_neoeinstein-prost` | [Cargo](https://buf.build/docs/bsr/generated-sdks/cargo/) |
| C# | `dotnet add package BSR.Parmigiano.Protocol.Protocolbuffers.Csharp` | [NuGet](https://buf.build/docs/bsr/generated-sdks/nuget/) |
| C++ | CMake `FetchContent`, цель `parmigiano_protocol_protocolbuffers_cpp` | [CMake](https://buf.build/docs/bsr/generated-sdks/cmake/) |

Для C готового SDK нет. Его генерируют локально через `buf.gen.examples/c`.

### Go

[Go SDK](https://buf.build/docs/bsr/generated-sdks/go/). Свежий push может появиться в `proxy.golang.org` примерно через 30 минут. Чтобы брать модуль прямо из BSR, задайте `GOPRIVATE=buf.build/gen/go`.

```bash
go get buf.build/gen/go/parmigiano/protocol/protocolbuffers/go@latest
```

Путь импорта: `buf.build/gen/go/parmigiano/protocol/protocolbuffers/go/parmigiano/v1`.

### JavaScript и TypeScript

[npm SDK](https://buf.build/docs/bsr/generated-sdks/npm/). Один пакет покрывает оба языка. Один раз направьте scope `@buf` на BSR:

```bash
npm config set @buf:registry https://buf.build/gen/npm/v1
npm install @buf/parmigiano_protocol.bufbuild_es
```

### Java и Kotlin

[Maven и Gradle](https://buf.build/docs/bsr/generated-sdks/maven/). Добавьте репозиторий `https://buf.build/gen/maven` и зависимости:

```text
build.buf.gen:parmigiano_protocol_protocolbuffers_java
build.buf.gen:parmigiano_protocol_protocolbuffers_kotlin
```

Строку версии возьмите со вкладки [SDKs](https://buf.build/parmigiano/protocol/sdks). Kotlin-код опирается на Java-типы, поэтому приложению на Kotlin нужен и Java-артефакт.

### Swift

[Swift SDK](https://buf.build/docs/bsr/generated-sdks/swift/).

```bash
swift package-registry set https://buf.build/gen/swift --scope=buf
```

Идентификатор пакета: `buf.parmigiano_protocol_apple_swift`. Имя продукта: `Parmigiano_Protocol_Apple_Swift`. Для цели Xcode вставьте Git URL со вкладки SDKs, а не реестр SPM.

### Python

[Python SDK](https://buf.build/docs/bsr/generated-sdks/python/).

```bash
pip install parmigiano-protocol-protocolbuffers-python --extra-index-url https://buf.build/gen/python
```

### Rust

[Cargo SDK](https://buf.build/docs/bsr/generated-sdks/cargo/). Крейты появляются, когда открыта страница модуля или когда Cargo запрашивает пакет, а дальше — на каждом push в метку по умолчанию.

`.cargo/config.toml`:

```toml
[registries.buf]
index = "sparse+https://buf.build/gen/cargo/"
credential-provider = "cargo:token"
```

```bash
cargo login --registry buf "Bearer TOKEN"
cargo add --registry buf parmigiano_protocol_community_neoeinstein-prost
```

### C#

[NuGet SDK](https://buf.build/docs/bsr/generated-sdks/nuget/). Добавьте источник `https://buf.build/gen/nuget/index.json` и сопоставьте ему пакеты `BSR.*`, как на той странице. Затем:

```bash
dotnet add package BSR.Parmigiano.Protocol.Protocolbuffers.Csharp
```

### C++

[CMake SDK](https://buf.build/docs/bsr/generated-sdks/cmake/). Нужен CMake 3.14 или новее. Один раз выполните `buf registry login`. Скопируйте `FetchContent_Declare` со вкладки [SDKs](https://buf.build/parmigiano/protocol/sdks) для плагина `protocolbuffers/cpp`. Имя библиотеки: `parmigiano_protocol_protocolbuffers_cpp`. В схеме нет gRPC, плагин `grpc/cpp` не добавляйте.

SDK даёт классы сообщений и `SerializeToArray` / `ParseFromArray`. Префикс из 4 байт длины остаётся в коде приложения.

### Генерация на своей машине

Локальная генерация необязательна. Она нужна, если у языка нет пакета в BSR или файлы должны лежать в дереве проекта. Шаблоны лежат в `buf.gen.examples/`. Из корня репозитория:

```bash
buf generate --template buf.gen.examples/go/buf.gen.yml
buf generate --template buf.gen.examples/java/buf.gen.yml
buf generate --template buf.gen.examples/kotlin/buf.gen.yml
buf generate --template buf.gen.examples/csharp/buf.gen.yml
buf generate --template buf.gen.examples/swift/buf.gen.yml
buf generate --template buf.gen.examples/python/buf.gen.yml
buf generate --template buf.gen.examples/rust/buf.gen.yml
buf generate --template buf.gen.examples/cpp/buf.gen.yml
buf generate --template buf.gen.examples/c/buf.gen.yml
buf generate --template buf.gen.examples/typescript/buf.gen.yml
buf generate --template buf.gen.examples/javascript/buf.gen.yml
```

| Шаблон | Каталог |
| --- | --- |
| `go` | `gen/go` |
| `java` | `gen/java` |
| `kotlin` | `gen/kotlin` |
| `csharp` | `gen/csharp` |
| `swift` | `gen/swift` |
| `python` | `gen/python` |
| `rust` | `gen/rust` |
| `cpp` | `gen/cpp` |
| `c` | `gen/c` |
| `typescript` | `gen/ts` |
| `javascript` | `gen/js` |

`gen/` в git не входит.

## Лицензия

См. [LICENSE](LICENSE).
