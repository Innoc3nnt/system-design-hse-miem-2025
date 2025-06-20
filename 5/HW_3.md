## Домашка 3: Базы данных

1. Взять ФТ и НФТ из HW_1
2. Взять сервисы из HW_2
3. Подумать какие БД будут под какие сценарии, описать сценарий выбора
4. Описать дополнительно: репликация, шардинг

### Базы данных и сценарии

| Сервис                  | Тип данных                       | Выбор БД                    | Обоснование выбора                                         |
| ----------------------- | -------------------------------- | --------------------------- | ---------------------------------------------------------- |
| Auth Service            | Пользователи, токены             | PostgreSQL                  | Поддержка ACID, транзакций, хорошо работает с IAM и OAuth2 |
| Config Service          | Настройки клиентов               | PostgreSQL / Etcd (для kv)  | Простые схемы, возможен кэш, нужна консистентность         |
| Dashboard Service       | Визуальные дашборды              | Redis (кэш) + PostgreSQL    | Кэширование + хранение пользовательских шаблонов           |
| Report Service          | PDF/Excel отчёты, шаблоны        | PostgreSQL + Object Storage | Структурированные шаблоны + бинарные файлы                 |
| Notification Service    | Уведомления                      | MongoDB                     | Часто обновляемые, semi-structured сообщения               |
| Deployment Service      | Статус развёртывания             | PostgreSQL / TimescaleDB    | Журналирование событий, расписания обновлений              |
| Grid Monitoring Service | Сырые данные от устройств        | TimescaleDB                 | Высокая вставка, агрегаты, window-функции                  |
| Analytics Engine        | Модели, расчёты, прогнозы        | ClickHouse / MinIO          | Быстрые агрегации, аналитика на больших объёмах            |
| Security Event Service  | Инциденты, попытки проникновения | ElasticSearch + PostgreSQL  | Full-text + структурированные события для аудита и отчётов |

| Сценарий                      | Где храним                        | Как получаем                                  |
| ----------------------------- | --------------------------------- | --------------------------------------------- |
| Настройки клиента             | `Config Service → PostgreSQL`     | REST/GRPC + кэш в памяти                      |
| Мониторинг 1.9 млн устройств  | `Grid Monitoring → TimescaleDB`   | Постоянный ingestion + materialized views     |
| Отчёт за период               | `Report → PostgreSQL + S3`        | Синхронная генерация, с кэшированием шаблонов |
| Попытки атак                  | `Security → Elastic + PostgreSQL` | Поиск по индексу, отправка в уведомления      |
| Уведомления о событиях        | `Notification → MongoDB`          | Событийная модель, WebSocket или push         |
| Аналитика оптимального тарифа | `Analytics → ClickHouse`          | Периодические батчи, быстрые SELECT-запросы   |

### Репликации
|Компонент|Репликация|Цель|
|---|---|---|
|PostgreSQL (Auth, Config, Reports)|Master-Slave, async|Масштабирование чтения, отказоустойчивость|
|Redis|Active-Replica, Redis Sentinel|Высокая доступность кеша|
|MongoDB|Replica Set|Надёжность хранения semi-structured данных|
|TimescaleDB|Streaming Replication|Быстрое восстановление, горизонтальное чтение|
|ClickHouse|Distributed Tables|Высокопроизводительные аналитические запросы|
|ElasticSearch|Multi-node + replicas|Доступность и отказоустойчивость|
### Шардинг
| БД / Сервис              | Шардинг                    | Ключ шардинга                    | Обоснование                             |
| ------------------------ | -------------------------- | -------------------------------- | --------------------------------------- |
| TimescaleDB (Monitoring) | Time-based + по клиенту    | `tenant_id + timestamp`          | Локализация данных по клиенту и времени |
| ClickHouse (Analytics)   | Distributed Shards         | `tenant_id`                      | Разделение аналитики по заказчикам      |
| MongoDB (Notifications)  | Sharded Cluster (optional) | `user_id` или `tenant_id`        | Распределение сообщений между узлами    |
| PostgreSQL               | На уровне БД               | `schema per tenant` или `column` | Изоляция данных клиентов                |
