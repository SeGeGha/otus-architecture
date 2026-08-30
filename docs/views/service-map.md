# Карта сервисов

Здесь всё, что разобрано в `[04-capabilities.md](../04-capabilities.md)`, `[05-service-boundaries.md](../05-service-boundaries.md)` и `[06-cohesion-coupling.md](../06-cohesion-coupling.md)`, сведено в одну картинку. Сервисы, чем каждый владеет и что передаётся по связям.

## Диаграмма

```mermaid
flowchart TB
    user["Участник"]:::person
    admin["Модератор"]:::person

    subgraph platform["Сервис планирования путешествий"]
        direction TB
        RTE["<b>Маршруты</b> (core)<br/>события, проекция, точки, сессия"]:::core
        GEO["<b>Геокаталог</b><br/>карточки мест, кеш, placeId"]:::supporting
        ACS["<b>Доступ</b><br/>участники, роли, ссылки"]:::supporting
        MOD["<b>Модерация</b><br/>решения, блокировки"]:::supporting
        NTF["<b>Уведомления</b><br/>заявки, статусы, шаблоны"]:::generic
        AUTH["<b>Авторизация</b><br/>JWT / JWKS"]:::generic
    end

    subgraph prov["Внешние провайдеры"]
        POI["Google Places / Foursquare"]:::external
        OSM["OpenStreetMap"]:::external
        MBX["Mapbox"]:::external
        IDP["IdP OIDC"]:::external
    end

    user --> RTE
    user --> GEO
    user --> ACS
    admin --> MOD

    RTE -->|"контракт:<br/>можно ли править"| ACS
    RTE -->|"контракт:<br/>карточка места"| GEO
    ACS -.->|"событие:<br/>участник приглашён"| NTF
    RTE -.->|"событие:<br/>маршрут откатился"| NTF
    MOD -.->|"событие:<br/>маршрут скрыт"| RTE
    MOD -.->|"событие:<br/>пользователь заблокирован"| ACS
    GEO -.->|"событие:<br/>место обновлено"| RTE

    GEO -->|Геопровайдер| POI
    GEO -->|Геопровайдер| OSM
    GEO -->|Геопровайдер| MBX
    user -->|"OIDC login"| IDP

    classDef core fill:#1168bd,stroke:#0b4884,color:#ffffff;
    classDef supporting fill:#438dd5,stroke:#2e6295,color:#ffffff;
    classDef generic fill:#85bbf0,stroke:#5d82a8,color:#000000;
    classDef external fill:#999999,stroke:#6b6b6b,color:#ffffff;
    classDef person fill:#08427b,stroke:#052e56,color:#ffffff;
```

Сплошная стрелка — синхронный вызов по контракту, пунктирная — асинхронное событие. Цвет показывает тип поддомена: тёмно-синий core, светлее supporting, самый светлый generic.

Токен авторизации проверяет каждый сервис на входе, и если нарисовать все связи, получится пачка линий, которые перечеркнут схему и ничего не объяснят.