# ИБП Мастер

## 1. Развертывание

### 1.1. Требования

- Развернуты чистые БД Postgres и ClickHouse
- На машине с БД ClickHouse открыт порт `9000`
- Наличие SSL сертификатов (можно использовать `letsencrypt`)

### 2.1. Настройка

#### 2.1.1. Общие настройки переменных среды

2.1.1.1. Заполнить шаблон конфига `compose.env`, прописав пути к SSL сертификатам.

#### 2.1.2. Развертывание KeyCloak

2.1.2.0. В Postgres создать базу данных `keycloak`.

2.1.2.1. Заполнить шаблон конфига `envs/keycloak.env`.

2.1.2.2. Запустить KeyCloak выполнив:

```bash
docker compose up keycloak -d
```

2.1.2.3. В браузере открыть `https://<KC_HOSTNAME>:8443` и войти под администратором: логин - `<KEYCLOAK_ADMIN>`, пароль - `<KEYCLOAK_ADMIN_PASSWORD>`.

2.1.2.4. Открыть `https://<KC_HOSTNAME>:8443/admin/master/console/#/master/add-realm` и создать новый realm с идентификатором `<KC_REALM>`.

2.1.2.5. Открыть `https://<KC_HOSTNAME>:8443/admin/master/console/#/<KC_REALM>/clients/add-client` и создать нового client:

- General settings / Client ID = `<KC_CLIENT>`
- Capability config / Client authentication = `On`
- Capability config / Authorization = `On`
- Login settings / \* = `https://<KC_HOSTNAME>`

2.1.2.6. Выбрать realm `<KC_REALM>`, перейти в Client / `<KC_CLIENT>` / Credentials. Скопировать Client Secret в переменную `KC_SECRET`.

2.1.2.7. Выбрать realm `<KC_REALM>`, перейти в Client / `<KC_CLIENT>` / Service accounts roles. Нажать Assign role, выбрать Filter by client, добавить все роли из "realm-management".

2.1.2.8. Выбрать realm `<KC_REALM>`, перейти в Realm settings / Keys. Для алгоритма RS256 нажать Public key и скопировать содержимое в переменную `KC_TOKEN_PUBKEY`.

2.1.2.9. Выбрать realm `<KC_REALM>`, перейти в Realm settings / Login. Установить параметры:

- Remember me = `On`
- Email as username = `On`
- Login with email = `On`

#### 2.1.3. Развертывание приложения

2.1.3.0. В случае отсутствия необходимости наличия отдельных портов для каждого компонента приложения, их нужно удалить в `docker-compose.yml`.

2.1.3.1. Заполнить оставшиеся шаблоны конфигов (кроме `x-transfer-agent.env`).

2.1.3.2. В Postgres создать базы данных: `xks`, `xlogs`, `xmodel`, `xtransfer`, `xuser`.

2.1.3.3. В Postgres базе данных `xtransfer` создать схему `integration`.

2.1.3.4. Запустить RabbitMQ выполнив:

```bash
docker compose up rabbitmq -d
```

2.1.3.5. Войти в management интерфейс RabbitMQ по ссылке `https://<KC_HOSTNAME>:15671`, с логином `<RABBITMQ_DEFAULT_USER>` и паролем `<RABBITMQ_DEFAULT_PASS>`.

- Создать виртуальные хосты `xmodel` и `xtransfer`.
- В виртуальном хосте `xmodel` создать очередь `xmodel_sync_agent`.

2.1.3.6. Полностью остановить и запустить все компоненты приложения выполнив:

```bash
docker compose stop && docker compose up -d
```

2.1.3.7. В Postgres в базе данных `xuser` в таблице `role` добавить новую запись с параметрами:

- id = `ADMIN`
- name = `Администратор`
- createdBy = `0`
- updatedBy = `0`

2.1.3.8. В Postgres в базе данных `xuser` в таблице `catalog` добавить новые записи с параметрами:

- id = `DICTEDIT`, name = `Изменение данных словарей`
- id = `DICTVIEW`, name = `Просмотр содержимого словарей`
- id = `INTEGRATIONEDIT`, name = `Изменение интеграции`
- id = `INTEGRATIONRUN`, name = `Запуск интеграций`
- id = `INTEGRATIONVIEW`, name = `Просмотр интеграций`
- id = `KSEDIT`, name = `Управление подключением к Knowledge Space`
- id = `MODELEDIT`, name = `Изменение модели данных`
- id = `MODELVIEW`, name = `Просмотр модели данных`
- id = `SCDESIGNEDIT`, name = `Изменение примитивов`

2.1.3.9. В Postgres в базе данных `xuser` в таблице `rolecatalog` добавить новые записи с параметрами:

- catalogId = `DICTEDIT`, roleId = `ADMIN`
- catalogId = `DICTVIEW`, roleId = `ADMIN`
- catalogId = `INTEGRATIONEDIT`, roleId = `ADMIN`
- catalogId = `INTEGRATIONRUN`, roleId = `ADMIN`
- catalogId = `INTEGRATIONVIEW`, roleId = `ADMIN`
- catalogId = `KSEDIT`, roleId = `ADMIN`
- catalogId = `MODELEDIT`, roleId = `ADMIN`
- catalogId = `MODELVIEW`, roleId = `ADMIN`
- catalogId = `SCDESIGNEDIT`, roleId = `ADMIN`

#### 2.1.4. Создание пользователя-администратора

2.1.4.1. Открыть `https://<KC_HOSTNAME>:8443/admin/master/console/#/<KC_REALM>/users/add-user` и создать пользователя с параметрами:

- Email verified = `On`
- Email = `<KEYCLOAK_ADMIN>@px`
- First name = `Admin`
- Last name = `Admin`

После создания, перейти на вкладку Credentials, нажать Set password, установить Temporary = `Off`, установить пароль `<KEYCLOAK_ADMIN_PASSWORD>`.

2.1.4.2. В Postgres в базе данных `xuser` в таблице `user` добавить новую запись с параметрами:

- id = `1`
- email = `<KEYCLOAK_ADMIN>@px`
- password = ` `
- activated = `true`
- blocked = `false`
- name = `Admin`
- surname = `Admin`
- createdBy = `0`
- updatedBy = `0`

2.1.4.3. В Postgres в базе данных `xuser` в таблице `userrole` добавить новую запись с параметрами:

- userId = `1`
- roleId = `ADMIN`

#### 2.1.5. Развертывание агента интеграции

2.1.5.0. Войти в приложение под пользователем-администратором.

2.1.5.1. В приложении открыть вкладку "Агенты интеграции" и создать нового агента с идентификатором `MAIN`.

2.1.5.2. В `envs/x-transfer-agent.env` вставить значение из "Ключ" в переменную `PLANX_AGENT_UUID` и значение из "Секрет" в переменную `SECRET`.

2.1.5.3. Перезапустить агент интеграции выполнив:

```bash
docker compose stop xtransfer-agent && docker compose up xtransfer-agent -d
```
