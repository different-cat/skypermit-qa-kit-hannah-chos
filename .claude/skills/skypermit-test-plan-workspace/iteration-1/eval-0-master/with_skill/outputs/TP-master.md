# TP-master: Master тест-план SkyPermit — перший реліз

| Поле | Значення |
|---|---|
| ID | `TP-master` |
| Тип | Master |
| Версія | 0.1 |
| Дата | 2026-10-20 |
| Автор | згенеровано Claude (скіл `skypermit-test-plan`) |
| Джерела | `../skypermit/docs/` `01`–`08` (`08` — лише допоміжно); `../skypermit` @ `78407f1` (2026-09-25) |

Скорочення у колонці «Джерело»: `01` = `01-data-model.md`, `02` = `02-state-machine.md`, `03` = `03-api-contract.md`, `04` = `04-frontend-spec.md`, `05` = `05-open-questions.md`, `06` = `06-business-rules.md`, `07` = `07-clarifications.md`, `08` = `08-uml-diagrams.md`.

## 1. Мета

Визначити повний обсяг мануального тестування SkyPermit перед першим релізом: які сценарії можна виконати вже зараз (`Ready`), що заблоковано відкритими питаннями з `05`, і що потребує уточнень власника docs. За цим планом приймається рішення go/no-go для релізу. Головний ризик — система видає дозвіл на політ, якого за правилами бути не повинно (або відмовляє безпідставно).

## 2. Обсяг

### 2.1 В обсязі

Усі області з карти покриття:

| AREA | Що | Ендпоінти / сторінки |
|---|---|---|
| `AUTH` | реєстрація, логін/логаут, `me`, `401`/`403`, reviewer-or-owner reads, `X-Reviewer-Id` | `POST /operators`, `/auth/login`, `/auth/logout`, `GET /auth/me`, `GET /operators/{id}` |
| `REG` | пілоти й дрони, похідні currency / weight band, ліміт 30 000 г | `POST/GET /pilots`, `GET /pilots/{id}`, `POST/GET /aircraft`, `GET /aircraft/{id}` |
| `ZREF` | довідник зон | `GET /zones`, `GET /zones/{code}` |
| `FR` | створення `DRAFT`, валідація тіла, список і фільтри | `POST /flight-requests`, `GET /flight-requests`, `GET /flight-requests/{id}` |
| `SM` | усі легальні й нелегальні переходи (`409`) | submit / approve / reject / cancel / complete |
| `GP`, `GA`, `GZ`, `GT` | чотири гейти | `.../submit`, `.../validate` |
| `PIPE` | порядок гейтів, first failure, тіло `422`, збереження `failing_gate`/`rule_reference` | `.../submit`, `.../validate` |
| `VAL` | dry-run без зміни стану | `POST /flight-requests/{id}/validate` |
| `REV` | approve/reject, `ReviewAction` | `.../approve`, `.../reject` |
| `CAN` | скасування і дедлайн 2 год | `.../cancel` |
| `CMP` | `COMPLETED` тільки через maintenance endpoint | maintenance endpoint (див. `F-13`) |
| `AUD` | `AuditLog` append-only | (немає API — див. `F-22`) |
| `SUSP` | оператор `SUSPENDED` | create / submit / approve |
| `UIO` | портал оператора | `/login`, `/register`, реєстри, форма заявки, «Check before submitting», «My requests», деталі |
| `UIR` | черга рев'юера | таблиця `UNDER_REVIEW`, деталі, approve/reject |
| `TID` | контракт `data-testid` | усі інтерактивні елементи |

### 2.2 Поза обсягом

| Що | Чому |
|---|---|
| Автотести, навантаження, продуктивність, безпека (pentest) | план для мануального функціонального тестування |
| Створення/редагування зон через продукт | API і UI для цього немає (`03` Reference data; `01` Zone) — перевіряємо лише відсутність |
| Підвищення `authorisation_level`, зміна `Operator.status`/`Pilot.status`/`Aircraft.status`, деактивація зони | out-of-band адмін-дії (`03` Auth; `01`) — використовуються лише як підготовка даних |
| Скасування з `SUBMITTED` (`02`, рядок `SUBMITTED` → cancel) | стан транзитний, «caller never observes `SUBMITTED` at rest» (`02`) — мануально недосяжний без спецзасобів (інструментування/затримки в pipeline). Зафіксовано як залишковий ризик |
| Значення з `OH-1` v1.4 і `CR-1` як очікувані | superseded / не прийнятий (`06` вступ, §2, §3) — використовуються тільки в регресійних сценаріях «НЕ застосовується» |
| Supabase-міграції, тригер аудиту на рівні БД як окремий об'єкт | внутрішня реалізація (`02` Implementation note); перевіряємо лише спостережуваний результат у `AuditLog` |
| Візуальний дизайн | `04`: «Styling is intentionally minimal» |

## 3. Об'єкт тестування

- Сутності: `Operator`, `Pilot`, `Aircraft`, `Zone`, `FlightRequest`, `ReviewAction`, `AuditLog` (`01`). Похідні, не збережені: currency пілота, weight band дрона (`01` Derived).
- Стани `FlightRequest`: `DRAFT`, `SUBMITTED` (транзитний), `UNDER_REVIEW`, `APPROVED`, `REJECTED`, `CANCELLED`, `COMPLETED` (`02`).
- Гейти: `PILOT` → `AIRCRAFT` → `ZONE` → `TIME_WINDOW`, first failure (`06` Evaluation order; `03` Gate pipeline).
- API: 16 нумерованих ендпоінтів + 4 auth + розширення `GET /pilots`, `GET /aircraft` (`03`).
- UI: портал оператора і черга рев'юера, перемикач ролей у хедері (`04`).

## 4. Підхід

**Рівні / типи.**
- API-функціональне тестування мануально через REST-клієнт (Postman/Bruno/curl): сесійна cookie оператора, заголовок `X-Reviewer-Id` для рев'юера. Основний рівень для гейтів, state machine і кодів помилок — точні значення `failing_gate`/`rule_reference` видно лише в тілі відповіді.
- UI-тестування двох порталів (`04`) — флоу, відображення результатів гейтів, причини відмови, `data-testid`.
- Наскрізні сценарії (E2E): реєстрація → пілот/дрон → заявка → validate → submit → черга → approve/reject → cancel.
- Перевірка побічних ефектів: після кожної дії — `GET /flight-requests/{id}` (стан, `failing_gate`, `rule_reference`, `rejection_reason`, `submitted_at`, `decided_at`, `decided_by`) + read-only запит до БД стенду для `AuditLog` і `ReviewAction` (API для читання цих таблиць немає — `F-22`).

**Техніки тест-дизайну.**

| Техніка | Де |
|---|---|
| State transition | `SM`: 10 легальних рядків таблиці `02` + повна матриця 6 спостережуваних станів × 5 дій для нелегальних пар → `409` |
| Decision table | `GP` рейтинг × `zone_class` (3×3); `GZ` `authorisation_level` × `required_authorisation_level` (3×3); `GA` weight band × `permitted_bands` по 6 зонах |
| Boundary value analysis | маса (900/901, 3 500/3 501, 12 000/12 001, 30 000/30 001 г); стеля (ceiling / ceiling+1 м); тривалість (14/15/240/241 хв); lead time (6 год ± Δ); advance (60 днів ± Δ); currency (18 міс. ± 1 день); дедлайн скасування (2 год ± Δ); `flight_purpose` (2 000/2 001 символ); overlap (дотик меж vs 1 хв перетину) |
| Equivalence partitioning | класи станів для overlap (враховуються `APPROVED`/`UNDER_REVIEW`; ігноруються `DRAFT`/`REJECTED`/`CANCELLED`/`COMPLETED`); зони з/без curfew; ролі (анонім / власник / чужий оператор / рев'юер) |
| Pairwise | порядок гейтів: усі 6 пар гейтів + «падають усі чотири» |
| Negative | `400`, `401`, `403`, `404`, `409`; спроби update/delete `AuditLog`; спроби змінити зони |
| Регресія проти superseded значень | `OH-1` v1.4 (24 міс., межа C1/C2 = 4 000 г), базовий curfew `ZC-2026.1` для `KBP-CTR-3`/`ODS-CTR-5` |

**Примітка щодо curfew.** Повне «накриття» curfew вікном із кінцями поза ним неможливе: найкоротший curfew у каталозі — 6 год (`A-7` §1), а максимальна тривалість — 240 хв (`TSD-4` §3), і проміжок між двома curfew-періодами (≥ 16 год) теж довший за 240 хв. Тому вимогу «minute-by-minute» перевіряємо частковими перетинами на початку і в кінці вікна.

**Примітка щодо часових поясів.** Часовий пояс зон у docs не визначено (`F-5`). Сценарії curfew, у яких вікно лежить у curfew за **будь-якого** зміщення UTC+0…UTC+3, позначено `Ready`; граничні — `Needs clarification (F-5)`.

## 5. Тестові сценарії та матриця покриття

Пріоритети: **P1** — неправильний дозвіл/відмова на політ, порушення доступу, втрата аудиту; **P2** — некоректні коди/повідомлення, UI-флоу; **P3** — косметика.

### 5.1 AUTH — автентифікація і доступ

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-AUTH-01` | `POST /operators` з валідними credentials і профілем створює auth-користувача й `Operator` одним запитом, ставить сесійну cookie; `GET /auth/me` повертає профіль; `id` оператора = id auth-користувача | EP | P1 | `03` Authentication & authorisation; `01` Operator.id | Ready |
| `TS-AUTH-02` | Новий оператор має `authorisation_level = L1`; передача `authorisation_level: "L3"` у тілі реєстрації не дає оператора з `L3` | Negative | P1 | `03` «`authorisation_level` is no longer self-service» | Ready |
| `TS-AUTH-03` | Реєстрація з уже зайнятим `email` / `national_id` — код і тіло відповіді | Negative | P2 | `01` Operator (unique) | Needs clarification (F-19) |
| `TS-AUTH-04` | `POST /auth/login` з правильними email+password → сесія; `GET /auth/me` → профіль | EP | P1 | `03` Auth | Ready |
| `TS-AUTH-05` | Логін з неправильним паролем / неіснуючим email — код відповіді | Negative | P2 | `03` Auth (не визначено) | Needs clarification (F-19) |
| `TS-AUTH-06` | `POST /auth/logout` → наступний `GET /auth/me` повертає `401` | State transition | P1 | `03` Auth | Ready |
| `TS-AUTH-07` | Без сесії і без `X-Reviewer-Id`: `POST/GET /pilots`, `POST/GET /aircraft`, `POST/GET /flight-requests`, `.../submit`, `.../cancel`, `.../validate`, `GET /auth/me` → `401` | Negative, EP | P1 | `03` Enforcement; Error semantics | Ready |
| `TS-AUTH-08` | Оператор A робить `GET` пілота / дрона / заявки / `GET /operators/{id}` оператора B → `403` | Negative | P1 | `03` Enforcement; Reviewer-or-owner reads | Ready |
| `TS-AUTH-09` | Оператор A викликає `submit`, `cancel`, `validate` на заявці оператора B → `403`, стан заявки B не змінився, `AuditLog` без нових записів | Negative | P1 | `03` Enforcement; endpoints 10, 13, 14 | Ready |
| `TS-AUTH-10` | `operator_id` у тілі `POST /pilots`, `POST /aircraft`, `POST /flight-requests` (зі значенням оператора B) — створений ресурс ніколи не належить B, лише оператору сесії | Negative | P1 | `03` «none of them accept it as a request field» | Ready |
| `TS-AUTH-11` | `GET /flight-requests` без reviewer-заголовка повертає тільки заявки оператора сесії; `?operator_id=<B>` не розширює вибірку | Negative | P1 | `03` Reviewer-or-owner reads; endpoint 12 | Ready |
| `TS-AUTH-12` | З `X-Reviewer-Id` (без сесії): `GET` пілота / дрона / оператора / заявки будь-якого оператора → `200` | EP | P1 | `03` Reviewer-or-owner reads | Ready |
| `TS-AUTH-13` | З `X-Reviewer-Id`: `GET /flight-requests` повертає заявки всіх операторів; фільтр `operator_id` звужує вибірку | EP | P2 | `03` endpoint 12 | Ready |
| `TS-AUTH-14` | Лише `X-Reviewer-Id` (без сесії) на owner-only ендпоінтах `submit` / `cancel` / `POST /flight-requests` — код відповіді | Negative | P2 | `03` endpoints 9, 10, 14; Error semantics | Needs clarification (F-16) |
| `TS-AUTH-15` | Оператор із сесією додає `X-Reviewer-Id` і схвалює **власну** заявку — чи це дозволено | Negative | P1 | `03` Reviewer; Auth | Needs clarification (F-17) |
| `TS-AUTH-16` | `GET` неіснуючого `pilots/{id}`, `aircraft/{id}`, `operators/{id}`, `flight-requests/{id}` (від рев'юера) → `404` | Negative | P3 | `03` Error semantics | Ready |

### 5.2 REG — реєстр пілотів і дронів

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-REG-01` | `POST /pilots` для кожного `rating` (`R_BASIC`, `R_ADVANCED`, `R_BVLOS`) створює пілота під оператором сесії | EP | P2 | `01` Pilot; `03` endpoint 3 | Ready |
| `TS-REG-02` | `GET /pilots/{id}` містить похідну currency: `certified_on` = сьогодні − 6 міс. → current; `certified_on` = сьогодні − 19 міс. → not current | EP | P2 | `01` Derived; `06` §2; `03` endpoint 4 | Ready (назва/формат поля — `F-27`) |
| `TS-REG-03` | `GET /pilots` повертає лише пілотів оператора сесії, кожен рядок з похідною currency | EP | P2 | `03` Registry (extension) | Ready |
| `TS-REG-04` | Похідний weight band у `GET /aircraft/{id}`: 900 → `C0`; 901 → `C1`; 3 500 → `C1`; 3 501 → `C2`; 12 000 → `C2`; 12 001 → `C3`; 30 000 → `C3` | BVA | P1 | `06` §3; `01` Derived | Ready |
| `TS-REG-05` | Регресія `OH-1` v1.4: 3 700 г і 4 000 г → `C2` (не `C1`) | BVA, регресія | P1 | `06` §3 (⚠️ v1.4) | Ready |
| `TS-REG-06` | `POST /aircraft` з `mtom_grams` = 30 001 → `422`, дрон не створено (немає в `GET /aircraft`) | BVA, Negative | P1 | `06` §1.2, §3; `OH-1 §3` | Ready |
| `TS-REG-07` | Тіло `422` для 30 001 г: `rule_reference` = `OH-1 §3`; чи є `failing_gate` і яке значення | Negative | P2 | `06` §6; `03` The 422 body | Needs clarification (F-18) |
| `TS-REG-08` | `GET /aircraft` повертає лише дрони оператора сесії з похідним band | EP | P2 | `03` Registry (extension) | Ready |
| `TS-REG-09` | Жоден band поза `C0`–`C3` не з'являється (`C4`/`C5` з `CR-1` відсутні): 30 000 г → `C3`, `permitted_bands` зон лише з `C0`–`C3` | Регресія | P2 | `06` §3 | Ready |
| `TS-REG-10` | Невалідне тіло (не JSON, рядок замість числа в `mtom_grams`, невідомий `rating`) → `400` | Negative | P2 | `03` Error semantics (`400`) | Ready |
| `TS-REG-11` | Дублікат `national_id` пілота / `serial_number` дрона — код відповіді | Negative | P2 | `01` (unique) | Needs clarification (F-19) |
| `TS-REG-12` | `mtom_grams` = 0 або від'ємне — приймається чи ні | BVA | P3 | `06` §3 (нижня межа 0 г) | Needs clarification (F-19) |
| `TS-REG-13` | Виправлення маси перераховує band одразу, вже видані дозволи не переглядаються | State | P2 | `01` Derived | Needs clarification (F-25) |

### 5.3 ZREF — довідник зон

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-ZREF-01` | `GET /zones` повертає рівно 6 зон з даними каталогу (клас, стеля, bands, curfew, рівень, `active`) — див. §6.1 | EP | P1 | `06` §4; `03` endpoint 7 | Ready |
| `TS-ZREF-02` | `KBP-CTR-3` і `ODS-CTR-5` мають curfew 23:00–05:00 (за `A-7`), а не 22:00–06:00 | Регресія | P1 | `06` §1.4, §4; `A-7 §1` | Ready |
| `TS-ZREF-03` | `GET /zones/{code}` для кожного з 6 кодів → дані зони; невідомий код → `404` | EP, Negative | P2 | `03` endpoint 8; Error semantics | Ready |
| `TS-ZREF-04` | `POST`/`PUT`/`PATCH`/`DELETE` на `/zones`, `/zones/{code}` не змінюють довідник (наступний `GET` незмінний) | Negative | P2 | `03` Reference data | Ready |
| `TS-ZREF-05` | Чи потрібна автентифікація для `GET /zones` | Negative | P3 | `03` (не визначено) | Needs clarification (F-26) |

### 5.4 FR — створення і список заявок

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-FR-01` | `POST /flight-requests` з валідними даними → заявка в `DRAFT`, `operator_id` = оператор сесії; `failing_gate`, `rule_reference`, `rejection_reason`, `submitted_at` порожні | EP | P1 | `03` endpoint 9; `02` States; `01` FlightRequest | Ready |
| `TS-FR-02` | Створення свідомо невалідної комбінації (`R_BASIC` + `ZTM-RST-7` + C3 + висота 500 м) → успішно `DRAFT`: гейти на create не запускаються | EP | P1 | `02` States («No gates have run»); `04` «form does not pre-filter» | Ready |
| `TS-FR-03` | `flight_purpose` 2 000 символів → приймається; 2 000 кириличних символів → приймається (ліміт у символах) | BVA | P2 | `01` FlightRequest, Time handling | Ready |
| `TS-FR-04` | `flight_purpose` 2 001 символ → відхилено, заявку не створено | BVA, Negative | P2 | `01` Time handling | Ready (код — `F-19`) |
| `TS-FR-05` | `window_end` = `window_start` або раніше на create → відхилено чи прийнято як `DRAFT`; код | BVA | P2 | `01` FlightRequest; `06` §1.4 Duration | Needs clarification (F-19) |
| `TS-FR-06` | Неіснуючі / чужі `pilot_id`, `aircraft_id`, неіснуючий `zone_code` у тілі — код відповіді | Negative | P1 | `03` Enforcement; Error semantics | Needs clarification (F-19) |
| `TS-FR-07` | Невалідне тіло (немає `window_start`, неправильний формат дати, не JSON) → `400` | Negative | P2 | `03` Error semantics | Ready |
| `TS-FR-08` | `GET /flight-requests` фільтр `state` і `zone_code` повертає тільки відповідні заявки | EP | P2 | `03` endpoint 12 | Ready |
| `TS-FR-09` | Фільтр за діапазоном дат — назви параметрів і семантика (по `window_start`? включно?) | EP | P3 | `03` endpoint 12 | Needs clarification (F-26) |

### 5.5 SM — state machine `FlightRequest`

Легальні переходи (таблиця `02`):

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-SM-01` | `DRAFT` → submit (усі гейти pass) → `UNDER_REVIEW`; відповідь і подальший `GET` ніколи не показують `SUBMITTED`; `submitted_at` заповнено | State transition | P1 | `02` таблиця переходів, States | Ready |
| `TS-SM-02` | `DRAFT` → submit (гейт fail) → `REJECTED`; збережені `failing_gate`, `rule_reference`, `rejection_reason` | State transition | P1 | `02`; `01` FlightRequest.rule_reference | Ready (HTTP-відповідь — `TS-PIPE-10`) |
| `TS-SM-03` | `UNDER_REVIEW` → approve (з `X-Reviewer-Id`) → `APPROVED` | State transition | P1 | `02`; `03` endpoint 15 | Ready |
| `TS-SM-04` | `UNDER_REVIEW` → reject з `note` → `REJECTED` | State transition | P1 | `02`; `03` endpoint 16 | Ready |
| `TS-SM-05` | `DRAFT` → cancel → `CANCELLED` | State transition | P1 | `02` | Ready |
| `TS-SM-06` | `UNDER_REVIEW` → cancel → `CANCELLED` | State transition | P1 | `02` | Ready |
| `TS-SM-07` | `APPROVED` → cancel (> 2 год до `window_start`) → `CANCELLED` | State transition | P1 | `02`; `06` §5 | Ready |
| `TS-SM-08` | `APPROVED` → complete → `COMPLETED` | State transition | P2 | `02`; `01` | Needs clarification (F-13) |

Нелегальні пари `(from, action)` → `409`, стан не змінюється:

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-SM-09` | `DRAFT`: approve, reject (з `X-Reviewer-Id` і note), complete → `409` | State transition, Negative | P1 | `02` «Any `(from, action)` pair not listed» | Ready (complete — `F-13`) |
| `TS-SM-10` | `UNDER_REVIEW`: повторний submit, complete → `409` | State transition, Negative | P1 | `02` (приклад re-submit) | Ready (complete — `F-13`) |
| `TS-SM-11` | `APPROVED`: submit, approve, reject → `409` | State transition, Negative | P1 | `02` | Ready |
| `TS-SM-12` | `REJECTED` (після гейту і після reviewer reject): submit, approve, reject, cancel, complete → `409` | State transition, Negative | P1 | `02` «`REJECTED`→anything» | Ready (complete — `F-13`) |
| `TS-SM-13` | `CANCELLED`: submit, approve, reject, cancel, complete → `409` | State transition, Negative | P1 | `02` «`CANCELLED`→anything» | Ready (complete — `F-13`) |
| `TS-SM-14` | `COMPLETED`: submit, approve, reject, cancel, complete → `409` | State transition, Negative | P2 | `02` «`COMPLETED`→anything» | Needs clarification (F-13) — стан досяжний лише через maintenance endpoint |
| `TS-SM-15` | Після кожної `409` з `TS-SM-09…14`: стан, `decided_*` не змінились, немає нових рядків `AuditLog` і `ReviewAction` | State transition | P1 | `02`; `01` AuditLog | Ready |

### 5.6 GP — гейт `PILOT`

Передумова для pass-сценаріїв у `CONTROLLED`/`RESTRICTED`: оператор `L2`/`L3`, дрон C0, висота ≤ стелі, вікно поза curfew.

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-GP-01` | Рейтинг не покриває клас: `R_BASIC`+`CONTROLLED` (`KBP-CTR-3`), `R_BASIC`+`RESTRICTED` (`LVV-RST-2`), `R_ADVANCED`+`RESTRICTED` (`ZTM-RST-7`) → `422`, `failing_gate: PILOT`, `rule_reference: "OH-1 §2.1"` | Decision table | P1 | `06` §1.1, §6 | Ready |
| `TS-GP-02` | Рейтинг покриває клас: `R_BASIC`+`OPEN`; `R_ADVANCED`+`OPEN`/`CONTROLLED`; `R_BVLOS`+`OPEN`/`CONTROLLED`/`RESTRICTED` → гейт `PILOT` pass | Decision table | P1 | `06` §1.1, §4 | Ready |
| `TS-GP-03` | Currency: `window_start` за 1 день **до** 18-місячної позначки від `certified_on` → pass | BVA | P1 | `06` §2 | Ready |
| `TS-GP-04` | Currency: `window_start` через 1 день **після** 18-місячної позначки → `422`, `PILOT`, `"OH-1 §2.3"` (без grace period) | BVA | P1 | `06` §1.1, §2, §6 | Ready |
| `TS-GP-05` | Currency: `window_start` рівно в день 18-місячної позначки; `certified_on` 31-го числа (напр. 2025-04-30 vs 2025-05-31 + 18 міс.) | BVA | P2 | `06` §2 | Needs clarification (F-6) |
| `TS-GP-06` | Пілот current на момент submit, але не на `window_start` (позначка між зараз і `window_start`) → `PILOT`, `"OH-1 §2.3"` | BVA | P1 | `06` §1.1 («not submission time») | Ready |
| `TS-GP-07` | Регресія `OH-1` v1.4: `certified_on` = `window_start` − 20 міс. → fail `PILOT` (24 міс. не застосовується) | Регресія | P1 | `06` §2 (⚠️ v1.4) | Ready |
| `TS-GP-08` | Пілот `INACTIVE` з валідним рейтингом і currency → гейт `PILOT` pass, заявка може дійти до `UNDER_REVIEW` | EP | P2 | `05` row 2; `07` п.2 | Ready |
| `TS-GP-09` | Рейтинг і currency не виконуються одночасно → `failing_gate: PILOT`; який `rule_reference` | Decision table | P2 | `06` §1.1, §6 | Needs clarification (F-20) |

### 5.7 GA — гейт `AIRCRAFT`

Пілот `R_BVLOS` current, оператор `L3`, щоб `PILOT` не маскував результат.

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-GA-01` | C3 (12 001 г) у `KBP-CTR-3` / `ODS-CTR-5` (C0–C2) → `422`, `failing_gate: AIRCRAFT` (**не** `ZONE`), `rule_reference: "ZC-2026.1 §2"` | EP | P1 | `06` §1.2, §6; `07` п.1 | Ready |
| `TS-GA-02` | Межі band по зонах: `ZTM-RST-7` 900 г pass / 901 г fail; `LVV-RST-2` 3 500 г pass / 3 501 г fail; `KBP-CTR-3` 12 000 г pass / 12 001 г fail; `DNI-OPN-1`, `KHR-OPN-4` 30 000 г pass | BVA, Decision table | P1 | `06` §3, §4 | Ready |
| `TS-GA-03` | Регресія `OH-1` v1.4: 3 700 г (C2) у `LVV-RST-2` → fail `AIRCRAFT` | Регресія | P1 | `06` §3 (⚠️ v1.4) | Ready |
| `TS-GA-04` | Дрон `RETIRED` з допустимим band → гейт `AIRCRAFT` pass | EP | P2 | `05` row 2; `07` п.2; `06` §1.2 | Ready |

### 5.8 GZ — гейт `ZONE`

Пілот `R_BVLOS` current, дрон C0, щоб попередні гейти пройшли.

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-GZ-01` | Висота = стелі → pass; стеля + 1 м → `422`, `ZONE`, `"ZC-2026.1 §2"`: `KBP-CTR-3` 120/121, `ODS-CTR-5` 90/91, `LVV-RST-2` 60/61, `ZTM-RST-7` 45/46, `DNI-OPN-1` 120/121 | BVA | P1 | `06` §1.3, §4, §6 | Ready |
| `TS-GZ-02` | Рівень авторизації: `L1`→`KBP-CTR-3`/`ODS-CTR-5` (L2), `L1`/`L2`→`LVV-RST-2`/`ZTM-RST-7` (L3) → `422`, `ZONE`, `"OH-1 §4"` | Decision table | P1 | `06` §1.3, §6; `07` п.4; `05` row 3 | Ready |
| `TS-GZ-03` | Рівень достатній: `L1`→`DNI-OPN-1`/`KHR-OPN-4`; `L2`→`KBP-CTR-3`/`ODS-CTR-5`; `L3`→ усі 6 → гейт `ZONE` pass | Decision table | P1 | `06` §1.3 | Ready |
| `TS-GZ-04` | Рівень оцінюється незалежно від рейтингу і band: `R_BVLOS` + C0 + оператор `L1` у `ZTM-RST-7` → `ZONE` (не `PILOT`/`AIRCRAFT`) | Decision table | P1 | `06` §1.3 («Independent of pilot rating and aircraft band») | Ready |
| `TS-GZ-05` | Неактивна зона (підготовлена поза продуктом) → `422`, `failing_gate: ZONE` | EP | P1 | `05` §2; `06` §1.3 | Ready |
| `TS-GZ-06` | Неактивна зона — дослівне значення `rule_reference` | EP | P2 | `05` §2 | Blocked (OQ-5) — не визначено, див. `05` §2 |
| `TS-GZ-07` | Висота і рівень авторизації не виконуються одночасно → `ZONE`; який `rule_reference` | Decision table | P2 | `06` §1.3, §6 | Needs clarification (F-20) |

### 5.9 GT — гейт `TIME_WINDOW`

`T` — момент виклику submit. Пілот `R_BVLOS`, дрон C0, оператор `L3`, зона `DNI-OPN-1` (без curfew), якщо не вказано інше.

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-GT-01` | Lead time: `window_start` = `T` + 6 год 10 хв → pass; `T` + 5 год 50 хв → `422`, `TIME_WINDOW`, `"TSD-4 §2.1"` | BVA | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-02` | Advance: `window_start` = `T` + 59 днів → pass; `T` + 61 день → `422`, `TIME_WINDOW`, `"TSD-4 §2.2"` | BVA | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-03` | Advance рівно 60 днів (± хвилини) — одиниця «днів» і точка відліку | BVA | P3 | `06` §1.4 | Needs clarification (F-8) |
| `TS-GT-04` | Тривалість 15 і 240 хв → pass; 14 і 241 хв → `422`, `TIME_WINDOW`, `"TSD-4 §3"` | BVA | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-05` | Вікно повністю в curfew за будь-якого зміщення UTC+0…+3 (напр. 23:30–00:30 UTC): `KBP-CTR-3`, `ODS-CTR-5` → `"A-7 §1"`; `LVV-RST-2`, `ZTM-RST-7` → `"ZC-2026.1 §3"`; `failing_gate: TIME_WINDOW` | EP | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-06` | Curfew через північ — один період: `LVV-RST-2`, вікно 00:00–01:00 UTC (локально в межах 00:00–04:00 за UTC+0…+3) → fail | EP | P1 | `06` §1.4 («crossing midnight are one continuous period») | Ready |
| `TS-GT-07` | Зони без curfew (`DNI-OPN-1`, `KHR-OPN-4`): нічне вікно 23:30–00:30 UTC → curfew-перевірка pass | EP | P2 | `06` §1.4, §4 | Ready |
| `TS-GT-08` | Регресія `A-7`: `KBP-CTR-3`/`ODS-CTR-5` вікна 22:00–23:00 і 05:00–06:00 **локального** часу → pass (базове значення 22:00–06:00 не застосовується) | Регресія, BVA | P1 | `06` §1.4; `A-7 §1, §4` | Needs clarification (F-5) |
| `TS-GT-09` | Частковий перетин curfew (початок поза, кінець усередині на 1 хв; і навпаки) → fail (перевірка minute-by-minute, не лише кінців) | BVA | P1 | `06` §1.4 | Needs clarification (F-5) |
| `TS-GT-10` | Вікно, що закінчується рівно на початку curfew / починається рівно в кінці curfew | BVA | P2 | `06` §1.4 | Needs clarification (F-5, F-9) |
| `TS-GT-11` | Перехід на літній/зимовий час: curfew у ніч зміни часу | BVA | P3 | `06` §1.4; `01` Time handling | Needs clarification (F-5) |
| `TS-GT-12` | Overlap з іншою заявкою того ж дрона в `UNDER_REVIEW` (перетин 1 хв) → заявка **не** потрапляє в `UNDER_REVIEW` | BVA | P1 | `06` §1.4 (`TSD-4 §5`); `02` | Ready (код/тіло/стан — `TS-GT-14`) |
| `TS-GT-13` | Overlap з `APPROVED` заявкою того ж дрона → заявка **не** потрапляє в `UNDER_REVIEW` | EP | P1 | `06` §1.4 | Ready (код/тіло/стан — `TS-GT-14`) |
| `TS-GT-14` | Відповідь на submit при overlap: `422` `TIME_WINDOW` `"TSD-4 §5"` + стан `REJECTED`, чи `409` + стан `DRAFT` | Negative | P1 | `06` §1.4, §6 vs `03` Error semantics | Needs clarification (F-3) |
| `TS-GT-15` | Дотик меж: існуюча `APPROVED` 12:00–14:00, нова 14:00–15:00 того ж дрона → overlap pass | BVA | P1 | `06` §1.4 | Ready |
| `TS-GT-16` | Існуюча заявка того ж дрона в `DRAFT`, `REJECTED`, `CANCELLED` з тим самим вікном → overlap pass | EP | P1 | `06` §1.4; `02` | Ready |
| `TS-GT-17` | Існуюча `COMPLETED` заявка того ж дрона з перетином → overlap pass | EP | P3 | `06` §1.4 | Needs clarification (F-13) — потрібен `COMPLETED` |
| `TS-GT-18` | Інший дрон (того ж або іншого пілота) з тим самим вікном → overlap pass; той самий пілот на двох дронах не є overlap | EP | P2 | `06` §1.4 («same aircraft») | Ready |
| `TS-GT-19` | Дві умови `TIME_WINDOW` одночасно (тривалість 241 хв + curfew) → `TIME_WINDOW`; який `rule_reference` | Decision table | P3 | `06` §1.4, §6 | Needs clarification (F-20) |

### 5.10 PIPE — порядок гейтів і формат `422`

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-PIPE-01` | `PILOT` + `AIRCRAFT` fail (`R_BASIC` + C3 у `KBP-CTR-3`) → лише `PILOT` | Pairwise | P1 | `06` Evaluation order; `03` Gate pipeline | Ready |
| `TS-PIPE-02` | `PILOT` + `ZONE` fail (`R_BASIC` у `ZTM-RST-7` оператором `L1`) → `PILOT` | Pairwise | P1 | там само | Ready |
| `TS-PIPE-03` | `PILOT` + `TIME_WINDOW` fail (`R_BASIC` у `KBP-CTR-3`, тривалість 241 хв) → `PILOT` | Pairwise | P1 | там само | Ready |
| `TS-PIPE-04` | `AIRCRAFT` + `ZONE` fail (C3 + висота 121 м у `KBP-CTR-3`) → `AIRCRAFT` | Pairwise | P1 | там само | Ready |
| `TS-PIPE-05` | `AIRCRAFT` + `TIME_WINDOW` fail (C3 у `ODS-CTR-5`, lead time 1 год) → `AIRCRAFT` | Pairwise | P1 | там само | Ready |
| `TS-PIPE-06` | `ZONE` + `TIME_WINDOW` fail (висота 61 м у `LVV-RST-2`, тривалість 14 хв) → `ZONE` | Pairwise | P1 | там само | Ready |
| `TS-PIPE-07` | Падають усі чотири → тільки `PILOT`, відповідь не перелічує інших гейтів | Pairwise | P1 | `06` Evaluation order («never every gate») | Ready |
| `TS-PIPE-08` | Тіло `422` має `failing_gate` (одне з 4), `rule_reference` дослівно з `06` §6, непорожній `message` для оператора; `rule_reference` ніколи не `"TSD-4 §3.2"` (ілюстративний приклад) | EP | P2 | `03` The 422 body; `06` §6 | Ready |
| `TS-PIPE-09` | Altitude-fail і band-fail мають однаковий `rule_reference` `"ZC-2026.1 §2"` і розрізняються лише `failing_gate` (`ZONE` vs `AIRCRAFT`) — це не дефект | EP | P2 | `06` §1.2 note | Ready |
| `TS-PIPE-10` | HTTP-відповідь submit при провалі гейту: `422` з тілом чи `200/201` з новим станом `REJECTED` | Negative | P2 | `03` endpoint 10, Error semantics vs `08` Gate Pipeline | Needs clarification (F-12) |
| `TS-PIPE-11` | Відхилена гейтом заявка: `GET /flight-requests/{id}` після перезавантаження містить ті самі `failing_gate`, `rule_reference`, що були у відповіді, і непорожній `rejection_reason` | EP | P1 | `01` FlightRequest (`rule_reference` survives); `02` States | Ready |

### 5.11 VAL — `validate`

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-VAL-01` | `validate` на `DRAFT` з провальною комбінацією: результат називає гейт; стан лишається `DRAFT`, `submitted_at` порожній, `failing_gate` у заявці не записано, немає `AuditLog`/`ReviewAction` | State | P1 | `03` endpoint 13; `02` | Ready |
| `TS-VAL-02` | `validate` на `DRAFT`, що проходить усі гейти: стан лишається `DRAFT` | State | P1 | `03` endpoint 13 | Ready |
| `TS-VAL-03` | 5 послідовних `validate` → однаковий результат, жодних побічних ефектів | EP | P2 | `03` «cheap and safe to call repeatedly» | Ready |
| `TS-VAL-04` | Для кожного провального сценарію з `TS-PIPE-01…07` `validate` і подальший submit дають той самий `failing_gate` і `rule_reference` | EP | P1 | `03` «single shared function»; Gate pipeline | Ready |
| `TS-VAL-05` | `validate` pass → інша заявка того ж дрона з перетином схвалена → submit першої не дає `UNDER_REVIEW` (очікувана поведінка, не дефект) | State | P1 | `02` Aircraft schedule overlap | Ready (код — `F-3`) |
| `TS-VAL-06` | Формат результату: per-gate pass/fail для всіх 4 гейтів чи лише перша відмова; HTTP-статус `validate` при провалі гейту | EP | P2 | `03` endpoint 13; `04` «per-gate pass/fail», «all four gate results»; `08` | Needs clarification (F-10) |
| `TS-VAL-07` | `validate` на `UNDER_REVIEW` (як у деталях рев'юера) — чи заявка не конфліктує сама з собою в overlap; `validate` на `APPROVED`/термінальних | State | P1 | `04` Reviewer queue; `03` endpoint 13; `06` §1.4 | Needs clarification (F-11) |
| `TS-VAL-08` | Lead time / advance у `validate`: від якого моменту рахується «submission» | BVA | P2 | `06` §1.4 | Needs clarification (F-8) |

### 5.12 REV — рішення рев'юера

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-REV-01` | approve `UNDER_REVIEW` з `X-Reviewer-Id: rev-qa-1` → `APPROVED`; `ReviewAction` (`action=APPROVE`, `reviewer_id=rev-qa-1`, `created_at`); `decided_at` і `decided_by=rev-qa-1` у заявці | EP | P1 | `03` Reviewer; `01` ReviewAction, FlightRequest | Ready |
| `TS-REV-02` | reject з `note` → `REJECTED`; `ReviewAction` (`REJECT`, note збережено); `decided_*` заповнено | EP | P1 | `03` endpoint 16; `01` ReviewAction | Ready |
| `TS-REV-03` | Після reviewer reject: значення `failing_gate` і `rejection_reason` у заявці | EP | P2 | `01` FlightRequest vs `02` States | Needs clarification (F-14) |
| `TS-REV-04` | reject без `note` / з порожнім `note` → відмова, стан `UNDER_REVIEW`, немає `ReviewAction` | Negative | P1 | `03` endpoint 16; `01` ReviewAction.note | Ready (код — `F-15`) |
| `TS-REV-05` | approve / reject без `X-Reviewer-Id` (у т.ч. з сесією оператора) → `400`, стан не змінився | Negative | P1 | `03` Reviewer («Return `400`») | Ready |
| `TS-REV-06` | approve з необов'язковим `note` → `APPROVED` (note обов'язковий лише для `REJECT`) | EP | P3 | `01` ReviewAction.note | Ready |
| `TS-REV-07` | `X-Reviewer-Id` з порожнім значенням / пробілами | Negative | P3 | `03` Reviewer | Needs clarification (F-16) |
| `TS-REV-08` | approve, коли з моменту submit lead time вже < 6 год або зона стала неактивною — чи гейти перевіряються повторно | State | P2 | `02`; `06` §1.4 | Needs clarification (F-21) |

### 5.13 CAN — скасування

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-CAN-01` | cancel `APPROVED`, `window_start` = зараз + 2 год 10 хв → `CANCELLED` | BVA | P1 | `06` §5; `02` | Ready |
| `TS-CAN-02` | cancel `APPROVED`, `window_start` = зараз + 1 год 50 хв → `409` (**не** `422`), стан `APPROVED` | BVA, Negative | P1 | `02` Cancellation deadline; `06` §5 | Ready |
| `TS-CAN-03` | cancel `APPROVED` рівно за 2 год до `window_start` | BVA | P3 | `02`; `06` §5 («up to 2 hours») | Needs clarification (F-7) |
| `TS-CAN-04` | cancel `APPROVED` після `window_start` / після `window_end` → `409` | BVA | P2 | `02` | Ready |
| `TS-CAN-05` | Дедлайн не поширюється на `DRAFT` і `UNDER_REVIEW`: cancel `UNDER_REVIEW` з `window_start` < 2 год → `CANCELLED` | EP | P2 | `02` таблиця (дедлайн лише для `APPROVED`) | Ready |
| `TS-CAN-06` | Після cancel `APPROVED` заявки нова заявка того ж дрона з тим самим вікном проходить overlap | EP | P1 | `06` §1.4 | Ready |

### 5.14 CMP — завершення

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-CMP-01` | `APPROVED` заявка, у якої `window_end` минув, лишається `APPROVED` (не стає `COMPLETED` за часом) | State | P1 | `02` States; `01` | Ready |
| `TS-CMP-02` | Maintenance endpoint: шлях, доступ, відповідь; `APPROVED` → `COMPLETED`; інші стани → `409` | State transition | P2 | `02` | Needs clarification (F-13) |

### 5.15 AUD — аудит

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-AUD-01` | Кожен перехід (`TS-SM-01…07`) створює рядок `AuditLog` з `entity_type="FlightRequest"`, `entity_id`, правильними `from_state`/`to_state`, `actor`, `created_at` | EP | P1 | `01` AuditLog; `02` Implementation note | Ready |
| `TS-AUD-02` | Створення заявки → рядок з `from_state = null`, `to_state = DRAFT` | EP | P2 | `01` AuditLog (nullable for creation events) | Ready |
| `TS-AUD-03` | Submit: один рядок `DRAFT→UNDER_REVIEW`/`REJECTED` чи два через `SUBMITTED`; значення `actor` для автоматичного переходу і дій рев'юера | EP | P2 | `01` AuditLog; `02` | Needs clarification (F-22) |
| `TS-AUD-04` | Немає шляху update/delete `AuditLog`: в API немає ендпоінта, UI не має елементів редагування/видалення аудиту | Negative | P1 | `01` AuditLog («Never expose») | Ready |
| `TS-AUD-05` | `validate`, відповіді `400`/`401`/`403`/`409` не створюють рядків `AuditLog` | Negative | P1 | `01` AuditLog; `03` endpoint 13 | Ready |
| `TS-AUD-06` | Переходи інших сутностей (`Operator.status`, `Pilot.status`, `Aircraft.status` — змінюються out-of-band) логуються | EP | P3 | `01` AuditLog («on any entity») | Needs clarification (F-22) |

### 5.16 SUSP — оператор `SUSPENDED`

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-SUSP-01` | `SUSPENDED` оператор: create, submit, approve (його заявки) **не** повертають `422` з `failing_gate` | Negative | P1 | `05` §1; `07` п.3 | Ready |
| `TS-SUSP-02` | `SUSPENDED` оператор: create не створює заявку; submit не змінює стан `DRAFT`; approve не змінює стан `UNDER_REVIEW` | Negative | P1 | `05` §1 («request is blocked»); `07` п.3 | Ready |
| `TS-SUSP-03` | `POST /flight-requests` від `SUSPENDED` — статус-код і тіло | Negative | P1 | `05` §1 | Blocked (OQ-4) — не визначено, див. `05` §1 |
| `TS-SUSP-04` | `.../submit` від `SUSPENDED` — статус-код і тіло | Negative | P1 | `05` §1 | Blocked (OQ-4) — не визначено, див. `05` §1 |
| `TS-SUSP-05` | `.../approve` заявки `SUSPENDED` оператора — статус-код і тіло | Negative | P1 | `05` §1 | Blocked (OQ-4) — не визначено, див. `05` §1 |
| `TS-SUSP-06` | `SUSPENDED` оператор: cancel, validate, реєстрація пілота/дрона, логін | Negative | P2 | `05` §1 (перелічено лише create/submit/approve) | Needs clarification (F-23) |

### 5.17 UIO — портал оператора

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-UIO-01` | `/register`: одна форма з email, password і полями профілю (`legal_name`, `national_id`, …); поля `authorisation_level` немає; після submit оператор одразу залогінений (видно `operator-session-info`) | EP | P2 | `04` Operator portal | Ready |
| `TS-UIO-02` | `/login` → сесія, `operator-logout-button` → вихід; сторінки порталу без логіну недоступні | EP | P2 | `04` Operator portal («Requires login») | Ready |
| `TS-UIO-03` | Реєстр пілотів: форма створення, список лише своїх пілотів з currency | EP | P2 | `04`; `03` Registry | Ready |
| `TS-UIO-04` | Реєстр дронів: форма створення, список з band; 30 001 г — помилка, дрон не з'являється у списку | EP, BVA | P2 | `04`; `06` §3 | Ready |
| `TS-UIO-05` | Форма заявки: zone picker показує 6 зон з `GET /zones`; pilot/aircraft pickers — лише свої | EP | P2 | `04` Request creation form | Ready |
| `TS-UIO-06` | Pickers **не** префільтровані: можна обрати `R_BASIC` + `ZTM-RST-7` + C3; правило спрацьовує лише на validate/submit | Negative | P1 | `04` «The form does not pre-filter» | Ready |
| `TS-UIO-07` | «Check before submitting» показує результат по гейтах; після нього заявка в «My requests» лишається `DRAFT` | State | P1 | `04` Operator portal; `03` endpoint 13 | Ready (формат — `F-10`) |
| `TS-UIO-08` | Коли в UI-флоу створюється `DRAFT` (до «Check before submitting» потрібен `id`); є елементи submit і cancel | EP | P2 | `04` Request creation form | Needs clarification (F-24) |
| `TS-UIO-09` | «My requests»: лише свої заявки, фільтр за станом | EP | P2 | `04` | Ready |
| `TS-UIO-10` | Деталі відхиленої гейтом заявки показують rejection reason і rule reference (зберігаються після перезавантаження) | EP | P1 | `04`; `01` FlightRequest.rule_reference | Ready |
| `TS-UIO-11` | Перемикач ролей у хедері: оператор ↔ рев'юер | EP | P3 | `04` вступ | Ready |

### 5.18 UIR — черга рев'юера

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-UIR-01` | Рев'юер вводить `X-Reviewer-Id` у хедері без логіну і бачить чергу | EP | P2 | `04` вступ, Reviewer queue | Ready |
| `TS-UIR-02` | Таблиця містить усі `UNDER_REVIEW` заявки всіх операторів і жодної в іншому стані | EP | P1 | `04` Reviewer queue; `03` endpoint 12 | Ready |
| `TS-UIR-03` | Сортування за часом подачі (`submitted_at`) в обох напрямках | EP | P3 | `04` | Ready |
| `TS-UIR-04` | Деталі заявки показують результати всіх 4 гейтів | EP | P2 | `04`; `03` Reviewer-or-owner reads | Needs clarification (F-10, F-11) |
| `TS-UIR-05` | Approve з деталей → заявка зникає з черги, стан `APPROVED` | State | P1 | `04`; `02` | Ready |
| `TS-UIR-06` | Reject без note неможливий (UI або сервер відхиляє), з note → `REJECTED`, зникає з черги | Negative | P1 | `04` «reject requires a note» | Ready |
| `TS-UIR-07` | Без введеного `X-Reviewer-Id` approve/reject не виконується | Negative | P2 | `03` Reviewer (`400`) | Ready |

### 5.19 TID — контракт `data-testid`

| ID | Сценарій | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-TID-01` | Кожен інтерактивний елемент (кнопки, поля, pickers, посилання, рядки таблиці з діями) на всіх сторінках обох порталів має непорожній `data-testid` | EP | P2 | `04` Selector stability | Ready |
| `TS-TID-02` | Розлогінений хедер: `operator-login-link`, `operator-register-link`; залогінений: `operator-session-info`, `operator-logout-button` | EP | P2 | `04` Two contracts | Ready |
| `TS-TID-03` | `operator-id-input` відсутній на всіх сторінках | Negative | P3 | `04` Two contracts | Ready |
| `TS-TID-04` | Значення `data-testid` стабільні між перезавантаженнями і станами сторінки | EP | P3 | `04` Selector stability | Ready |
| `TS-TID-05` | Звірка конкретних значень `data-testid` з еталоном (крім 4 auth-елементів) | EP | P2 | `04` | Needs clarification (F-4) |

## 6. Тестові дані

### 6.1 Зони (засіяні; лише читання)

| Code | Class | Ceiling (pass / fail) | Bands | Межа маси (pass / fail) | Curfew (локальний) | `rule_reference` curfew | Рівень |
|---|---|---|---|---|---|---|---|
| `KBP-CTR-3` | `CONTROLLED` | 120 / 121 м | C0–C2 | 12 000 / 12 001 г | 23:00–05:00 | `A-7 §1` | `L2` |
| `ODS-CTR-5` | `CONTROLLED` | 90 / 91 м | C0–C2 | 12 000 / 12 001 г | 23:00–05:00 | `A-7 §1` | `L2` |
| `LVV-RST-2` | `RESTRICTED` | 60 / 61 м | C0–C1 | 3 500 / 3 501 г | 22:00–06:00 | `ZC-2026.1 §3` | `L3` |
| `ZTM-RST-7` | `RESTRICTED` | 45 / 46 м | C0 | 900 / 901 г | 22:00–06:00 | `ZC-2026.1 §3` | `L3` |
| `DNI-OPN-1` | `OPEN` | 120 / 121 м | C0–C3 | 30 000 г (вище — реєстрація неможлива) | немає | — | `L1` |
| `KHR-OPN-4` | `OPEN` | 120 / 121 м | C0–C3 | 30 000 г | немає | — | `L1` |

Джерело: `06` §4, §1.4, §6. Неактивна зона: усі 6 активні; для `TS-GZ-05/06` потрібна підготовка поза продуктом (окремий стенд або тимчасова зміна seed з відновленням — інакше зламаються інші тести).

### 6.2 Оператори

| Дані | Рівень | Статус | Як створити |
|---|---|---|---|
| `OP-A-L1`, `OP-B-L1` (для тестів доступу) | `L1` | `ACTIVE` | через `/register` |
| `OP-L2` | `L2` | `ACTIVE` | реєстрація + out-of-band підвищення рівня |
| `OP-L3` | `L3` | `ACTIVE` | реєстрація + out-of-band підвищення рівня |
| `OP-SUSP` (з готовими `DRAFT` і `UNDER_REVIEW` заявками до блокування) | `L3` | `SUSPENDED` | реєстрація + out-of-band зміна статусу |

### 6.3 Пілоти (`D` = дата `window_start` тестової заявки)

| Дані | `rating` | `certified_on` | Очікування currency на `D` |
|---|---|---|---|
| `P-BASIC`, `P-ADV`, `P-BVLOS` | кожен | сьогодні − 6 міс. | current |
| `P-EDGE-IN` | `R_BVLOS` | `D` − 18 міс. + 1 день | current (`TS-GP-03`) |
| `P-EDGE-OUT` | `R_BVLOS` | `D` − 18 міс. − 1 день | not current (`TS-GP-04`) |
| `P-LAPSE` | `R_BVLOS` | сьогодні − 18 міс. + 10 днів, `D` = сьогодні + 20 днів | current зараз, not current на `D` (`TS-GP-06`) |
| `P-20M` | `R_BVLOS` | `D` − 20 міс. | not current (регресія v1.4) |
| `P-INACTIVE` | `R_BVLOS` | сьогодні − 6 міс., `status=INACTIVE` (out-of-band) | current, гейт pass |

### 6.4 Дрони (`mtom_grams`)

| Маса | Band | Призначення |
|---|---|---|
| 900 / 901 | `C0` / `C1` | межа C0/C1, `ZTM-RST-7` |
| 3 500 / 3 501 | `C1` / `C2` | межа C1/C2, `LVV-RST-2` |
| 3 700, 4 000 | `C2` | регресія `OH-1` v1.4 |
| 12 000 / 12 001 | `C2` / `C3` | межа C2/C3, `KBP-CTR-3`/`ODS-CTR-5` |
| 30 000 | `C3` | верхня межа, `DNI-OPN-1` |
| 30 001 | — | `422` на реєстрації |
| 500 (`RETIRED`, out-of-band) | `C0` | `TS-GA-04` |

### 6.5 Часові вікна (`T` — момент submit / cancel, UTC)

| Перевірка | Pass | Fail |
|---|---|---|
| Lead time (`TSD-4 §2.1`) | `T` + 6 год 10 хв | `T` + 5 год 50 хв |
| Advance (`TSD-4 §2.2`) | `T` + 59 днів | `T` + 61 день |
| Тривалість (`TSD-4 §3`) | 15 хв, 240 хв | 14 хв, 241 хв |
| Curfew `KBP`/`ODS`/`LVV`/`ZTM` (надійно за UTC+0…+3) | — | 23:30–00:30 UTC; 00:00–01:00 UTC |
| Curfew, межі / `A-7` регресія | після `F-5` | після `F-5` |
| Overlap (`TSD-4 §5`) | існуюча 12:00–14:00, нова 14:00–15:00 | існуюча 12:00–14:00, нова 13:59–15:00 |
| Дедлайн скасування (`TSD-4 §6`) | `window_start` − 2 год 10 хв | `window_start` − 1 год 50 хв |

Δ ±10 хв обрано через ручне виконання (затримка між підготовкою і викликом). Сценарії дедлайну скасування потребують `APPROVED` заявки з `window_start` < 2 год 10 хв від моменту cancel, а submit вимагає lead time ≥ 6 год — тобто чекати ~4 год після submit або керувати часом стенду.

## 7. Середовище та передумови

- Розгорнутий тестовий стенд SkyPermit з засіяним каталогом зон (§6.1) і зафіксованою версією docs (`78407f1`).
- REST-клієнт зі збереженням cookie (окремі сесії для `OP-A`, `OP-B`, `OP-L2`, `OP-L3`, `OP-SUSP`) і значення `X-Reviewer-Id`: `rev-qa-1`, `rev-qa-2`.
- Read-only доступ до БД стенду для `AuditLog` і `ReviewAction` (API для них немає).
- Out-of-band адмін-доступ (або підготовлені дані): `L2`/`L3`, `SUSPENDED`, `INACTIVE`, `RETIRED`, неактивна зона.
- Спосіб завершити заявку (maintenance endpoint) — після `F-13`.
- Бажано: керування часом стенду для `TS-CAN-*`, `TS-CAN-05`, `TS-GP-06`; без нього — планувати ці сценарії з очікуванням.
- Підтверджений часовий пояс кожної зони (`F-5`) — для curfew-граничних сценаріїв.

## 8. Критерії входу, виходу, призупинення

- **Вхід:** стенд доступний, `GET /zones` повертає 6 зон з §6.1; тестові акаунти і дані з §6 підготовлені; smoke (`TS-AUTH-01`, `TS-FR-01`, `TS-SM-01`, `TS-REV-01`) пройдено; відповіді власника на `F-3`, `F-10`, `F-12` отримані або явно відкладені.
- **Вихід:** 100 % P1 `Ready` виконано і пройдено; ≥ 95 % P2 `Ready` виконано; немає відкритих дефектів P1, дефекти P2 мають погоджене рішення; усі `Blocked` і `Needs clarification` перелічено у звіті з посиланням на OQ/F; знахідки `F-*` передано власнику docs.
- **Призупинення:** неправильний seed зон; `validate` змінює стан (зламана база для `VAL`/`UIO`); не працює auth (блокує все); масові `500`. **Відновлення:** після фіксу — повторний smoke + повний прогін зачепленої області.

## 9. Ризики та припущення

| Ризик / припущення | Вплив | Пом'якшення |
|---|---|---|
| Часовий пояс зон не визначено (`F-5`) | curfew-межі й регресія `A-7` (P1) не тестуються точно; DST може зсувати curfew на годину | надійні сценарії за UTC+0…+3 (`TS-GT-05/06`); отримати відповідь до релізу |
| Невизначена відповідь на overlap (`F-3`) і на провал гейту при submit (`F-12`) | тестувальник може позначити коректну систему як зламану | перевіряти лише «не `UNDER_REVIEW`» до уточнення |
| `X-Reviewer-Id` — мітка, не автентифікація; оператор може себе «схвалити» (`F-17`) | неправомірний дозвіл на політ | ескалувати як продуктовий ризик до релізу |
| Скасування з `SUBMITTED` не тестується мануально | незнайдений дефект у вузькому вікні | винести на рівень code review / автотестів |
| Out-of-band підготовка даних (`L2`/`L3`, `SUSPENDED`, неактивна зона) | залежність від адміна/БД; зміна seed може зламати інші тести | окремий стенд або відновлення seed після `TS-GZ-05/06` |
| Без керування часом дедлайн скасування вимагає очікування ~4 год | затримка виконання | групувати такі сценарії в одну сесію |
| `AuditLog` читається лише з БД | без доступу P1-сценарії `AUD` не виконуються | забезпечити read-only доступ на вході |
| Припущення: `T` у lead time = момент виклику submit (`submitted_at`) | можливе розходження для `validate` | `F-8` |

## 10. Заблоковані області (відкриті питання з `05`)

| OQ | Питання (з `05`) | Які сценарії заблоковано | Що потрібно для розблокування |
|---|---|---|---|
| OQ-4 | `Operator.status = SUSPENDED` — статус-код і форма відповіді | `TS-SUSP-03`, `TS-SUSP-04`, `TS-SUSP-05` | визначений статус-код і тіло для create / submit / approve (`03` пропонує `403`, але `05` вважає це лише заглушкою — `F-1`) |
| OQ-5 | Zone `active` — цитата | `TS-GZ-06` | підтвердження `ZC-2026.1 §4` як `rule_reference` (саме правило вже тестується в `TS-GZ-05`) |

## 11. Знахідки в документації

| ID | Опис | Де (обидві сторони) | Вплив на тестування | Питання власнику |
|---|---|---|---|---|
| `F-1` | `03` подає `403` як відповідь на `SUSPENDED`-precondition, ніби це вирішено; `05` каже, що статус-код не визначено, `403` — лише заглушка | `03` Error semantics vs `05` §1 / row 4 | статус береться з `05` → `Blocked (OQ-4)`; `403` не можна використовувати як очікуваний | Прибрати згадку `403` з `03` або закрити OQ-4 |
| `F-2` | `02` каже, що overlap рахується «inside the `AIRCRAFT` / `TIME_WINDOW` gates»; `06` відносить overlap лише до `TIME_WINDOW` (`TSD-4 §5`) | `02` Rules that affect…, Aircraft schedule overlap vs `06` §1.4, §6 | `failing_gate` для overlap: план очікує `TIME_WINDOW` (06 + першоджерело `TSD-4`), але потрібне підтвердження | Підтвердити, що overlap — лише `TIME_WINDOW` |
| `F-3` | `03`: `409` — «scheduling conflict between requests»; `06`: overlap — умова гейту `TIME_WINDOW` (тобто `422` і стан `REJECTED`) | `03` Error semantics vs `06` §1.4, §6 | неможливо визначити код, тіло і стан після submit з overlap (`TS-GT-14`, частково `TS-VAL-05`) | `422`+`REJECTED` чи `409`+`DRAFT`? Що мається на увазі під «scheduling conflict» у `03`? |
| `F-4` | `04` вимагає `data-testid` на кожному інтерактивному елементі, але конкретні значення дає лише для 4 auth-елементів; `flight-request-submit-button` — лише «e.g.» | `04` Two contracts | немає еталону для звірки (`TS-TID-05`) | Опублікувати реєстр `data-testid` |
| `F-5` | Часовий пояс зон не названо; curfew — у «local zone time», вікна — UTC; DST не згадано | `06` §1.4; `01` Time handling; `01` Zone (немає поля timezone) | граничні curfew-сценарії і регресія `A-7` (`TS-GT-08…11`) не мають очікуваних UTC-значень | Який IANA-пояс для кожної зони (напр. `Europe/Kyiv`)? Як обробляється DST? |
| `F-6` | Не визначено, чи pilot current рівно в день 18-місячної позначки; як рахувати від 29–31 числа; порівняння `certified_on` (date) з `window_start` (timestamptz) — за датою UTC чи моментом | `06` §2; `01` Pilot, FlightRequest | `TS-GP-05` без очікування | Правило «+18 міс.» для кінця місяця і включність позначки |
| `F-7` | «up to 2 hours before `window_start`» — чи рівно 2 год ще дозволено; від якого моменту (виклик cancel) | `02` Cancellation deadline; `06` §5 | `TS-CAN-03` | Межа включна чи ні? |
| `F-8` | Lead time / advance рахуються «between submission and `window_start`» — для `validate` моменту подачі немає; «60 днів» — календарні дні чи 60×24 год | `06` §1.4; `03` endpoint 13 | `TS-VAL-08`, `TS-GT-03` | Точка відліку для `validate` і одиниця «днів» |
| `F-9` | Для overlap явно сказано, що дотик меж не є перетином; для curfew аналогічного правила немає (вікно до 23:00 при curfew з 23:00; кінець curfew 05:00 — включно?) | `06` §1.4 Curfew vs No overlap | `TS-GT-10` | Чи є дотик межі curfew порушенням? |
| `F-10` | Pipeline зупиняється на першій відмові, але `04` очікує «per-gate pass/fail» і «all four gate results» від `validate`; `08` показує `validate` як `200 with per-gate results`, водночас у тій самій діаграмі pipeline повертає `422` | `03` endpoint 13, Gate pipeline; `06` Evaluation order vs `04` Operator portal, Reviewer queue; `08` Gate Pipeline | невідомий формат і HTTP-статус `validate` (`TS-VAL-06`, `TS-UIO-07`, `TS-UIR-04`) | Що повертає `validate` для гейтів після першої відмови (not evaluated?) і з яким статусом? |
| `F-11` | Поведінка `validate` на не-`DRAFT` заявках не визначена; при `validate` `UNDER_REVIEW` заявки overlap-перевірка може знайти саму заявку (вона `UNDER_REVIEW`) | `03` endpoint 13; `04` Reviewer queue; `06` §1.4 | `TS-VAL-07`, `TS-UIR-04` — ризик хибної відмови у деталях рев'юера | Чи виключається сама заявка з overlap? Які стани допустимі для `validate`? |
| `F-12` | Відповідь submit при провалі гейту: `03` описує `422` з тілом; `08` (похідний) показує `200/201 with new state` після запису `REJECTED` | `03` endpoint 10, Error semantics vs `08` Gate Pipeline | `TS-PIPE-10`; за правилами пріоритету `08` поступається, але `03` не каже явно, що `422` і збереження `REJECTED` відбуваються разом | Підтвердити: `422` + стан `REJECTED` збережено? |
| `F-13` | Maintenance completion endpoint згаданий у `02`/`01`, але відсутній у `03` (шлях, доступ, коди) | `02` таблиця, States vs `03` (16 ендпоінтів + auth) | `COMPLETED` недосяжний: `TS-SM-08`, `TS-SM-14`, `TS-CMP-02`, `TS-GT-17`, complete-частина `TS-SM-09…13` | Описати endpoint у `03` і спосіб доступу на стенді |
| `F-14` | `02` States: для `REJECTED` «`failing_gate` and `rejection_reason` are populated»; `01`: `failing_gate` — лише при відмові гейтом. Що записується в `rejection_reason` при reviewer reject (note?), чи заповнюються `decided_*` при відмові гейтом — не сказано | `02` States vs `01` FlightRequest | `TS-REV-03` | Значення `failing_gate`, `rejection_reason`, `decided_*` для кожного типу відмови |
| `F-15` | Код відповіді на reject без `note` не визначено (`400` «malformed body» чи `422`) | `03` endpoint 16, Error semantics; `01` ReviewAction | `TS-REV-04` (лише факт відмови) | Який статус? |
| `F-16` | Відсутній `X-Reviewer-Id` на approve/reject → `400`, але загальна семантика: «`401` — no session, no reviewer header». Не визначено: порожнє значення заголовка; запит рев'юера на owner-only ендпоінти (`submit`, `cancel`, create) | `03` Reviewer vs Error semantics | `TS-AUTH-14`, `TS-REV-07` | `400` чи `401` для approve/reject без заголовка й без сесії? Що для reviewer на owner-only? |
| `F-17` | Будь-хто може передати `X-Reviewer-Id`, включно з оператором, що схвалює власну заявку; обмежень docs не описують | `03` Authentication history, Reviewer | `TS-AUTH-15` — потенційний неправомірний дозвіл (P1) | Чи прийнятний цей ризик для релізу? |
| `F-18` | `422` на реєстрації дрона > 30 000 г — «not a gate», але контракт `422` вимагає `failing_gate` з чотирьох значень | `06` §1.2, §6 vs `03` The 422 body | `TS-REG-07` | Форма тіла `422` для `/aircraft` |
| `F-19` | Коди відповідей не визначені для: дублікатів (`email`, `national_id`, `serial_number`), неправильних credentials, неіснуючих/чужих `pilot_id`/`aircraft_id`/`zone_code` у create, `flight_purpose` > 2 000, `window_end` ≤ `window_start` на create (`400` чи гейт `TSD-4 §3`), `mtom_grams` ≤ 0 | `01`; `03` Error semantics; `06` §1.4, §3 | `TS-AUTH-03/05`, `TS-REG-11/12`, `TS-FR-04/05/06` | Таблиця кодів для кожного випадку |
| `F-20` | Порядок умов **усередині** гейту не визначений — який `rule_reference`, якщо падають дві умови одного гейту | `06` §1.1, §1.3, §1.4, §6 | `TS-GP-09`, `TS-GZ-07`, `TS-GT-19` | Фіксований порядок умов у гейті? |
| `F-21` | Чи перевіряються гейти повторно на approve (lead time міг стати < 6 год, зона — неактивною) | `02`; `03` endpoint 15; `06` §1.4 | `TS-REV-08` | Approve без повторної перевірки — очікувано? |
| `F-22` | `AuditLog`: немає API для читання; не визначено, чи логується транзитний `SUBMITTED`, які значення `actor`, чи логуються out-of-band зміни статусів інших сутностей | `01` AuditLog; `02` | `TS-AUD-03`, `TS-AUD-06`; потрібен доступ до БД | Очікувані рядки аудиту для submit і значення `actor` |
| `F-23` | Для `SUSPENDED` перелічено лише create/submit/approve; cancel, validate, реєстрація пілотів/дронів, логін не згадані | `05` §1; `07` п.3 | `TS-SUSP-06` | Чи блокуються інші операції? |
| `F-24` | `04` не описує: коли створюється `DRAFT` у формі (а `validate` потребує `id`), кнопку submit, дію cancel у порталі | `04` Operator portal vs `03` endpoints 9, 10, 13, 14 | `TS-UIO-08`; cancel не має UI-кроків | Описати UI-флоу draft → check → submit → cancel |
| `F-25` | `01` каже, що виправлення маси одразу перераховує band, але в `03` немає ендпоінта оновлення дрона | `01` Derived vs `03` Registry | `TS-REG-13` нетестовний | Як виправляється маса? |
| `F-26` | Не визначено: параметри фільтра за діапазоном дат у `GET /flight-requests`; чи потрібна автентифікація для `GET /zones` | `03` endpoints 7, 12 | `TS-FR-09`, `TS-ZREF-05` | Описати параметри й доступ |
| `F-27` | Назва й формат похідного поля currency у `GET /pilots/{id}` і band у `GET /aircraft/{id}` не визначені; currency на `GET` рахується відносно «зараз» | `03` endpoints 4, 6; `01` Derived | `TS-REG-02/04` — перевірка значення без точної назви поля | Схема відповіді |

## 12. Підсумок

| | Всього | Ready | Blocked | Needs clarification |
|---|---|---|---|---|
| Сценарії | 167 | 123 | 4 | 40 |

| Пріоритет | Всього | Ready | Blocked | Needs clarification |
|---|---|---|---|---|
| P1 | 93 | 84 | 3 | 6 |
| P2 | 58 | 33 | 1 | 24 |
| P3 | 16 | 6 | 0 | 10 |

Сценарії зі статусом `Ready (… — F-N)` рахуються як `Ready`: перевіряється визначена частина, невизначена винесена в окремий сценарій або знахідку. Найкритичніші для релізу уточнення: `F-3`/`F-12` (що бачить оператор при відмові), `F-5` (curfew), `F-10`/`F-11` (validate і деталі рев'юера), `F-17` (самосхвалення).
