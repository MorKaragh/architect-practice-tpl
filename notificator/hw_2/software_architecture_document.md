# Технический проект "Сервис отправки оповещений"

>Автор: Сергей Андреевич Степанов

---

## Текущая архитектура

В текущей архитектуре у нас есть мобильное приложение, которое общается с компонентом "Controller", а он в свою очередь делает запросы к "Foo" и "Bar".

![alt text](static/current_arch.svg)


## Целевая архитектура

### С1; Уровень контекста
Основная функция системы - предоставление REST API для отправки нотификаций по одному или сразу нескльким каналам (push, SMS, email).

В дополнение, система предоставляет средства управления шаблонами сообщений, средства управления ограничениями и лимитами на отправку для пользователей, а так же средствами сбора аналитической информации.

![C1](static/c1.png)

### С2: Уровень модулей ("контейнеров")

Система распределённая, состоящая из нескольких частей, взаимодействующих в основном по протоколам http(s) (кроме БД). О причинах выбора распределённой архитектуры читайте в (ADR #3).

#### Основные части системы:
- **Core API**. Основной модуль, предоставляющий RESTful API потребителям и реализующий алогоритмы обработки сообщений, лимитов, шаблонов, а так же обработку критических ошибок отправки. Технология - Java + Spring (ADR #1).
- **Sender'ы**. Набор похожих друг на друга модулей системы, ответственных за взаимодействие с конкретным внешним сервисом отправки того или иного вида нотификаций. Технология - Java + Spring (ADR #1).
- **OLTP база данных**. Хранилище транзакционных данных, обслуживающее модуль Core API. Досканально описывать что именно там будет храниться мы не будем, но основные объекты это: лимиты и настройки ограничений для пользователей, шаблоны сообщений, учётные записи и авторизационные данные (но это не точно). В процессе дальнейшего проектирования и разработки список может меняться. Технология - Postgresql (ADR #1).
- **Cache**. Распределенный кеш, откуда система будет получать шаблоны и настройки пользователя. Требуется для безпроблемного масштабирования системы и обеспечения высокой пропускной способности и производительности. Управление данными кеша реализуем паттерном Cache-Aside, протухание ключей политикой volatile-ttl. Подробнее в (ADR #2).
- **Load Balancer**. Т.к. система предполагает горизонтальное масштабирование - необходимый элемент (надеюсь, что это очевидно). Обоснование - в (ADR #1), а о масштабируемости поговорим чуть ниже.
- **OLAP хранилище**. Тут 100% уверенности нет, но предполагается что бизнес захочет получать аналитику на основании данных отправленных уведомлений. Предусмотрим этот компонент заранее. Подробнее в (ADR #1).

#### Интеграции системы
- Для отправки нотификаций на iOS - сервис [APNs](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/sending_notification_requests_to_apns)
- Для отправки нотификаций на Android - сервис [FCM](https://firebase.google.com/docs/cloud-messaging?hl=ru)
- Для отправки email и SMS - неустановленные абстрактные сервисы, т.к. я не хочу их брать "с потолка", а вариантов на рынке масса и их выбор зависит от конкретного производственного контекста предприятия.

![C2](static/c2.png)

### С3: Уровень компонентов

#### Основные объекты данных

- Задание на нотификацию. Входные данные для процесса отправки нотификации. Содержит перечисление каналов, через которые требуется отправить нотификацию, контент сообщения для каждого из каналов, идентификатор шаблона сообщения (опционально) для каждого из каналов, время для отложенной отправки (опционально). Обязательные мета-данные запроса - аутентификационные заголовки.
- Лимиты и ограничения. Объект, содержащий информацию, на основании которой система разрешает или запрещает отправку тех или иных уведомлений.
- Шаблон. Текстовый шаблон, служащий для генерирования типовых сообщений. Подробнее в (ADR #4).
- Нотификация. Конкретное сообщение, привязанное к каналу оправки нотификации (push, SMS, email), обрабатываемое одним из модулей-отправщиков (sender).

#### Верхне-уровневое описание процесса обработки задания на нотификацию

1) Запрос от пользователя поступает через *Load Balancer* в модуль *Core API* в компонент *Notification Controller*
2) Запрос от пользователя проходит авторизацию и аутентификацию с помощью компонента *Security* (используем Spring Security)
3) Зарос от пользователя проверяется на превышение лимитов и другие ограничения в компоненте *Limiter*. При необходимости, лимиты пользователя корректируются в *Cache* и *OLTP DB* (см. стратегию кеширования в (ADR #2))
4) Если требуется, запрос обогощается шаблонизированным контентом с помощью компонента *Template Processor*, который берёт шаблон из *Cache* или *OLTP DB* (см. стратегию кеширования в (ADR #2))
5) Запрос от пользователя разбивается на отдельные нотификации и они массивом передаются в компонент *Router*
6) *Router* отправляет каждую из полученных нотификаций в соответствующей ей топик в модуле *Message Broker*, то есть применяется [паттерн Datatype channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DatatypeChannel.html)
7) Модули *Sender* получают сообщения (сразу или по расписанию) и используя соответствующий модулю сторонний сервис производят отправку
8) Если отправка не удалась и есть смысл попробовать ещё раз (например, внешний сервис временно не доступен) - *sender* пытается отправить ещё раз, т.е. используется [паттерн Retry](https://dev.to/frosnerd/resilience-design-patterns-retry-fallback-timeout-circuit-breaker-2870)
9) Если лимит повторных попыток исчерпан или полученная ошибка говорит о бесполезности повторных попыток - нотификация отправляется в топик DLQ в брокере сообщений, то есть применяется [паттерн Dead Letter Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DeadLetterChannel.html)
10) Компонент *Failure Processor* получает сообщение из DLQ и вносит корректировки в лимиты пользователя в компоненте *Limiter*, а так же передаёт "мёртвую" нотификацию в компонент *Notification Processor* для возможной последующей обработки и отражения в аналитике

#### Верхне-уровневое описание администрирования 
> Тут ничего писать не буду. В реальности эту систему точно хотели бы конфигурировать, но как именно и что именно делать - надо конкретные требования анализировать. Поэтому в домашке я просто ограничусь упоминанием о неких административных действиях, но не буду вдаваться в то, что они из себя представляют. И так уже тонна текста в документе.


![C3](static/c3.png)


## Пара слов про прототипирование

Описанное выше не является архитектурой прототипа и разрабатывалась как архитектура MVP и последующих стадий развития проекта. Из постановки задачи создается ощущение что мы уже знаем чего хотим. Но если представить что мы решили потратить время на прототип, то давайте определим что мы хотим проверить и как можно реализовать прототип так, чтобы потом из него вырастить целевую архитектуру. 

#### Гипотезы, которые стоит проверить:

- Если выделить обработчики для каналов в отдельные компоненты то их будет просто добавлять
- Если не использовать кеширование то база данных будет read-heavy и станет узким горлом системы
- Если реализовать retry-стратегию и DLQ то итоговая надёжность доставки будет выше, чем если применить только один из паттернов или вовсе обойтись без них

#### Некоторые тезисы для проверки на прототипе:
- Предлагаемое API удобно в использовании
- Нам нужен отдельный компонент для аналитики
- Нас устраивает аутентификация и авторизация с хранением УЗ в своей БД

#### Упрощения архитектуры, рекомендуемые к применению в прототипе:
- отказаться от Cache, пусть будет только БД
- не делать ничего для аналитики
- не разворачивать отдельный брокер сообщений, заменить внутренними очередями в Java
- не делать распределенную систему, вместо этого сделать монолит и модули отделять как java-модули или пакеты с идеей вынести их потом отдельно

По результатам прототипирования можно будет внести коррективы в документ.

---

## Журнал архитектурных решений


### Выбор стека технологий 

#### ID: 1

#### Дата: 08.10.2023

#### Статус: Принято

#### Участники:

* Степанов Сергей Андреевич

#### Решения:
* Использование [Java + Spring](https://spring.io/) в качестве ЯП и основного фреймворка для разрабатываемых модулей
* Использование [Postgresql](https://www.postgresql.org/) в качестве OLTP базы данных
* Использование [ActiveMQ](https://activemq.apache.org/) в качестве Message Broker
* Использование [Redis](https://redis.io/) в качестве системы кеширования
* Использование [Nginx](https://nginx.org/ru/) в качестве балансировщика нагрузки
* Использование [ClickHouse](https://clickhouse.com/) в качестве хранилища аналитических данных

#### Контекст:
Так как в задании ничего не сказано про стек - он выбирается по усмотрению архитектора (меня). 

В качестве платформы разработки будет использована [Java + Spring](https://spring.io/). Отлично подходит, т.к. широко развит и распространен, не имеет критичных ограничений для задачи и содержит все необходимые инструменты.

В качестве БД используем [Postgresql](https://www.postgresql.org/). Подходит, т.к. тоже широко распространен и развит, содержит механизмы масштабирования из коробки, а так же на территории РФ в наличии множество предложений по администрированию, настройке и решению сложных проблем. 

В качестве Message Broker выбран [ActiveMQ](https://activemq.apache.org/). Подходящее решение, т.к. обеспечивает высокую надёжность и производительность, а так же имеет функциональность "умного брокера", важную в текущем проекте - возможность отложенной (по расписанию) отправки сообщений.

В качестве системы кеширования выбран [Redis](https://redis.io/) из-за возможностей масштабирования, высокой производительности, распространенности на рынке и низкого порога входа, а так же наличия подходящих типов данных для кеширования в системе (вообще, нам хватит банального key-value где value будет строкой).

В качестве балансировщика нагрузки применим [Nginx](https://nginx.org/ru/) как де-факто стандарт в индустрии, с которым знакома масса специалистов на рынке.

В качестве OLAP-хранилища выбран [ClickHouse](https://clickhouse.com/) потому что я ничего особенного про OLAP не знаю, узнавать сейчас не хочу, но в курсе что в Яндексе его любят. 

Все выбранные технологии прекрасно совместимы между собой.

Все выбранные технологии - open source, что позволит сэкономить на лицензиях и быстро запуститься в полноценном production-режиме.

#### Последствия:

#### Положительные:
* Система использует современный, актуальный, подходящий под задачу стэк технологий
* Стек технологий не требует затрат денег (по крайней мере пока не понадобится поддержка)

#### Отрицательные:
* фанаты языка Go будут расстроены и сядут изучать\вспоминать Java
---
### Система кеширования

#### ID: 2

#### Дата: 08.10.2023

#### Статус: Принято

#### Участники:

* Степанов Сергей Андреевич

#### Решения:
* Использовать систему кеширования для быстрого доступа к часто запрашиваемым данным (шаблонам и настройкам пользователя)
* Управлять стратегией кеширования в модуле Core API, применяя паттерн [Cache-Aside](https://dev.to/husniadil/cache-aside-pattern-559f)

#### Контекст:

Т.к. система нотификаций в своей сути предполагает значительное количество запросов, количество которых по мере роста бизнеса будет кратно увеличиваться, ожидается большая нагрузка на чтение из хранилища. Для обработки высокой нагрузки на чтение применим кеширование с помощью Redis (ADR #1). 

Наличие отдельного распределенного кеша позволит легко горизонтально масштабировать систему, добавляя дополнительные Core API модули по мере необходимости.

Управление кешированием сделаем максимально простым и прозрачным способом, применив паттерн паттерн [Cache-Aside](https://dev.to/husniadil/cache-aside-pattern-559f). 

Суть подхода:

при чтении:
1) ищем данные в кеше
2) если промах - достаем данные из БД, кладем в кеш и используем

при записи:
1) записываем данные в БД
2) записываем данные в кеш

Кеш применяем только к тем данным, которые ожидаем запрашивать часто. 

Возникающий риск неконсистентности данных БД-кеш митигируем "протуханием" ключа, используя политику volatile-ttl. Подробнее тут: [Redis Eviction Policies](https://redis.io/docs/reference/eviction/)

#### Последствия:

#### Положительные:
* Значительные возможности масштабирования системы
* Увеличение производительности и пропускной способности

#### Отрицательные:
* Незначительное усложнение системы
* Риск неконсистентности данных (митигирован стратегией кеширования)
---
### Распределённость системы

#### ID: 3

#### Дата: 08.10.2023

#### Статус: Принято

#### Участники:

* Степанов Сергей Андреевич

#### Решения:
* Использовать распределённую архитектуру для системы
* Выделение обработчика сообщений (sender'а) для каждого канала нотификаций в отдельный модуль
* Реализуемые модули системы должны быть [stateless](https://www.techtarget.com/whatis/definition/stateless-app)

#### Контекст:

Два основных аргумента в пользу распределённого решения:
1) Предполагаемая высокая нагрузка
2) Предполагаемая разница в нагрузке на разные каналы нотификаций (push, SMS, email)

Говоря про первое, в какой-то момент в систему может начать приходить значительное количество запросов, и один модуль Core API перестанет справляться. Это риск, избежать которого мы сможем заранее заложив в систему возможность горизонтального масштабирования. 

Таким образом, модуль Core API должен иметь возможность быть реплицированным в любом количестве в любой момент времени без необходимости доработки системы. Поэтому обязательное требование, применяемое к нему - отсутствие состояния, т.е. он должен быть stateless.

Второй аргумент неразрывно связан с первым и к нему применимы те же идеи, но тут есть нюанс - нам может понадобиться точечное масштабирование. Специфика системы предполагает, что на разные каналы нотификаций может быть разная нагрузка, меняющаяся со временем - например, на старте мы будем отправлять email'ов в 100 раз больше, чем SMS, а затем роль лидера по объему могут занять push-уведомления на Android. 

Неравномерное распределение нагрузки порождает ряд требований к модулям-обработчикам (sender'ам): модули должны быть независимыми (чтобы нагрузка на один канал не мешала работать другому и не было "соревнования" за ресурсы) и каждый вид обработчика должен иметь возможность быть масштабирован в любой момент времени. Значит, тут тоже обязательный пункт - stateless.

Бонусом мы получим упрощение добавления\замены конкретных реализаций отправки нотификаций и типов самих нотификаций, а так же увеличение надежности за счет репликации (если потребуется).

#### Последствия:

#### Положительные:
* Возможность простого горизонтального масштабирования системы
* Изолированность ресурсов каналов отправки нотификаций друг от друга
* Простота добавления\изменения реализаций каналов нотификаций
* Простота замены и сочетания внешних поставщиков сервисов нотификаций
* Увеличение надёжности системы за счёт репликации важных узлов

#### Отрицательные:
* Увеличение сложности реализации системы
* Увеличение сложности инфрастурктуры и развёртывания
---
### Асинхронное взаимодействие

#### ID: 4

#### Дата: 08.10.2023

#### Статус: Принято

#### Участники:

* Степанов Сергей Андреевич

#### Решения:
* Отправку конкретных нотификаций сделать с применением очереди сообщений, каждому каналу - отдельная очередь (топик)
* Обработчики очередей (sender'ы) разделить по каналам, одному обработчику - один тип сообщений (один топик)
* Реализовать retry-стратегию на sender'ах
* Сообщения, которые не возможно обработать, отправлять в специализированный канал для "мертвых" сообщений (DLQ)

#### Контекст:

Чтобы функционирование системы не имело жесткой зависимости от внешних сервисов (таких, как APNs или email-шлюз), взаимодействие с ним сделаем через очередь сообщений. Для каждого такого сервиса сделаем отдельную очередь чтобы добиться независимости каналов друг от друга и предотвратить гонку за ресурсы, то есть применим [паттерн Datatype channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DatatypeChannel.html).

Чтобы обрабатывать временные ошибки отправки на sender'ах применим [паттерн Retry](https://dev.to/frosnerd/resilience-design-patterns-retry-fallback-timeout-circuit-breaker-2870) c количеством попыток = 5 и задержкой = 1 минуте (может меняться в зависимости от каналов и требований).

Чтобы обрабатывать критичные ошибки отправки нотификаций применим [паттерн Dead Letter Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DeadLetterChannel.html), возвращая "мёртвую" нотификацию назад в Core API и там как-то обрабатывая (как именно выдумывать тут не буду).

#### Последствия:

#### Положительные:
* Высокая надёжность отправки нотификаций
* Митигация риска недоступности внешних сервисов

#### Отрицательные:
* Увеличение сложности реализации системы
* Увеличение сложности поддержки, необходимость администрирования и мониторинга брокера сообщений
---