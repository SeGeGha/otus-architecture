# C4, уровень 2 — Containers

У системы одно клиентское приложение, три приложения BFF, один Projection Worker, два хранилища данных, один кеш и один брокер событий.

## Диаграмма

```mermaid
flowchart TB
    user["👤 Участник маршрута"]
    admin["👤 Администратор / Модератор"]

    subgraph system["Сервис планирования путешествий"]
        spa["Web App (SPA)\n[JavaScript, браузер]\nUI + локальный Event Store\n(IndexedDB) для офлайн-режима"]

        qbff["Query BFF\nАгрегация геоданных,\nчтение маршрутов из Read Model;\nвалидация JWT локально"]

        cbff["Command BFF\nОбработка команд: создание\nи изменение маршрута;\nзапись в Event Store +\npublish в Kafka;\nвалидация JWT локально"]

        projector["Projection Worker\nConsume событий из Kafka,\nобновление Read Model"]

        rbff["Realtime BFF\nWebSocket-сервер;\nCRDT-синхронизация правок;\nJWT при handshake"]

        eventstore[("Event Store\n[БД]\nAppend-only поток событий\nмаршрута — аудит, откат,\nисточник истины")]

        readmodel[("Read Model\n[БД]\nCQRS-проекция:\nденормализованное состояние\nдля быстрого чтения")]

        redis[("Redis\nКеш геоданных L2;\npub/sub шина для\nRealtime BFF")]

        kafka["Kafka\n[брокер событий]\nТрансляция событий\nCommand BFF → Projection Worker"]
    end

    subgraph geoproviders["Внешние геопровайдеры"]
        poi["Google Places / Foursquare\n(POI)"]
        osm["OpenStreetMap\n(геоданные)"]
        mapbox["Mapbox\n(геокодинг, роутинг)"]
    end

    idp["🔑 Провайдер авторизации\n(OAuth2 / OIDC)\nвыдаёт JWT"]
    storage["📦 Файловое хранилище\n(S3 / CDN)"]

    user -->|"HTTPS"| spa
    admin -->|"HTTPS"| spa

    spa -->|"OIDC login\n(получить JWT)"| idp
    spa -->|"REST + JWT\n(чтение маршрутов, геоданные)"| qbff
    spa -->|"REST + JWT\n(создать / изменить маршрут)"| cbff
    spa <-->|"WebSocket + JWT\n(правки в реальном времени)"| rbff

    qbff -->|"REST API"| poi
    qbff -->|"REST API"| osm
    qbff -->|"REST API"| mapbox
    qbff -->|"кеш L2 (get/set)"| redis
    qbff -->|"чтение текущего\nсостояния маршрута"| readmodel
    qbff -->|"скачивание фото"| storage

    cbff -->|"append событий"| eventstore
    cbff -->|"publish (новое событие)"| kafka
    cbff -->|"загрузка фото"| storage

    kafka -->|"consume"| projector
    projector -->|"обновление проекции"| readmodel

    rbff -->|"pub/sub\n(трансляция правок\nмежду узлами BFF)"| redis

    classDef app fill:#438dd5,stroke:#2e6295,color:#ffffff;
    classDef spa fill:#23b26d,stroke:#178049,color:#ffffff;
    classDef db fill:#e8a838,stroke:#b07820,color:#000000;
    classDef broker fill:#d45c13,stroke:#a03a00,color:#ffffff;
    classDef external fill:#999999,stroke:#6b6b6b,color:#ffffff;
    classDef person fill:#08427b,stroke:#052e56,color:#ffffff;
    class qbff,cbff,rbff,projector app;
    class spa spa;
    class eventstore,readmodel,redis db;
    class kafka broker;
    class user,admin person;
    class poi,osm,mapbox,idp,storage external;
```

## Контейнеры и откуда их требования

**Web App (SPA)** — клиентское приложение в браузере. Разворачивается на CDN, не содержит серверной логики. Внутри SPA — локальный Event Store на IndexedDB для офлайн-режима (D2, S5): без сети правки копятся локально, при восстановлении соединения уходят на Command BFF. Один UI и для участников, и для модераторов (разные права в JWT / claims).

**Command BFF** — «пишущий» BFF. Принимает команды (создать маршрут, добавить точку, откатить версию, заблокировать пользователя — для модератора), пишет событие в Event Store и публикует его в Kafka. Сам Read Model не обновляет — это делает Projection Worker. Разделение Command/Query — CQRS (D3, D4; [trade-offs.md](../trade-offs.md), [ADR-0001](../adr/0001-event-sourcing-cqrs.md)). Загружает фотографии в файловое хранилище.

**Projection Worker** — отдельный сервис-подписчик. Читает события из Kafka и обновляет Read Model. В IDP и к пользователям не ходит — у него нет пользовательских запросов. Вынесен из Command BFF, чтобы запись и построение проекции масштабировались независимо. Отставание проекции ≤ 2 с допустимо.

**Query BFF** — «читающий» BFF. Параллельно опрашивает гео-провайдеров (Google Places, OpenStreetMap, Mapbox), нормализует форматы, обогащает маршрут и отдаёт единый ответ. Геоданные кешируются в Redis (L2). При отказе провайдера — отвечает из кеша с пометкой «устаревшие данные» (circuit breaker, D1, S1 и S4). Отдаёт фотографии из файлового хранилища.

**Realtime BFF** — держит WebSocket-соединения. Правки транслируются через Redis pub/sub между узлами (D4, S2, S3).

**Event Store** — источник истины. Append-only: события только добавляются. Откат — новое событие в логе. Снапшоты ускоряют восстановление (D3, S6 и S7).

**Read Model** — CQRS-проекция текущего состояния маршрута. Обновляется Projection Worker'ом из Kafka; Query BFF читает отсюда. Отставание ≤ 2 с допустимо; выгода — загрузка ≤ 500 мс без replay истории (S1).

**Redis** — кеш геоданных L2 для Query BFF и pub/sub для Realtime BFF.

**Kafka** — шина: Command BFF публикует, Projection Worker потребляет.

**Провайдер авторизации (OAuth2 / OIDC)** — выдаёт JWT при логине. На схеме с BFF **не** ходит на каждый запрос: BFF валидируют JWT сами. Редко (не нарисовано) BFF обновляют JWKS с IDP при ротации ключей.

**Файловое хранилище (S3/CDN)** — фото точек маршрута. Query читает, Command пишет.

## Что не нарисовано и почему

- **Circuit breaker и ACL-адаптеры к провайдерам** — компоненты внутри Query BFF.
- **CRDT-движок (Yjs / Automerge)** — библиотека внутри Realtime BFF и SPA.
- **Снапшот-сервис** — фоновый поток внутри Command BFF.
- **Transactional Outbox** — деталь dual write (Event Store + Kafka) внутри Command BFF.
- **L1 in-memory кеш** — внутри Query BFF.
- **Валидация JWT / JWKS-клиент** — middleware внутри каждого BFF; редкий fetch JWKS с IDP не рисуем как постоянную связь.
- **Сервис загрузки файлов** — логика внутри Command / Query BFF.
- **Админ-панель модератора** — тот же SPA с другими claims в JWT.
