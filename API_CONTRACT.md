# ALPHAMARKET — Контракт API (новый бэкенд на Next.js)

> Что это: список «дверей», через которые фронт нового приложения общается с сервером.
> Составлен на основе PROJECT_MAP.md + разбора `testsite/views.py` (21 700 строк), `apiviews.py` и `urls.py`.
> Дата: 2026-06-17. Источник истины для переноса логики 1:1.

---

## 0. Как читать этот документ (для владельца, простыми словами)

Старый проект — это 278 «адресов» (URL). Но это НЕ 278 разных функций. Они делятся на 4 типа:

1. **Страницы (PAGE)** — отдают целую HTML-страницу. В новом стеке это **React-страницы Next.js**, а не эндпоинты API. Им нужен не «контракт», а перенос вёрстки + запрос данных.
2. **Действия (ACTION)** — что-то меняют: создать заявку, закрыть сделку, подтвердить оплату. Это **POST-эндпоинты API**. Тут вся ценность и весь риск (деньги).
3. **Справочники/подсказки (LOOKUP)** — отдают список для выпадашек (города по стране, способы оплаты). Их в старом коде ~30 почти одинаковых — в новом API схлопываются в **~8 штук**.
4. **adminapi (DRF)** — готовый REST для админки p2p-admin. Переносим **как есть, 1:1**.

**Главный вывод:** реальных «умных» эндпоинтов с логикой — несколько десятков, а не двести. Остальное — страницы и повторы.

---

## 1. Общие правила (конвенции) — действуют для ВСЕХ эндпоинтов

| Тема | Старый код (как было) | Новый стек (как делаем) |
|---|---|---|
| **Авторизация** | Дырявая: роль берётся из глобального `cache['user_type']`, у большинства эндпоинтов вообще нет проверки входа. `send_message_modal` — `@csrf_exempt` без auth. | **Supabase Auth**, роль (`admin` / `client` / `pccntr`) в JWT-claims. Проверка входа + роли на КАЖДОМ эндпоинте. |
| **Деньги** | Float + `round()`. Часть операций без транзакций → гонки и двойные списания. | **Decimal** (Prisma `@db.Decimal`), `decimal.js` на вычислениях. Всё, что трогает деньги — в `prisma.$transaction` + блокировка строки (`SELECT … FOR UPDATE`). |
| **Округление** | Python `round()` = банковское (round-half-even). | Явный round-half-even хелпер, иначе паритет 1:1 не сойдётся. |
| **Статусы** | Строки/числа вперемешку: `'3'`,`'5'`,`'6'`,`'12'`,`'18'`, либо русское имя из таблицы `Status`. | Один **enum**, сгенерированный из выгрузки таблицы `Status` (имя → id). См. §7. |
| **Состояние между запросами** | Прячется в `cache['RequestInfo']`, `cache['orders_for_board']`, `cache['order_num']`, `session['selected_pccntr']` — глобально, НЕ по пользователю (баг). | Никакого скрытого состояния. Всё передаётся явно в теле запроса или берётся из auth. |
| **Формат ответа LOOKUP** | HTML-фрагменты `<option>`. | JSON-массивы `[{value, label}]`. |
| **Мутации через GET** | `Confirm/Decline_order_changes`, `close_deal` меняют данные по GET. | Только **POST** для любых изменений. |

---

## 2. Аутентификация и регистрация

> В новом стеке логин/OTP/сброс пароля даёт **Supabase Auth** из коробки. Эти эндпоинты в основном НЕ пишем сами — настраиваем Supabase. Но бизнес-привязку (профиль `Users`, тип пользователя) переносим.

| Новый эндпоинт | Метод | Тип | Что делает | Заменяет (старое) |
|---|---|---|---|---|
| Supabase signUp + `POST /api/profile/init` | POST | ACTION | Создать пользователя (неактивен), отправить email-OTP, завести профиль `Users` | `RegisterUser`, `register2` |
| Supabase verifyOtp | POST | ACTION | Подтвердить email-код, активировать | `email_confirm/<num>` (7 флоу → разнести!) |
| Supabase signIn | POST | ACTION | Вход | `LoginUser` |
| Supabase signOut | POST | ACTION | Выход | `LogOut` |
| Supabase reset password / email | POST | ACTION | Сброс пароля / смена email через OTP | `passwordreset`, `emailreset`, `emailresetcomplete` |
| `GET /api/check-user?username=` | GET | LOOKUP | Проверка занятости логина | `api_check_user` |
| `POST /api/account/role` | POST | ACTION | Выбрать тип (CLI/PART/ORG/EMPL), привязать обменник | `check_usertype`, `choose_exchange` |
| `DELETE /api/account/...` (несколько) | POST | ACTION | Откаты регистрации (удалить user/pccntr/org/exch/empl) | `delete_user`, `delete_user_2`, `delete_pccntr`, `delete_org`, `delete_exch`, `delete_empl`, `delete_exch_operations` |

**Внимание при переносе:** `email_confirm/<num>` в старом коде — это 7 разных сценариев в одном адресе (активация, сброс пароля, смена email…). Разнести на отдельные эндпоинты.

---

## 3. P2P-движок сделок — ФИНАНСОВОЕ ЯДРО 💰

> Это самая важная и самая опасная часть. Здесь деньги, резервы, гонки. Каждый ACTION ниже в новом коде ОБЯЗАТЕЛЬНО оборачивается в транзакцию с блокировкой строки.

### 3.1. Заявки, ордеры, офферы, доска объявлений

| Новый эндпоинт | Метод | Тип | Деньги/риск | Заменяет |
|---|---|---|---|---|
| `GET /api/p2p/board` | GET | PAGE-data | чтение | `P2Pmarket_Bulletin_board`, `P2Pmarket_Deal_board` |
| `POST /api/p2p/orders` | POST | ACTION | пишет `Orders` — **в транзакцию** | `..._Exchange_order_confirm`, `..._final` |
| `PUT /api/p2p/orders/{id}` | POST | ACTION | пишет Orders, удаляет Requests/DealReserve, **меняет `PCCNTR.Reserve`** — гонка! | `P2Pmarket_change_Exchange_order` |
| `POST /api/p2p/orders/{id}/offers` | POST | ACTION | **самая сложная логика** (9 этапов расчёта), пишет `Requests`, комиссии, профит | `P2Pmarket_Exchange_order_offer` |
| `GET /api/p2p/orders/{id}/offer-metrics` | GET | LOOKUP | расчёт профита/комиссии, **без записи** (8+ фолбэков курса, мостик через USDT) | `offer_metrics` |
| `POST /api/p2p/requests` | POST | ACTION | **создаёт Request + DealReserve, `PCCNTR.Reserve +=`** — гонка, обход лимита! | `P2Pmarket_Exchange_request` |
| `POST /api/p2p/requests/{id}/delete` | POST | ACTION | **`PCCNTR.Reserve -=`**, удаляет DealReserve — гонка | `..._Bulletin_board_delete_final` |
| `GET /api/p2p/requests/client/{orderId}` | GET | LOOKUP | офферы по ордеру клиента | `api_get_client_exchangerequests` |
| `GET /api/p2p/orders/client` | GET | LOOKUP | свои ордеры клиента | `api_get_client_orders` |

### 3.2. Комната сделки + чат

| Новый эндпоинт | Метод | Тип | Деньги/риск | Заменяет |
|---|---|---|---|---|
| `GET /api/p2p/deals/{id}/room` | GET | PAGE-data | чтение, сложная машина состояний UI | `room`, `room_closed_deal` |
| `POST /api/p2p/requests/{id}/accept` | POST | ACTION | создаёт Deal, отменяет конкурирующие заявки + снимает их резерв (`select_for_update` есть) | `chat` |
| `POST /api/p2p/deals/{id}/messages` | POST | ACTION | сообщение/файл (→ Supabase Realtime + таблица `deal_messages`) | `send_message`, `send_file` |
| `POST /api/p2p/deals/{id}/employee` | POST | ACTION | назначить курьера/сотрудника | `send_employee`, `send_delivery_type_employee`, `load_employee` |
| `POST /api/p2p/deals/{id}/time-interval` | POST | ACTION | временной интервал | `send_timeinterval` |

### 3.3. Платёжный диалог (подтверждение оплаты)

> Здесь модель резервов: **резерв при заявке → списание при подтверждении**. Часть эндпоинтов уже сделана правильно (atomic) — переносим их паттерн как образец.

| Новый эндпоинт | Метод | Деньги/риск | Заменяет |
|---|---|---|---|
| `POST /api/p2p/deals/{id}/client-pay-text` | POST | пишет `Balance`, `select_for_update` | `get_payment_text`, `client_get_recv_payment_text` |
| `POST /api/p2p/deals/{id}/client-confirm-balance` | POST | ✅ atomic (ОБРАЗЕЦ) | `client_confirm_payment_balance` |
| `POST /api/p2p/deals/{id}/pccntr-confirm` | POST | ✅ atomic (ОБРАЗЕЦ) | `pccntr_confirm_client_payment` |
| `POST /api/p2p/deals/{id}/client-confirm-recv` | POST | ✅ atomic (ОБРАЗЕЦ) | `client_confirm_recv_payment` |
| `POST /api/p2p/deals/{id}/pccntr-confirm-recv` | POST | `select_for_update` | `pccntr_confirm_recv_payment` |
| `POST /api/p2p/deals/{id}/start-payment` | POST | смена статуса + сумма | `pccntr_update_rate_and_start_payment`, `pccntr_update_rate_cash` |
| `POST /api/p2p/deals/{id}/cash-confirm` | POST | ✅ atomic | `client_confirm_cash_payment` |
| `POST /api/p2p/deals/{id}/client-paid` | POST | **`Client_Approved +=`, гонка** | `send_client_payment` |
| `POST /api/p2p/deals/{id}/pccntr-paid` | POST | **`ExchangePoint_Reserve→Approved`, гонка** | `send_pccntr_payment`, `pccntr_send_payment_notification` |
| `POST /api/p2p/deals/{id}/payment-notify` | POST | **пишет резервы, гонка** | `send_payment_notification`, `pccntr_change_deal_status_payment_notification`, `change_deal_status_payment_notification` |
| `POST /api/p2p/deals/{id}/check-limit` | POST | **`In_Daily_limit_used +=` без границ, гонка** | `count_pccntr_limit` |
| `POST /api/p2p/deals/{id}/cancel-data` | GET/POST | чтение/запись причины отмены | `get_cancel_data`, `write_cancel_type_to_deal` |

### 3.4. Закрытие сделки, апелляции, бонусы, отзывы

| Новый эндпоинт | Метод | Деньги/риск | Заменяет |
|---|---|---|---|
| `POST /api/p2p/deals/{id}/close` | POST | **`PCCNTR.Reserve -=`, статистика — гонка** | `close_deal`, `close_deal_after_agreement` |
| `POST /api/p2p/deals/{id}/settle` | POST | 🔴 **КРИТИЧНО: `PCCNTR.Balance/Reserve`, начисление бонусов, отрицательные Payments — двойное списание!** | `P2Pmarket_afterdeal_count_bonus_and_balance` |
| `POST /api/p2p/deals/{id}/appeal` | POST | создать/снять апелляцию (блокирует участников) | `create_appeal`, `change_deal_status_appeal`, `change_deal_status_delete_appeal` |
| `POST /api/p2p/deals/{id}/review` | POST | пересчёт рейтинга | `P2Pmarket_Review`, `P2Pmarket_Review_List` |

---

## 4. Балансы обменника (PCCNTR) 💰

| Новый эндпоинт | Метод | Деньги/риск | Заменяет |
|---|---|---|---|
| `POST /api/balance/deposit` | POST | создаёт **pending** Payments (ручная проверка админом), безопасно | `pccntr_balance_deposit`, `api_adm_mrkt_deposit_create` |
| `POST /api/balance/withdraw` | POST | ✅ atomic + lock (ОБРАЗЕЦ): `Reserve += amount`, проверка ≤ доступного | `pccntr_balance_withdrawal` |
| `GET /api/balance/history` | GET | чтение | `pccntr_balance_history` |
| `GET /api/balance` | GET | чтение балансов | `balance_settings` |
| ⚠️ `balance_settings_refill_balance` / `balance_settings_withdraw_funds` | — | 🔴 **НЕ переносить как есть**: прямое изменение баланса без проверки = дыра/фрод. Переделать через pending-заявку + подтверждение | — |

---

## 5. Профиль, обменники, филиалы, сотрудники, скидки

> В основном PAGE + CRUD. Деньги затрагивают только конфиг скидок/комиссий (это «цены», не движение средств).

- **Компании/транзакции (движок A, DealComp):** `companies_list`, `company_detail`, `transactions`, `transaction_detail`, и платёжный флоу `transaction_confirm/cancel/select_requisites/set_requisites/manual_requisites/submit_requisites/confirm_payment`. ⚠️ `confirm_payment` при частичной оплате **создаёт дочернюю сделку** — перенести аккуратно. Статусы строковые ('3','5','6','12','13','15','18').
- **Профиль обменника:** `profile_pccntr_*`, `profile_branch_*`, `profile_pccntr_staff_*`, `profile_pccntr_directions_*` + их `api/*`-помощники (next-code, create, branches, staff/search/add/delete/toggle, opername-lookup).
- **Скидки/бонусы (деньги-цены):** `profile_pccntr_alldiscount_*` (пишет `PCCNTR.Bonus`), `profile_pccntr_discount_*` (пишет `Users.bonusPercentFull` 0–50%). ⚠️ `PCCNTR.Bonus` хранится как строка-с-разделителями — **нормализовать в таблицу**.
- **Реквизиты:** `pccntr_paym_detail_*`, `client_paym_detail_*`, `create_*_paym_detail_from_modal`. Валидация лимитов `Daily ≤ Monthly ≤ All`.

---

## 6. Общие настройки — ДВИЖОК ЦЕН (комиссии/нормы прибыли/COGS) 💰🧮

> Многошаговые мастера. Сейчас состояние шагов лежит в глобальном `cache` — переделать на явные черновики/состояние на клиенте. Финансово важная зона: тут формируется курс.

- **Структура обменника:** `general_settings_change/new_exchange_structure_*` (мастера 1–3: организатор → точка → сотрудники).
- **Сделки обменника (нормы прибыли):** `general_settings_change/new/add/delete_exchange_deals_*` — тиры `Norm_Prib_*`, `Min/Max_amount`.
- **COGS + комиссия:** `general_settings_*_cogs_comission_*` (`commission_abs/percent_1-5`, `cog_1-5`).
- **Источник курса:** `general_settings_*_exchange_rate_source*` (`chosen_quote_1..5`).
- **Бонусы:** `general_settings_change_exchange_deals_bonus_2`.
- **Проверка/формула курса:** `general_settings_check_exchange_deals_rate*` + `..._formula`.
  → В новом стеке **`POST /api/rate/calculate`** — JSON-эндпоинт расчёта (вход явно, не из cache). Возвращает пошаговую деривацию: рыночная комиссия → конвертация → комиссия → COGS → норма прибыли, прямой и обратный расчёт.

---

## 7. Справочники и подсказки (LOOKUP) — схлопнуть ~30 → 8

| Новый эндпоинт | Параметры | Заменяет (сколько старых) |
|---|---|---|
| `GET /api/lookup/pay-types` | `currency`, `side=sell\|buy\|account` | `load_pay_type(_sell/_buy/_num...)` — 6 функций |
| `GET /api/lookup/fin-offices` | `currencySell,currencyBuy,paySell,payBuy,direction=from\|to` | все `load_payment_method*` + `p2p_*` — ~12 роутов. ⚠️ **взять только версию со строк 12678/12743** (первые определения 12161/12219 — мёртвый код, перекрыты) |
| `GET /api/lookup/countries` | — | `api_countries`, `api_get_country` — 2 |
| `GET /api/lookup/cities` | `country` | `load_cities`, `api_cities`, `api_get_city` — 3 |
| `GET /api/lookup/exchange-point-cities` | `country`, `exchangePoint`, `mode=all\|partner\|unregistered` | `load_cities_exchname/_pccntr/_pccntr_register/_num/_text` — 5 |
| `GET /api/lookup/employees` | `dealId`, `deliveryType` | `load_employee` — 1 |
| `GET /api/lookup/reference?type=gender\|status\|exchange-id` | `type` | `api_get_gender/status/exchangeid` — 3 |
| `POST /api/rate/calculate` | явные входные данные | формула курса (см. §6) |

**Все LOOKUP возвращают JSON `[{value, label}]`, не HTML.**
**Исключить из переноса:** `load_notifications` (захардкожен `@m`, debug-заглушка).

---

## 8. Уведомления

| Новый эндпоинт | Метод | Заменяет |
|---|---|---|
| `GET /api/notifications` | GET | `Notification` (страница), `api_get_notifications` |
| `POST /api/notifications/{id}/read` | POST | `mark_notification_read` |
| `POST /api/notifications/read-all` | POST | `mark_all_notifications_read` |
| `POST /api/notifications/order-changes/{id}/confirm` | POST | `Confirm_order_changes` (был GET!) |
| `POST /api/notifications/order-changes/{id}/decline` | POST | `Decline_order_changes` (был GET!, освобождает резерв) |

⚠️ Поле `Unread` в старой БД **инвертировано**: `False` = непрочитано, `True` = прочитано. Легко ошибиться при переносе.

---

## 9. adminapi (DRF) — переносим 1:1

> Готовый REST для админки p2p-admin. Префикс `/adminapi/v1/`. Auth: JWT в httpOnly-cookie `access_token` + роль staff/admin. **Пагинации НЕТ** — фронт ждёт плоские массивы (сохранить поведение).

**Только чтение (ListAPIView):** `orders`, `requests`, `pccntr`, `users_pccntr`, `pccntr_exchp`, `pccntr_opertypes`, `exchangeid`, `ep_exchangeid`, `curr_cogs_commission`, `companiesactive` (limit 100), `subactive`, `groupactive`, `blacklist`, `companiesbalance`.

**CRUD (ModelViewSet):** `usersmodel`, `authusermodel`, `contacttypes`, `countries`, `cities` (фильтр `?country=`), `deals`, `paym_detail`, `paymdetailscomp`, `market_commission`, `companydealsstatuslogs`, `market_payment_details`, `fin_offices` (только crypto, без BTC), `chats`, `messages`.

**Со спец-логикой (не чистый CRUD) — перенести внимательно:**
- `payments` + actions `{id}/confirm`, `{id}/reject`, `{id}/aml_check` (AML — заглушка 501). Confirm требует `TxHash` для вывода; меняет статус План→Выполнено/Отменён; шлёт уведомление. **atomic.**
- `appealsmodel` + `{id}/take-to-work`, `unblock-expired`. При создании апелляции **блокирует участников**; при решении (`status=Done`) применяет вердикт (`apply_appeal_decision`: «Ничья»/блок пользователя/обменника, временно/постоянно). Логика блокировок — `get_temporary_block_until` (5/15/30 дней по счётчику).
- `notifications` (CRUD).
- `company_commission` — **синглтон** (один объект pk=1, GET возвращает его, POST обновляет, DELETE запрещён).
- `support-tickets` (FeedBack) + actions `send_message`, `take_in_work`, `close_ticket`, `reopen`, `set_status`. Фильтры status/priority/user/unread/assigned_admin.
- `transaction/` (DealComp) — POST принимает **массив ИЛИ объект**; при создании пишет `Company_Deals_Status_Logs`; конвертирует статус имя↔id. **atomic.**

**Auth-эндпоинты:** `token/` (логин, кладёт JWT в httpOnly-cookie), `token/refresh/`, `token/verify/`. В новом стеке заменяются на Supabase Auth для админ-роли (или сохранить JWT-мостик, если p2p-admin не трогаем).

---

## 10. Приоритеты реализации (порядок переноса)

1. **Справочники (§7)** — простые, разблокируют формы. Начать отсюда.
2. **adminapi (§9)** — уже спроектирован, переносится механически. Параллельно.
3. **Профиль/настройки цен (§5–6)** — конфиг до сделок.
4. **P2P-ядро (§3) + балансы (§4)** — последними и с максимальной осторожностью: каждый эндпоинт = транзакция + блокировка + тест паритета против эталона Python.

### 🔴 Список «обязательно в транзакцию + блокировку» (риск денег):
`settle` (двойное списание бонусов), `refill/withdraw_funds` (овердрафт/фрод), `requests` + `requests/delete` + `close` (резерв PCCNTR), `client-paid`/`pccntr-paid`/`payment-notify`/`check-limit` (поля Approved/Reserve/лимит), создание ордеров/офферов.

### ✅ Образцы для копирования (уже сделаны правильно):
`client-confirm-balance`, `pccntr-confirm`, `client-confirm-recv`, `cash-confirm`, `balance/withdraw` — паттерн `transaction.atomic` + `select_for_update`. Модель резервов: «резерв при заявке → списание при подтверждении».
