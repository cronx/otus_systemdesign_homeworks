# Сценарий 1. Пользователь загружает видео в YouTube

## 1. Порядок взаимодействия
1. **Пользователь** — в WebClient выбирает видеофайл и инициирует загрузку
2. **WebClient** — отправляет в API запрос на загрузку видео
3. **API** — обращается к AuthService для проверки прав пользователя
4. **AuthService** — проверяет права пользователя и возвращает результат в API
5. **API** — перенаправляет запрос на создание загрузки в Upload Service
6. **Upload Service** — сохраняет метаданные видео в DB
7. **WebClient** — передаёт видеофайл в Object Storage
8. **Object Storage** — сохраняет видеофайл
9. **WebClient** — уведомляет Upload Service через API о завершении загрузки
10. **Upload Service** — проверяет наличие загруженного файла в Object Storage
11. **Upload Service** — отправляет в Broker событие что загружен файл и нужно его обработать
12. **Broker** — передаёт событие в TranscodingService
14. **TranscodingService** — получает видеофайл из Object Storage, перекодирует видео в несколько разрешений и форматов, сохраняет обработанные версии видео в Object Storage.
15. **TranscodingService** — уведомляет Publication Service о завершении обработки видео
16. **Publication Service** — обновляет статус видео в DB на READY
17. **CDN** — при первом запросе получает необходимые видеосегменты из Object Storage и кэширует их для последующих просмотров. Затем держит в кеше

## 2. Схема

```mermaid
flowchart TD
    U["Пользователь"]
    C["WebClient"]
    API["API"]
    AUTH["AuthService"]
    UP["Upload Service"]
    OBJ[("Object Storage")]
    DB[("DB")]
    MQ["Broker"]
    TC["TranscodingService"]
    PUB["Publication Service"]
    CDN["CDN"]

    U -->|"1. Выбирает видео"| C
    C -->|"2. Запрашивает загрузку"| API
    API -->|"3. Проверяет права"| AUTH
    AUTH -->|"4. Возвращает результат"| API

    API -->|"5. Создаёт загрузку"| UP
    UP -->|"6. Сохраняет метаданные UPLOADING"| DB
    UP -->|"7. Возвращает параметры загрузки через API"| C

    C -->|"8. Передаёт видеофайл"| OBJ
    OBJ -->|"9. Подтверждает сохранение"| C
    C -->|"10. Уведомляет о завершении через API"| UP
    UP -->|"11. Проверяет наличие файла"| OBJ
    UP -->|"12. Обновляет статус UPLOADED"| DB

    UP -->|"13. Публикует VIDEO_UPLOADED"| MQ
    MQ -->|"14. Передаёт событие"| TC

    TC -->|"15. Читает исходный файл"| OBJ
    TC -->|"16. Перекодирует видео"| TC
    TC -->|"17. Сохраняет готовые версии"| OBJ

    TC -->|"18. Сообщает о завершении"| PUB
    PUB -->|"19. Устанавливает статус READY"| DB
    PUB -->|"20. Разрешает доставку видео"| CDN
    CDN -->|"21. Получает сегменты при Cache MISS"| OBJ

    classDef client fill:#e3f0ff,stroke:#3978b8,color:#183b62
    classDef service fill:#e9f5e9,stroke:#438c59,color:#244d30
    classDef storage fill:#fff0d9,stroke:#c28b38,color:#694619
    classDef async fill:#f0e4fa,stroke:#9566be,color:#563573

    class U,C client
    class API,AUTH,UP,TC,PUB,CDN service
    class DB,OBJ storage
    class MQ async
```

## 3. Архитектурные решения

| Компонент | Ответственность |
|---|---|
| WebClient | Пользовательский интерфейс |
| API | Приём и маршрутизация запросов |
| AuthService | Аутентификация и авторизация |
| Upload Service | Управление загрузкой видео |
| Object Storage | Хранение исходных и обработанных видео |
| DB | Хранение метаданных и статусов видео |
| Broker | Асинхронная доставка событий |
| TranscodingService | Перекодирование видео |
| Publication Service | Управление публикацией |
| CDN | Кэширование и доставка видео |

### Ключевые принципы

1. Видеофайл загружается напрямую в Object Storage, минуя API.
2. Upload Service управляет загрузкой, но не передаёт через себя весь видеофайл.
3. Перекодирование выполняется асинхронно через Broker.
4. Исходные и обработанные видео хранятся отдельно от метаданных.
5. CDN обеспечивает доставку готового видео зрителям.
