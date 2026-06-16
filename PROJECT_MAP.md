# ALPHAMARKET — Карта старого проекта (результат разбора агентами)

> Источник истины для переписывания на TypeScript. Составлено 2026-06-16 пятью research-агентами.

## 📊 Масштаб (поверхность миграции)
- **~95 HTML-страниц** (Telegram WebApp, server-rendered) — переписываем в React/Next.js
- **~185 JSON-эндпоинтов** (функции-вьюхи) + **~40 DRF** (adminapi) — переписываем в API Next.js
- **58 моделей** / 74 таблицы в БД
- `views.py` — 21 700 строк бизнес-логики

---

## 🔴 КРИТИЧНЫЕ ОТКРЫТИЯ (меняют план)

### 1. Дамп БД — это SQLite, а НЕ MySQL
`MarketDatabase.sql` содержит `PRIMARY KEY AUTOINCREMENT`, `DEFERRABLE`, `BOOL`, `REAL` — признаки SQLite. Важно: SQLite не хранит точность Decimal в DDL → **истинные типы денег брать из `models.py`**, а не из дампа. Миграция: SQLite → PostgreSQL (Supabase).

### 2. Деньги в `Deals` хранятся как FloatField (плавающая точка!)
В модели `Deals` (сделки физлиц) суммы/курсы — `FloatField`. А в `DealComp`/`Balance` те же сущности — корректный `Decimal`. **Несогласованность.** При переносе ВСЕ деньги → `Decimal` (Prisma `@db.Decimal(p,s)`), точность из models.py. Иначе ошибки округления на реальных деньгах.

### 3. ДВА разных движка сделок — не путать
- **Движок A «компанейский» (`DealComp`)**: статусы — строки-числа (`'5'`,`'6'`,`'12'`,`'18'`...) захардкожены. Логи `Company_Deals_Status_Logs`. Деньги в Decimal.
- **Движок B «P2P» (`Deals`/`Requests`/`Orders`)**: статусы по русскому имени из таблицы `Status` (`Status.objects.get(Status_Name='Выполнено')`). Логи `Deal/Request/Order_Status_Logs`. Балансы `Balance`, резервы `DealReserve`. Основной движок. Деньги во Float.
- **Для переноса 1:1 нужно выгрузить таблицу `Status` (имя→pk) как enum.**

### 4. Авторизации через Telegram по факту НЕТ
Telegram WebApp — просто обёртка-браузер. Никакой проверки `initData`/hmac нет. Реальный вход — классический Django: **username/пароль + код подтверждения на email (OTP)**. JWT-токены — только для админки (доступ при `is_staff=True`, payload декодируется БЕЗ проверки подписи — дыра). Bot-токен захардкожен в `bot.py`.
→ Для нового стека: Supabase Auth даёт email-OTP из коробки. Telegram-валидацию (если нужна) строить с нуля. Роли: `is_staff`→admin, клиент→client.

### 5. Деньги местами БЕЗ транзакций БД (race conditions)
НЕ atomic (риск): `close_deal`, `close_deal_after_agreement`, `P2Pmarket_afterdeal_count_bonus_and_balance`, создание Request+DealReserve+Reserve в `P2Pmarket_Exchange_request`.
Образцовые (atomic + select_for_update): `chat`, `pccntr_balance_withdrawal`, `client_confirm_payment_balance`, `pccntr_confirm_client_payment`.
→ В TS ВСЁ money-touching заворачивать в транзакцию обязательно. Резерв `PCCNTR.Reserve` мутируется в ~14 местах → централизовать в один сервис.

---

## 🧩 Домены бизнес-логики (views.py)

| Домен | Ключевые функции (строки) | Риск |
|---|---|---|
| **Сделки B (P2P)** | `chat`(18241, ЯДРО, atomic), `close_deal`(19828), `close_deal_after_agreement`(19895), `P2Pmarket_afterdeal_count_bonus_and_balance`(20668), `P2Pmarket_change_Exchange_order`(13160, ~1100 строк) | 💰💰💰 |
| **Сделки A (DealComp)** | `confirm_payment`(1328, split при частичной оплате), `transaction_cancel`(1239), `transaction_confirm`(1490) | 💰💰 |
| **Комиссии** (3 слоя: market/COGS/норма прибыли) | `_build_calc_for_order`(5672), `count_exchange_details_send_amount`(17177), `count_exchange_details_receive_amount`(17438), формульная песочница(5952) | 🧮💰 |
| **Балансы/резервы** | `pccntr_balance_deposit`(9220), `pccntr_balance_withdrawal`(10218, образец), `count_pccntr_limit`(20432) | 💰💰 |
| **Заявки/ордера/офферы** | `P2Pmarket_Exchange_request`(17720), `P2Pmarket_Exchange_order_offer`(15639, ~1200 строк, самая сложная), `P2Pmarket_Exchange_order_final`(13003) | 💰🧮 |
| **Апелляции** | `create_appeal`(19937), `change_deal_status_appeal`(19951), `change_deal_status_delete_appeal`(19964) | 💰(косв.) |
| **Курсы валют** | `_get_send_to_receive_rate`(5607), `update_deal_floating_price`(5871, плавающая цена), `recalc_deal_actuals`(5936) | 🌐🧮 |
| **Чат сделки + платёжный диалог** | `room`(18650), `send_message`(19703), `client_confirm_payment_balance`(20203, atomic), `pccntr_confirm_client_payment`(20305, atomic) | 💰🔒 |

**Состояние сессии через Django `cache`** (`user_type`, `ExchangeName`, `chosen_ep_deals`) — заменить нормальной сессией/контекстом. Внешних API нет, только SMTP. TxHash вводится вручную, подтверждается админом.

---

## 🗄️ Данные — денежные таблицы (для Prisma)

| Таблица | Деньги | FK | Заметки |
|---|---|---|---|
| `Deals` (`testsite_deals`) | **Float (→Decimal!)** | нет (связи строковые/int) | Сделки физлиц |
| `DealComp` (`deals_comp`) | Decimal(20,8) ✅ | `parent_deal→self` | Сделки юрлиц, split |
| `Balance` (`testsite_balance`) | Decimal ✅ | `deal→Deals`, реквизиты | Единственная таблица с реальными FK на сделку |
| `Transaction` | Decimal(12,2) | `sender/receiver→PretragaComp` | Транзакции компаний |
| `DealReserve` | **Float (→Decimal!)** | нет | Резервы под сделки |
| `Payments` | Decimal(18,8) | `Status→Status` | Журнал депозит/вывод |
| `ADM_MRKT_COMM` | **Float** | нет | Комиссия маркета |
| `ADM_COMP_COMM` | Decimal | нет | 3 интервала комиссий |
| `CURR_COGS_COMMISSION` | **Float** | строковые | COGS обменника |

**Особенности схемы для Prisma:**
- Почти все связи **строковые/числовые, БЕЗ FK** (`TG_Contact`, `PCCNTR`, `ExchangePointID`, числовые `DealID/OrderID/RequestID`). Целостность не гарантирована БД → при добавлении настоящих FK чистить orphan-записи.
- Кастомные имена таблиц (`@@map`): `ADM_COMP_COMM`, `deals_comp`, `TRNSFRTYPE`, `PAYM_DETAILS_COMP`, `pretraga_companies`.
- Имена колонок с пробелами/кириллицей в `TransferType`: `"Код элемента"`(PK), `"ACCOUNT_PL REVENUE"`, `"ACCOUNT_PL COGS"` → `@map`.
- Псевдо-JSON в TextField: `Users.Devices`, `PretragaComp.selected_*_codes` → `jsonb`.
- `BigIntegerField` как timestamp: `Balance.completedAt` (Unix-ms).
- Разнотипные PK: `Users`=UUID, `PretragaComp`=reg_broj(str), остальные=int.
- PascalCase/SCREAMING_CASE поля везде → массовый `@map` для camelCase в Prisma.

---

## 🔌 Существующий adminapi (референс для нового API)
- Префикс `/adminapi/v1/`, ~40 эндпоинтов. **Пагинации НЕТ** — фронт ждёт плоские массивы.
- Авторизация: JWT в httpOnly cookie `access_token` + `IsAdminUser` (staff). `withCredentials:true`.
- CRUD через ModelViewSet (`deals`, `payments`, `usersmodel`, `appealsmodel`, `notifications`, `support-tickets`, комиссии...).
- Кастомные actions с логикой (не чистый CRUD): `payments/{id}/confirm|reject`, `appealsmodel/{id}/take-to-work` + `unblock-expired`, `support-tickets/{id}/send_message|close_ticket|...`.
- Singleton: `company_commission` (один объект). `transaction/` POST принимает массив или объект.
- Статусы — ID в справочнике `Status`, сериализаторы делают reverse-lookup в имя.

---

## ⚙️ Инфраструктура → новый стек

| Подсистема | Факт | Перенос |
|---|---|---|
| **Фоновые задачи** | Реальная одна — `update_currency`. Beat-расписание сломано (бьёт в несуществующую `rates.tasks.update_currency_rates`). Парсеры лёгкие на `requests` (TradingView, Bybit/Binance P2P, Rapira, BestChange), без captcha/selenium. TwoCaptcha — мёртвая зависимость. | **Vercel Cron** `*/5 * * * *`. НО полный цикл по всем источникам может превысить лимит времени → батчить/параллелить (`Promise.all`) или отдельный воркер. |
| **Realtime** | Только чат сделки. `ChatConsumer`, группа `chat_<dealId>`, **stateless** (не пишет в БД). Нет канала уведомлений в WS. | **Supabase Realtime** + таблица `deal_messages` (история «бесплатно») + RLS по участникам. |
| **Auth** | Django session + email-OTP; JWT только админ. Нет Telegram-валидации. | **Supabase Auth** (email-OTP из коробки); роли admin/client; Telegram-bridge при необходимости — Edge Function. |

---

## 🔁 ОБНОВЛЕНИЕ ПОСЛЕ ЧИСТКИ (2026-06-16, второй проход)

**Что изменилось:** удалены `node_modules` (мусор). Исходный код НЕ тронут (views.py 21700, models.py 2136 и пр. — байт в байт). Появился новый файл `services/rate_calculator.py` (107 строк). PROJECT_MAP актуальна.

**🗑️ Мусор на удаление при миграции:** `celerybeat-schedule.*` (4 файла), `dump.rdb`, файл `delete files` (содержит SQL `delete from testsite_messages where MessageType="File"`), `testsite/package.json` + `package-lock.json`, `testsite/users.ini`.

### 📋 forms.py (2920 строк) — ранее не разбирали. 33 формы.
Настоящих Django-валидаторов всего **2**: `RegisterUserForm.clean_email` (уникальность email), `GeneralDiscountForm.clean` (4 правила непересечения интервалов скидок). Остальная валидация — атрибуты полей + динамические каскады `choices` в `__init__` (страна→город, валюта→тип перевода→финофис).
**⚠️ Денежные бизнес-правила, которые НЕ проверяются на бэке (закодировать в zod на фронте):**
- Иерархия лимитов реквизитов: `Daily ≤ Monthly ≤ All` (×In/Out) — формы `PaymentDetailNew`/`ChangePaymentDetail`/клиентские.
- Интервалы норм прибыли (`ChooseDeals*`, `ChangeDealInfo`): `Min ≤ Max`, монотонность границ, «если интервал начат — все 3 поля обязательны».
- `Exchangeorder`: `Send_or_Receive` определяет смысл `Order_amount`; суммы > 0.
- `RefillBalance`/`WithdrawBalance`: сумма > 0; вывод ≤ доступного.
- `Feedback.TG_contact`: условная обязательность (`param ∈ {2,4,38}`).
- Проценты — Decimal 0..100, 2 знака; суммы — целые.

### 🧮 services/rate_calculator.py + parsers/ — движок курсов
- **`RateCalc` импортируется (views.py:54), но `calculate_rate()` НИГДЕ не вызывается.** Боевой расчёт сумм/норм прибыли — **инлайн в views.py** (~13870–14035, 17816–17820) поверх готового `Currency_source.Value`. → переносить НАДО инлайн-логику (она каноническая); RateCalc — вторая реализация той же формулы, решить какая канон.
- **Баги в RateCalc** (перенести «как есть» с пометкой или починить): ветка `send` (1 OperType) делает `rate = full * amount` (вероятно должно быть `/`); опечатка ветки 2 (проверяет record_1, читает record_2).
- **Деньги везде float + `round(x,2)`.** ⚠️ Python round = banker's rounding (round-half-even), JS Math.round/toFixed = half-up → для паритета 1:1 нужен явный round-half-even хелпер или `decimal.js`.
- **Парсеры (5):** синхронный `requests`, контракт `get_rate(config) -> float|None`, ошибки глотаются в None. Источники: TradingView (scanner POST), Bybit P2P (api2, захардкожен userId), Binance P2P (bapi, `time.sleep(random 1-3с)` + антибот), Rapira (open API, только USDT/RUB), **BestChange = HTML-scraping** (самый хрупкий). Фабрика `parser_factory` — map с опечатками в ключах (перенести дословно).
- **Vercel Cron:** последовательный обход ВСЕХ источников (Binance/BestChange по 2-4с) = минуты → НЕ влезет в лимит serverless. Решение: батчить / функция-на-источник / очередь. Переписать на async `fetch` + пул (`p-limit`) + AbortController timeout. Риск: Bybit/Binance/BestChange могут гео-блокировать Vercel US IP.
