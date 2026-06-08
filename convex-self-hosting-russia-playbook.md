# Плейбук: перенос Finly на self-hosted Convex + PostgreSQL в РФ

Версия: `0.1`

Дата среза: `2026-06-08`

Решение в рамке: оставить архитектуру Convex, но перенести backend, PostgreSQL и S3-совместимое хранилище на российскую инфраструктуру.

## Короткий Вывод

Лучший рабочий вариант для Finly сейчас — `self-hosted Convex + управляемый PostgreSQL + S3` у Timeweb Cloud в РФ.

Почему:

- проект уже глубоко завязан на Convex: `29` таблиц, примерно `177` серверных Convex-функций и `64` клиентских хука `useQuery` / `useMutation` / `useAction`;
- сохраняются текущие `query`, `mutation`, `action`, `internalAction`, cron-задачи, schema/indexes, realtime-клиент и большая часть auth-логики;
- перенос решает главную боль: доступность без VPN на территории РФ;
- стоимость production-контура получается около `3 909 ₽/мес` без переписывания продукта;
- переписывание на Supabase или чистый PostgreSQL дороже по времени и рискованнее по функциональности.

Рекомендованная конфигурация:

| Компонент             |     Провайдер |                                      Конфигурация |        Стоимость |
| --------------------- | ------------: | ------------------------------------------------: | ---------------: |
| VM под Convex backend | Timeweb Cloud | Cloud MSK 80: 4 vCPU, 8 GB RAM, 80 GB NVMe + IPv4 |     ~1 980 ₽/мес |
| PostgreSQL            | Timeweb Cloud |                               Cloud DB 2 / 4 / 40 |     ~1 580 ₽/мес |
| S3-хранилище          | Timeweb Cloud |                                            100 GB |       ~349 ₽/мес |
| Итого                 |               |                                                   | **~3 909 ₽/мес** |

Для proof of concept можно держать PostgreSQL на той же VM и уложиться примерно в `2 059–2 329 ₽/мес`, но для финансового SaaS это слабее по отказоустойчивости и обслуживанию.

## Что Меняется

Сейчас:

```text
Пользователь в РФ
  -> finly-app.ru на Vercel
  -> Convex Cloud за пределами РФ
  -> задержки, отваливающиеся запросы, зависание без VPN
```

После переноса:

```text
Пользователь в РФ
  -> finly-app.ru
  -> NEXT_PUBLIC_CONVEX_URL=https://server.finly-app.ru
  -> self-hosted Convex в РФ
  -> PostgreSQL в РФ
  -> S3-совместимое хранилище в РФ
```

Next.js-приложение можно оставить на текущем хостинге на первом этапе. Критически важно перенести именно Convex endpoint, потому что на него завязаны данные, realtime, действия синхронизации WB/Ozon, auth и crons.

## Что Сохраняется

- Convex schema и индексы из `convex/schema.ts`.
- Серверные функции `query`, `mutation`, `action`, `internalQuery`, `internalMutation`, `internalAction`.
- Клиентская модель `useQuery`, `useMutation`, `useAction`.
- Основная логика синхронизации WB/Ozon.
- Cron-задачи в `convex/crons.ts`.
- Модель организаций, команд, приглашений и ролей.
- Дашборды, аналитика, финансовые страницы и unit economics.

## Что Нужно Проверить Отдельно

Главный риск — auth-контур. Finly использует `@convex-dev/auth`, password-provider, email verification, reset password и server-side middleware. Перед переносом production-данных нужен отдельный smoke-test:

- регистрация;
- вход;
- выход;
- восстановление пароля;
- подтверждение почты;
- приглашение в организацию;
- middleware-защита приватных страниц;
- сессии после перезагрузки страницы;
- работа `ConvexAuthNextjsProvider`.

Если auth в self-hosted окружении потребует ручной настройки, это не ломает стратегию, но добавляет отдельный этап перед импортом production-данных.

## Плюсы Варианта

| Плюс                       | Что это дает Finly                                                             |
| -------------------------- | ------------------------------------------------------------------------------ |
| Минимум переписывания      | Сохраняем Convex API и большую часть кода.                                     |
| Работает в РФ без VPN      | Убираем текущий блокер для клиентов.                                           |
| Быстрее до production      | Перенос инфраструктуры быстрее, чем миграция на другую backend-модель.         |
| Предсказуемая стоимость    | Стартовый production-контур — около `3 909 ₽/мес`.                             |
| Российское хранение данных | PostgreSQL и S3 можно держать в российском дата-центре.                        |
| Сохраняются crons/actions  | Не нужно заново строить воркеры, очереди и расписания с нуля.                  |
| Ниже риск регрессий UI     | Клиентские хуки остаются прежними, меняются env-переменные и backend endpoint. |

## Минусы И Риски

| Минус                                              | Что делать                                                         |
| -------------------------------------------------- | ------------------------------------------------------------------ |
| Появляется DevOps-ответственность                  | Нужны мониторинг, backup, обновления Convex backend, alerting.     |
| Self-hosted Convex моложе Convex Cloud             | Начать с staging и прогнать полный сценарий на копии данных.       |
| Auth нужно проверять вручную                       | Выделить отдельный этап smoke-test до миграции клиентов.           |
| Надо обслуживать PostgreSQL                        | Для production брать управляемый PostgreSQL, а не БД на той же VM. |
| S3 нужен для модулей, файлов, импортов и экспортов | Создать отдельные buckets, доступы и lifecycle-политику.           |
| Нужен план отката                                  | Старый Convex Cloud не выключать до 3–7 дней стабильной работы.    |
| Производительность зависит от размера raw-данных   | Дожать агрегаты и убрать тяжелые `.collect()` с горячих страниц.   |

## Сравнение Стоимости

Срез цен — публичные страницы провайдеров на `2026-06-08`. Перед оплатой нужно перепроверить корзину в личном кабинете: облака меняют скидки, региональные коэффициенты и стоимость IPv4.

| Вариант                               | Состав                                                              |                   Оценка в месяц | Вывод                                                                    |
| ------------------------------------- | ------------------------------------------------------------------- | -------------------------------: | ------------------------------------------------------------------------ |
| **Timeweb Cloud, production**         | VM 4/8/80 + IPv4, PostgreSQL 2/4/40, S3 100 GB                      |                     **~3 909 ₽** | Рекомендованный вариант: дешево и достаточно надежно.                    |
| Timeweb Cloud, минимальный production | VM 4/8/80 + IPv4, PostgreSQL 1/2/20, S3 100 GB                      |                         ~3 119 ₽ | Можно начать, но меньше запас под sync и аналитику.                      |
| Timeweb Cloud, POC                    | VM 4/8/80 + IPv4, PostgreSQL на VM, S3 10–100 GB                    |                   ~2 059–2 329 ₽ | Хорошо для теста, слабее для клиентов.                                   |
| Timeweb Cloud, запас                  | VM 4/8/80 + IPv4, PostgreSQL 4/8/80, S3 250 GB                      |                         ~5 779 ₽ | Больше запас для роста и тяжелой аналитики.                              |
| Selectel, похожий production          | VDS 4/8/80, PostgreSQL 2 vCPU / 4 GB / 40 GB, object storage 100 GB |                   ~7 610–7 967 ₽ | Надежно, но почти в 2 раза дороже Timeweb.                               |
| Beget, экономичный VPS                | VPS 4 vCPU / 6 GB / 80 GB + IPv4, self-managed PostgreSQL           |         от ~2 190 ₽ без S3/DBaaS | Дешево, но это скорее POC или ручной production с большим обслуживанием. |
| Yandex Cloud                          | Managed PostgreSQL и Object Storage                                 | обычно дороже стартового Timeweb | Сильная платформа, но не самый выгодный старт.                           |
| Cloud.ru                              | PaaS PostgreSQL / S3                                                |       enterprise-уровень по цене | Рационально для крупных требований, не для текущего старта.              |

### Что Именно Заказать В Timeweb

1. `Cloud MSK 80` или аналог в Москве/Санкт-Петербурге: `4 vCPU`, `8 GB RAM`, `80 GB NVMe`.
2. Публичный IPv4 для VM.
3. Managed PostgreSQL: минимум `2 vCPU`, `4 GB RAM`, `40 GB`.
4. S3 Object Storage: стартово `100 GB`.
5. Домены:
   - `server.finly-app.ru` — self-hosted Convex backend;
   - `convex.finly-app.ru` — dashboard/admin endpoint, если разделяем домены;
   - `finly-app.ru` — основной frontend.

## Этапы Внедрения

### Этап 0. Подготовка

Цель: понять, что именно переносим, и не трогать production до готового staging.

- зафиксировать текущие env-переменные Convex и Vercel;
- выгрузить список таблиц, функций, crons и auth-сценариев;
- описать rollback: вернуть `NEXT_PUBLIC_CONVEX_URL` и `CONVEX_SERVER_URL` на Convex Cloud;
- выбрать окно миграции с минимальной активностью клиентов.

Результат: чеклист миграции и точка отката готовы.

### Этап 1. Инфраструктура В РФ

Цель: поднять пустой self-hosted Convex.

- создать VM;
- создать управляемый PostgreSQL в том же регионе;
- создать S3 buckets для файлов, модулей, экспортов, импортов и поиска;
- поставить Docker / Docker Compose;
- настроить reverse proxy и TLS;
- выпустить admin key для Convex;
- проверить health endpoint и dashboard.

Результат: `https://server.finly-app.ru` отвечает, пустой Convex готов принимать deploy.

### Этап 2. Staging-Перенос

Цель: проверить код Finly на self-hosted Convex без риска для клиентов.

- поднять staging endpoint, например `staging-server.finly-app.ru`;
- выполнить `npx convex deploy` в self-hosted окружение;
- импортировать обезличенную или свежую тестовую копию данных;
- подключить локальный Next.js к staging Convex;
- пройти smoke-test auth, магазинов, дашбордов, финансов, команд и синхронизаций.

Результат: команда видит, что продукт работает на российском Convex endpoint.

### Этап 3. Production-Миграция Данных

Цель: перенести данные без потери функциональности.

Базовый сценарий:

```bash
npx convex export --prod --include-file-storage --path ./finly-convex-backup.zip
```

Затем импорт в self-hosted Convex:

```bash
CONVEX_SELF_HOSTED_URL=https://server.finly-app.ru \
CONVEX_SELF_HOSTED_ADMIN_KEY=... \
npx convex import --replace-all ./finly-convex-backup.zip
```

Важно:

- на время финального экспорта остановить пользовательские изменения или назначить короткое maintenance window;
- после импорта сверить количество документов по ключевым таблицам;
- проверить файлы и storage, если они используются;
- старую базу не удалять.

Результат: production-данные находятся в self-hosted Convex.

### Этап 4. Переключение Frontend

Цель: направить пользователей на новый backend.

Обновить env:

```env
NEXT_PUBLIC_CONVEX_URL=https://server.finly-app.ru
CONVEX_SERVER_URL=https://server.finly-app.ru
CONVEX_SITE_URL=https://server.finly-app.ru
```

Затем:

- пересобрать и задеплоить frontend;
- проверить `finly-app.ru` без VPN из РФ;
- проверить login/logout и защищенные страницы;
- вручную запустить sync для тестового магазина;
- убедиться, что crons работают по расписанию.

Результат: пользователи работают с российским Convex endpoint.

### Этап 5. Стабилизация

Цель: не просто переехать, а сделать систему наблюдаемой.

- настроить backup PostgreSQL;
- настроить backup S3 buckets;
- добавить мониторинг VM, PostgreSQL, disk, memory, CPU, HTTP 5xx;
- добавить алерты по sync failures;
- вести журнал инцидентов первые 7 дней;
- старый Convex Cloud держать как fallback минимум 3–7 дней.

Результат: production работает без VPN и с понятным планом обслуживания.

## Оптимизация До И После Переноса

Перенос решает доступность, но не отменяет оптимизацию. Для Finly особенно важны:

- дневные агрегаты для дашборда и аналитики;
- отказ от больших `.collect()` на горячих страницах;
- пагинация raw-таблиц `financials`, `orders`, `sales`, `stocks`;
- очереди и лимиты для WB/Ozon sync;
- архивирование raw-данных старше `12–18` месяцев;
- usage dashboard по клиентам: документы, storage, action runtime, sync errors;
- шифрование marketplace API keys.

Минимум до публичного масштабирования: проверить dashboard, financials и analytics на магазинах с крупной историей.

## Критерии Готовности

Переезд можно считать готовым, когда:

- `finly-app.ru` открывается из РФ без VPN;
- вход, регистрация, reset password и verify email работают;
- список магазинов и активная организация подтягиваются корректно;
- ручной sync WB/Ozon завершается без ошибок;
- crons отрабатывают хотя бы один полный цикл;
- ключевые страницы открываются без 5xx: dashboard, products, financials, analytics, settings;
- количество документов в основных таблицах совпадает с экспортом;
- backup PostgreSQL и S3 настроен;
- есть понятный rollback на старый Convex Cloud.

## Что Не Делаем В Первой Итерации

- Не переписываем backend на Supabase.
- Не переносим все на чистый PostgreSQL и Next API.
- Не меняем модель авторизации без необходимости.
- Не оптимизируем весь sync-контур до идеального состояния.
- Не выключаем старый Convex Cloud сразу после переключения.

## Почему Не Supabase Сейчас

Supabase self-hosted можно развернуть в РФ, но для Finly это почти новый backend:

- нужно переписать Convex functions в API/routes/edge-functions/workers;
- realtime-поведение придется проектировать заново;
- crons и internal actions нужно заменить отдельными jobs;
- auth, роли, организации и инвайты придется переносить в Supabase Auth/RLS;
- фронтенд потеряет текущую модель `useQuery` / `useMutation`.

Supabase стоит рассматривать, если мы осознанно хотим уйти от Convex-архитектуры. Для текущей цели — работа без VPN и быстрый production-переезд — self-hosted Convex безопаснее.

## Источники

- Convex Self-Hosting: https://docs.convex.dev/self-hosting
- Convex Database Setup: https://get-convex-convex-backend.mintlify.app/self-hosting/database-setup
- Convex Storage Setup: https://get-convex-convex-backend.mintlify.app/self-hosting/storage
- Supabase Self-Hosting: https://supabase.com/docs/guides/self-hosting/docker
- Timeweb Cloud Servers: https://timeweb.cloud/services/cloud-servers
- Timeweb Cloud Pricing PDF: https://st.timeweb.com/cloud-static/timeweb-cloud-pricing-15-12-2025.pdf
- Timeweb S3: https://timeweb.cloud/services/s3-storage
- Timeweb S3 Tariffication: https://timeweb.cloud/docs/s3-storage/tariffication
- Selectel Prices: https://selectel.ru/prices/
- Selectel VDS Pricing PDF: https://vds.selectel.ru/pdf/pricing_actual_en.pdf
- Beget VPS: https://beget.com/ru/vps
- Beget Managed PostgreSQL: https://beget.com/en/cloud/dbaas-postgresql
- Yandex Managed PostgreSQL Pricing: https://yandex.cloud/en/docs/managed-postgresql/pricing
- Cloud.ru PostgreSQL Pricing: https://cloud.ru/docs/paas-postgresql/ug/topics/pricing
