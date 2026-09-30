# C4, уровень 2 — Containers

Контейнерный вид сведён с [картой сервисов](service-map.md): каждый сервис из [`05-service-boundaries.md`](../05-service-boundaries.md) — отдельная рамка со своими деплой-единицами и своими хранилищами. Общих баз между сервисами нет; общая только шина событий.

## Диаграмма

```mermaid
flowchart TB
    user["👤 Участник маршрута"]
    admin["👤 Модератор"]

    subgraph system["Сервис планирования путешествий"]
        spa["Web App (SPA)<br/>[JavaScript, браузер]<br/>UI + локальный Event Store<br/>(IndexedDB) для офлайн-режима"]

        subgraph routes["Маршруты (core)"]
            cbff["Command BFF<br/>Команды маршрута;<br/>запись в Event Store;<br/>реакция на внешние события"]
            qbff["Query BFF<br/>Чтение маршрута<br/>из Read Model"]
            rbff["Realtime BFF<br/>WebSocket-сервер;<br/>CRDT-синхронизация правок"]
            projector["Projection Worker<br/>Построение проекции<br/>из событий маршрута"]
            eventstore[("Event Store<br/>[PostgreSQL]<br/>Append-only поток событий,<br/>снапшоты — источник истины")]
            readmodel[("Read Model<br/>[БД]<br/>Проекция текущего состояния<br/>со снимками мест")]
            pubsub[("Realtime Pub/Sub<br/>[Redis]<br/>Трансляция правок<br/>между узлами Realtime BFF")]
        end

        subgraph geo["Геокаталог (supporting)"]
            geoapi["Geo Catalog API<br/>Поиск и карточка места;<br/>адаптеры провайдеров,<br/>circuit breaker, кеш L1"]
            placesdb[("Places DB<br/>[БД]<br/>Нормализованные карточки,<br/>внешний id → placeId")]
            geocache[("Geo Cache<br/>[Redis]<br/>Кеш L2 ответов провайдеров")]
        end

        subgraph access["Доступ (supporting)"]
            accessapi["Access API<br/>Участники, роли,<br/>приглашения, шаринг-ссылки;<br/>решение allow / deny"]
            accessdb[("Access DB<br/>[БД]")]
        end

        subgraph moderation["Модерация (supporting)"]
            modapi["Moderation API<br/>Жалобы, решения<br/>по маршрутам и нарушителям"]
            moddb[("Moderation DB<br/>[БД]")]
        end

        subgraph notify["Уведомления (generic)"]
            ntfworker["Notification Worker<br/>Приём заявок, выбор канала,<br/>шаблоны, ретраи, DLQ"]
            ntfdb[("Notification DB<br/>[БД]<br/>Заявки, статусы, шаблоны")]
        end

        kafka["Kafka<br/>[брокер событий]<br/>Общая шина; топик<br/>принадлежит издателю"]
    end

    subgraph geoproviders["Внешние геопровайдеры"]
        poi["Google Places / Foursquare<br/>(POI)"]
        osm["OpenStreetMap<br/>(геоданные)"]
        mapbox["Mapbox<br/>(геокодинг, роутинг)"]
    end

    idp["🔑 Провайдер авторизации<br/>(OAuth2 / OIDC)<br/>выдаёт JWT"]
    delivery["📨 Email / Push-провайдеры"]

    user -->|"HTTPS"| spa
    admin -->|"HTTPS"| spa
    spa -->|"OIDC login"| idp

    spa -->|"REST: открыть маршрут"| qbff
    spa -->|"REST: команды маршрута"| cbff
    spa <-->|"WebSocket: живые правки"| rbff
    spa -->|"REST: поиск мест"| geoapi
    spa -->|"REST: приглашения, роли, ссылки"| accessapi
    spa -->|"REST: жалобы, решения"| modapi

    cbff -->|"append событий"| eventstore
    qbff -->|"чтение"| readmodel
    projector -->|"обновление проекции"| readmodel
    rbff <-->|"pub/sub"| pubsub

    cbff -->|"контракт: можно ли править"| accessapi
    rbff -->|"контракт: можно ли править<br/>(при handshake)"| accessapi
    qbff -->|"контракт: можно ли читать"| accessapi
    cbff -->|"контракт: GET /places/{placeId}"| geoapi

    geoapi --> placesdb
    geoapi -->|"get / set"| geocache
    geoapi -->|"Геопровайдер"| poi
    geoapi -->|"Геопровайдер"| osm
    geoapi -->|"Геопровайдер"| mapbox

    accessapi --> accessdb
    modapi --> moddb
    ntfworker --> ntfdb
    ntfworker -->|"SMTP / push API"| delivery

    cbff -.->|"события маршрута,<br/>RouteRolledBack"| kafka
    geoapi -.->|"PlaceUpdated"| kafka
    accessapi -.->|"ParticipantInvited"| kafka
    modapi -.->|"RouteHidden, UserBlocked"| kafka

    kafka -.->|"события маршрута"| projector
    kafka -.->|"RouteHidden, PlaceUpdated"| cbff
    kafka -.->|"UserBlocked"| accessapi
    kafka -.->|"ParticipantInvited,<br/>RouteRolledBack"| ntfworker

    classDef app fill:#438dd5,stroke:#2e6295,color:#ffffff;
    classDef spa fill:#23b26d,stroke:#178049,color:#ffffff;
    classDef db fill:#e8a838,stroke:#b07820,color:#000000;
    classDef broker fill:#d45c13,stroke:#a03a00,color:#ffffff;
    classDef external fill:#999999,stroke:#6b6b6b,color:#ffffff;
    classDef person fill:#08427b,stroke:#052e56,color:#ffffff;
    class cbff,qbff,rbff,projector,geoapi,accessapi,modapi,ntfworker app;
    class spa spa;
    class eventstore,readmodel,pubsub,placesdb,geocache,accessdb,moddb,ntfdb db;
    class kafka broker;
    class user,admin person;
    class poi,osm,mapbox,idp,delivery external;
```

Сплошная стрелка — синхронный вызов, пунктирная — асинхронное событие через Kafka. Обозначения те же, что на [карте сервисов](service-map.md).

## Контейнеры по сервисам

| Сервис | Контейнеры | Хранилища |
|---|---|---|
| Маршруты (core) | Command BFF, Query BFF, Realtime BFF, Projection Worker | Event Store, Read Model, Realtime Pub/Sub |
| Геокаталог | Geo Catalog API | Places DB, Geo Cache |
| Доступ | Access API | Access DB |
| Модерация | Moderation API | Moderation DB |
| Уведомления | Notification Worker | Notification DB |
| Авторизация | — (внешний IdP) | — |

Ни одно хранилище не встречается в двух строках — это та же проверка, что и в [карте владения данными](../05-service-boundaries.md#карта-владения-данными).

**Web App (SPA)** — клиентское приложение в браузере. Разворачивается на CDN, не содержит серверной логики. Внутри SPA — локальный Event Store на IndexedDB для офлайн-режима (D2, S5): без сети правки копятся локально, при восстановлении соединения уходят на Command BFF. Один UI и для участников, и для модераторов (разные права в JWT / claims).

### Маршруты

Четыре контейнера одного вертикального среза: все обслуживают одну ответственность и меняются по одной причине ([06](../06-cohesion-coupling.md#1-маршруты)).

**Command BFF** — «пишущий» BFF. Принимает команды (создать маршрут, добавить точку, откатить версию), спрашивает у Доступа «можно ли править», пишет событие в Event Store и публикует его в Kafka. При добавлении точки получает карточку у Геокаталога по контракту `GET /places/{placeId}` и сохраняет снимок отображаемых полей в событии — в кеш Геокаталога не ходит ([Риск 2](../06-cohesion-coupling.md#риск-2-маршруты-сами-лезут-в-кеш-геокаталога)). Кроме того, слушает чужие события и переводит их в события маршрута: `RouteHidden` → маршрут снят с публикации, `PlaceUpdated` → решение, обновлять ли снимок. Разделение Command/Query — CQRS (D3, D4; [03-trade-offs.md](../03-trade-offs.md), [ADR-0001](../adr/0001-event-sourcing-cqrs.md)).

**Projection Worker** — подписчик на события маршрута. Обновляет Read Model; к пользователям и в IdP не ходит. Вынесен из Command BFF, чтобы запись и построение проекции масштабировались независимо. Отставание проекции ≤ 2 с допустимо (S10).

**Query BFF** — «читающий» BFF. Отдаёт маршрут из Read Model вместе со снимками мест, поэтому при открытии маршрута Геокаталог не дёргает (S1). Перед ответом спрашивает у Доступа «можно ли читать» — это же покрывает открытие по шаринг-ссылке.

**Realtime BFF** — держит WebSocket-соединения. Право на правку проверяет у Доступа при handshake. Правки транслируются через Redis pub/sub между узлами (D4, S2, S3).

**Event Store** — источник истины. Append-only: события только добавляются. Откат — новое событие в логе. Снапшоты ускоряют восстановление (D3, S6 и S7).

**Read Model** — CQRS-проекция текущего состояния маршрута. Обновляется Projection Worker'ом; Query BFF читает отсюда. Выгода — загрузка ≤ 500 мс без replay истории (S1).

**Realtime Pub/Sub** — Redis только для шины правок между узлами Realtime BFF. С кешем Геокаталога не делится.

### Геокаталог

**Geo Catalog API** — единственный контейнер, который знает провайдеров ([Риск 1](../06-cohesion-coupling.md#риск-1-каждый-сервис-сам-ходит-в-геопровайдеры)). Параллельно опрашивает Google Places, OpenStreetMap и Mapbox, нормализует ответы в карточку места за ≤ 500 мс (S9). При отказе провайдера отвечает из кеша с пометкой «данные могут быть устаревшими» (circuit breaker, D1, S4). Отдаёт поиск участнику в UI и карточку `GET /places/{placeId}` Маршрутам. Когда провайдер поправил данные, публикует `PlaceUpdated`.

**Places DB** — нормализованные карточки и соответствие внешнего id внутреннему `placeId`.

**Geo Cache** — Redis, кеш L2 ответов провайдеров с TTL под тип данных.

### Доступ

**Access API** — решает, кто какой маршрут видит и меняет: отвечает `allow | deny` на запросы трёх BFF Маршрутов, принимает от SPA команды «пригласить», «выдать роль», «выпустить ссылку». Приглашение не доставляет сам — публикует `ParticipantInvited`. Слушает `UserBlocked` и отзывает права заблокированного.

**Access DB** — участники, роли, приглашения, шаринг-токены.

### Модерация

**Moderation API** — принимает жалобы участников и решения модератора. Маршрут не редактирует и писем не шлёт: наружу уходят только события `RouteHidden` и `UserBlocked`, применяют их Маршруты и Доступ.

**Moderation DB** — решения, причины, блокировки.

### Уведомления

**Notification Worker** — без публичного API: заявки приходят событиями (`ParticipantInvited`, `RouteRolledBack`) вместе с получателем. Выбирает канал, подставляет шаблон, отправляет через внешних Email / Push-провайдеров, повторяет при сбоях, неудачи уводит в DLQ. Статусы доставки обратно никто не читает.

**Notification DB** — заявки, история статусов, шаблоны, маршрутизация «канал → провайдер».

### Общее

**Kafka** — общая шина платформы. Это инфраструктура, а не общая база: у каждого топика один издатель, схема события — его контракт. Подписчик не знает, как устроено хранилище издателя.

**Провайдер авторизации (OAuth2 / OIDC)** — выдаёт JWT при логине. Сервисы с IdP на каждый запрос **не** ходят: каждый контейнер с внешним API валидирует JWT сам. Своего контейнера у «Авторизации» нет — мы владеем только контрактом проверки JWT.

## Что не нарисовано и почему

- **Circuit breaker, ACL-адаптеры провайдеров, L1 in-memory кеш** — компоненты внутри Geo Catalog API.
- **CRDT-движок (Yjs / Automerge)** — библиотека внутри Realtime BFF и SPA.
- **Снапшот-сервис** — фоновый поток внутри Command BFF.
- **Transactional Outbox** — у каждого издателя (Command BFF, Geo Catalog API, Access API, Moderation API): запись в свою БД и публикация в Kafka без dual write.
- **Валидация JWT / JWKS-клиент** — middleware внутри каждого контейнера с внешним API; редкий fetch JWKS с IdP не рисуем как постоянную связь.
- **API Gateway / маршрутизация запросов SPA** — на этом уровне не принципиальна; SPA показан обращающимся к контейнерам напрямую.
- **Админ-панель модератора** — тот же SPA с другими claims в JWT.
