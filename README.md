# MSA-GoFuture

Итоговый проект курса «Микросервисная архитектура». Кейс «GoFuture»: такси-агрегатор, который нужно перевести с Django-монолита на глобальную микросервисную платформу (500 тыс. конкурентных поездок, Юго-Восточная Азия и Южная Америка, 99,95-99,99% доступности, выход фичи за 2 недели).

Автор: Михаил К. Дата: 21.10.2026. Каждое задание оформлено как набор ADR (контекст, требования, решение, альтернативы, компромиссы и риски). Диаграммы в PlantUML (C4), PNG генерируются из `.puml` workflow'ом [plantuml.yml](.github/workflows/plantuml.yml).

## Артефакты по заданиям

| Задание | Что сдаём | Файлы |
|:--|:--|:--|
| **Task1.** Домены и декомпозиция | Нефункциональные требования, карта сервисов, диаграмма C2 To-Be с обратной совместимостью, очерёдность выделения, план миграции данных, дорожная карта на 12 месяцев | [ADR.md](Task1/ADR.md), [C2_ToBe](Task1/C2_ToBe.png) ([puml](Task1/C2_ToBe.puml)) |
| **Task2.** Event-Driven | Событийная платформа на Kafka (C2), каталог событий, топики и партиционирование по регионам, Flink, сага бронирования, надёжная доставка, подход, метрики и инструменты мониторинга | [ADR.md](Task2/ADR.md), [C2_EventPlatform](Task2/C2_EventPlatform.png) ([puml](Task2/C2_EventPlatform.puml)), [Saga_Booking_Sequence](Task2/Saga_Booking_Sequence.png) ([puml](Task2/Saga_Booking_Sequence.puml)) |
| **Task3.** Высокая нагрузка | Выбор регионов, репликация, геомаршрутизация, диаграмма C4 failover и таблица сценариев отказа, C2 с защитой периметра (изменения красным), стратегия комплаенса | [ADR.md](Task3/ADR.md), [C2_Security_Regions](Task3/C2_Security_Regions.png) ([puml](Task3/C2_Security_Regions.puml)), [Replication](Task3/Replication.png) ([puml](Task3/Replication.puml)), [GeoRouting](Task3/GeoRouting.png) ([puml](Task3/GeoRouting.puml)), [Failover_C4](Task3/Failover_C4.png) ([puml](Task3/Failover_C4.puml)) |
| **Task4.** Платформа данных | Диаграмма C4 пайплайна от микросервисов к ML-платформе и BI, ML-платформа, BI, механизмы качества данных | [ADR.md](Task4/ADR.md), [C4_DataPipeline](Task4/C4_DataPipeline.png) ([puml](Task4/C4_DataPipeline.puml)) |
| **Task5.** Мультитенантность | C2 с IAM, мониторингом и потоками данных онбординга, модель изоляции данных, процесс онбординга, таблица ролей и уровней доступа | [ADR.md](Task5/ADR.md), [C2_Multitenant_IAM](Task5/C2_Multitenant_IAM.png) ([puml](Task5/C2_Multitenant_IAM.puml)), [Onboarding_Sequence](Task5/Onboarding_Sequence.png) ([puml](Task5/Onboarding_Sequence.puml)), [C2_Data_Isolation](Task5/C2_Data_Isolation.png) ([puml](Task5/C2_Data_Isolation.puml)) |

## Ключевые решения

- **Декомпозиция (Task1).** Strangler Fig через API Gateway, Anti-Corruption Layer, CDC (Debezium) и единственный писатель на сущность, простой ноль. Порядок: платформа, Notification, Geography, Analytics, Payment Gateway, Pricing, Fraud, Dispatch и Driver, Booking, Payments (леджер последним).
- **События (Task2).** Отдельный Kafka-кластер на регион, ключ партиции равен агрегату, Flink для окон по ячейкам H3 (surge, горячие зоны), оркестрируемая сага бронирования, outbox и идемпотентные консьюмеры, три класса доставки.
- **Глобальное развёртывание (Task3).** Четыре региона (SGP, JKT, GRU, SCL) в парах внутри блока, асинхронная репликация RPO не более 5 с, RTO региона не более 5 мин (цель, проверяется учениями), Geo DNS и edge WAF, резидентность данных по странам.
- **Данные и ML (Task4).** Lakehouse (Iceberg) плюс ClickHouse, три скорости данных, псевдонимизация на границе региона, Feast, MLflow и KServe, контроль качества на каждом слое.
- **Мультитенантность (Task5).** Pool + Silo (RLS и выделенные ресурсы), Keycloak Organizations с SSO и SCIM, OPA, онбординг на Temporal (шаги O1-O10), матрица ролей с JIT-доступом.

## Допущения и неоднозначности кейса

Описаны в [Task1/ADR.md](Task1/ADR.md#допущения). Главные:

1. Нагрузка выведена из допущений: поездка 20 минут, геопозиция раз в 4 секунды, около 420 заказов и 160 тыс. событий геопозиции в секунду на пике. Требуется проверка замерами.
2. В кейсе доступность 99,95% (бизнес-требования) и 99,99% (Task3). Для Tier-0 принято 99,99%, для остальных 99,95%.
3. В кейсе облако названо как Yandex Cloud, EC2 и AWS ELB одновременно. Решение не привязано к вендору (Kubernetes, Terraform), провайдер выбирается тендером. Площадки Yandex Cloud не в целевых рынках.
4. Заданий пять, а в тексте кейса указаны директории Task1-Task4. Добавлена директория Task5.
5. Юридические нормы по странам даны как ориентир, перед запуском рынка нужно заключение юристов.

## Структура репозитория

```
MSA-GoFuture/
├── README.md
├── .github/workflows/plantuml.yml   # генерация PNG из .puml
├── Task1/   ADR.md, C2_ToBe.puml/.png
├── Task2/   ADR.md, C2_EventPlatform, Saga_Booking_Sequence
├── Task3/   ADR.md, C2_Security_Regions, Replication, GeoRouting, Failover_C4
├── Task4/   ADR.md, C4_DataPipeline
└── Task5/   ADR.md, C2_Multitenant_IAM, Onboarding_Sequence, C2_Data_Isolation
```
