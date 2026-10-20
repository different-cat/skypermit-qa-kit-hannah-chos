# Тест-план: гейти `PILOT` / `AIRCRAFT` / `ZONE` / `TIME_WINDOW` та ендпоінти `validate` / `submit`

| Атрибут | Значення |
|---|---|
| Проєкт | SkyPermit |
| Тип плану | Фічевий (feature test plan), мануальне тестування |
| Область | Гейтовий пайплайн і ендпоінти `POST /api/v1/flight-requests/{id}/validate`, `POST /api/v1/flight-requests/{id}/submit` |
| Версія | 1.0 (чернетка) |
| Дата | 2026-10-20 |
| Джерела вимог | `../skypermit/docs/01`–`08` (тільки читання), `../skypermit/CLAUDE.md` |

---

## 1. Мета

Перевірити, що:

1. Кожен із чотирьох гейтів приймає і відхиляє заявку точно за правилами з `06-business-rules.md` (з урахуванням виправлень `07-clarifications.md`), **на кожній межі кожного порогу**.
2. Гейти виконуються у фіксованому порядку `PILOT → AIRCRAFT → ZONE → TIME_WINDOW`, і `failing_gate` завжди називає **перший** гейт, що впав.
3. Відповідь `422` має контрактну форму (`failing_gate`, `rule_reference`, `message`) зі значеннями `rule_reference` дослівно з `06-business-rules.md` §6.
4. `validate` ніколи не змінює стан заявки, а `submit` переводить `DRAFT` у `UNDER_REVIEW` або `REJECTED` і зберігає причину відмови.
5. `validate` і `submit` дають однаковий вердикт для однакових вхідних даних (спільний пайплайн), за винятком задокументованого випадку з перетином розкладу.

## 2. Обсяг

### 2.1 У обсязі

| Область | Що перевіряється | Джерело |
|---|---|---|
| Гейт `PILOT` | рейтинг vs `zone_class`; актуальність сертифікації (18 місяців від `certified_on` на момент `window_start`) | `06` §1.1, §2 |
| Гейт `AIRCRAFT` | вагова категорія (`C0`–`C3`, похідна від `mtom_grams`) vs `permitted_bands` зони | `06` §1.2, §3; `07` п.1 |
| Гейт `ZONE` | висота vs `ceiling_metres`; `authorisation_level` оператора vs `required_authorisation_level`; `active` зони | `06` §1.3; `07` п.4 |
| Гейт `TIME_WINDOW` | мінімальний lead time 6 год; максимум 60 днів наперед; тривалість 15–240 хв; комендантська година (curfew) з урахуванням `A-7`; перетин з іншими заявками того ж дрона | `06` §1.4 |
| Порядок гейтів | перший гейт, що впав, при множинних порушеннях | `06` «Evaluation order»; `03` «Gate pipeline» |
| Контракт `422` | поля і значення `rule_reference` | `03` «The 422 body»; `06` §6 |
| `validate` | відсутність побічних ефектів, ідемпотентність, доступ (власник / рев'юер), 401/403/404 | `03` endpoint 13, «Reviewer-or-owner reads» |
| `submit` | переходи стану, збереження `failing_gate`/`rule_reference`/`rejection_reason`, `409` для не-`DRAFT`, доступ | `02`; `03` endpoint 10; `01` FlightRequest |
| Пов'язана межа реєстрації | `mtom_grams` 30 000 / 30 001 г на `POST /aircraft` (не гейт, але верхня межа `C3`) | `06` §1.2, §3 |
| UI | кнопка «Check before submitting», детальна сторінка заявки, відсутність префільтрації пікерів | `04` Operator portal |

### 2.2 Поза обсягом

- Дії рев'юера `approve`/`reject` і скасування (`cancel`), окрім використання їх для підготовки даних (стан `APPROVED`, `CANCELLED`, `REJECTED` для перевірок перетину).
- Реєстрація оператора/пілота поза граничними значеннями, потрібними гейтам.
- Пропозиція `CR-1` і застаріла `OH-1` v1.4 — **не є джерелом** очікуваних результатів (`06`, преамбула). Їхні значення (24 місяці, межа 4 000 г, `C4`/`C5`) використовуються лише як регресійні «пастки» (див. §6.6).
- Навантажувальне тестування, безпека поза перевірками 401/403.

## 3. Пріоритет джерел (як застосовано в цьому плані)

- `07-clarifications.md` > `06-business-rules.md`: перевірка ваги належить гейту `AIRCRAFT`, а не `ZONE`, попри дослівний текст `ZC-2026.1` §2.
- Статус відкритих питань — тільки з `05-open-questions.md`: відкриті рядки 4 (`SUSPENDED`) і 5 (цитата для `active` зони).
- `08-uml-diagrams.md` використовується лише як допоміжний; де він розходиться з 02/03 — див. §9.

---

## 4. Тестове середовище та дані

### 4.1 Передумови середовища

| # | Передумова | Чому |
|---|---|---|
| ENV-1 | Розгорнуте тестове середовище SkyPermit з застосованими міграціями і `seed.sql` (шість зон з `06` §4) | Каталог зон — референсні дані |
| ENV-2 | Можливість задати оператору `authorisation_level` `L2`/`L3` позаштатною адмін-дією | Реєстрація завжди створює `L1` (`03` «authorisation_level is no longer self-service») |
| ENV-3 | Можливість отримати зону з `active = false` у тестовому середовищі (окрема копія seed / адмін-дія) | Усі шість зон активні (`06` §4); без цього кейси `active` не виконуються |
| ENV-4 | Можливість задати `Operator.status = SUSPENDED` | Для кейсів, заблокованих відкритим питанням 4 |
| ENV-5 | Відомий часовий пояс зон (див. OQ-1) і синхронізований годинник клієнта з сервером (NTP), допуск ±1 хв | Межі lead time / 60 днів / curfew чутливі до часу виклику |
| ENV-6 | Доступ на читання до `AuditLog` і полів `FlightRequest` (через `GET /flight-requests/{id}` або БД тестового середовища) | Перевірка відсутності побічних ефектів `validate` |

### 4.2 Позначення часу

- `T` — момент виклику `validate`/`submit` (час сервера, UTC).
- `WS` / `WE` — `window_start` / `window_end` (UTC).
- `LOC(x)` — локальний час зони для моменту `x` (зміщення — див. OQ-1).
- «Хвилина» — найменша одиниця граничних значень у плані; поведінка на рівні секунд не задокументована (OQ-6).

### 4.3 Базова «зелена» заявка `BASE`

Усі граничні кейси будуються як **одна зміна** від `BASE`, яка проходить усі чотири гейти в будь-якій із шести зон:

| Поле | Значення `BASE` | Чому проходить |
|---|---|---|
| Оператор | `OP-L3` (`ACTIVE`, `L3`) | `L3` ≥ будь-якого `required_authorisation_level` |
| Пілот | `P-BVLOS` (`R_BVLOS`, `certified_on` = сьогодні − 1 місяць, `ACTIVE`) | `R_BVLOS` покриває всі класи; сертифікація актуальна |
| Дрон | `AC-500` (`mtom_grams` = 500 → `C0`, `ACTIVE`) | `C0` дозволено в усіх зонах |
| Висота | 30 м | ≤ найнижчої стелі (45 м, `ZTM-RST-7`) |
| `WS` | `T` + 2 доби, 12:00 за `LOC` | Поза будь-яким curfew; 6 год < lead < 60 днів |
| Тривалість | 60 хв | У межах 15–240 |
| Інші заявки цього дрона | немає `APPROVED`/`UNDER_REVIEW` | Немає перетину |
| `flight_purpose` | "QA test" | < 2 000 символів |

### 4.4 Довідник тестових сутностей

**Оператори:** `OP-L1`, `OP-L2`, `OP-L3` (усі `ACTIVE`), `OP-SUSP` (`SUSPENDED`, `L3`), `OP-OTHER` (`L3`, для перевірок 403).

**Пілоти (під `OP-L3`, якщо не зазначено інше):**

| ID | `rating` | `certified_on` | `status` |
|---|---|---|---|
| `P-BASIC` | `R_BASIC` | сьогодні − 1 міс | `ACTIVE` |
| `P-ADV` | `R_ADVANCED` | сьогодні − 1 міс | `ACTIVE` |
| `P-BVLOS` | `R_BVLOS` | сьогодні − 1 міс | `ACTIVE` |
| `P-CUR-x` | `R_BVLOS` | обчислюється в кожному кейсі §6.1.2 | `ACTIVE` |
| `P-INACTIVE` | `R_BVLOS` | сьогодні − 1 міс | `INACTIVE` |

Пілоти і дрони з тими самими параметрами під `OP-L1`/`OP-L2` створюються за потреби кейсу (матриця `authorisation_level`).

**Дрони (під `OP-L3`):**

| ID | `mtom_grams` | Очікувана категорія (`06` §3) |
|---|---|---|
| `AC-1` | 1 | `C0` |
| `AC-500` | 500 | `C0` |
| `AC-900` | 900 | `C0` (верхня межа включно) |
| `AC-901` | 901 | `C1` |
| `AC-3500` | 3 500 | `C1` (верхня межа включно) |
| `AC-3501` | 3 501 | `C2` |
| `AC-12000` | 12 000 | `C2` (верхня межа включно) |
| `AC-12001` | 12 001 | `C3` |
| `AC-30000` | 30 000 | `C3` (верхня межа включно) |
| `AC-RETIRED` | 500, `status = RETIRED` | `C0` |

Перед використанням кожного дрона переконатися через `GET /aircraft/{id}`, що похідна категорія відповідає таблиці (`03` endpoint 6).

---

## 5. Підхід і техніки

| Техніка | Де застосовується |
|---|---|
| Аналіз граничних значень (BVA, 3-точковий: межа − 1, межа, межа + 1) | вага, висота, lead time, 60 днів, тривалість, curfew, перетин, 18 місяців |
| Класи еквівалентності | рейтинг × клас зони, рівень авторизації × вимога зони, стан конкуруючої заявки |
| Таблиці рішень | порядок гейтів при множинних порушеннях |
| Тестування переходів стану | `submit` з кожного з 7 станів |
| Порівняльне (differential) тестування | однакові дані → `validate` і `submit` дають однаковий вердикт |

**Правило виконання кожного кейсу гейта:** якщо не вказано інше, кейс виконується **двічі** — через `validate` (очікується той самий вердикт, стан лишається `DRAFT`) і через `submit` на свіжій `DRAFT`-копії (очікується відповідний перехід стану). Це обов'язкова вимога «single shared function» (`03` endpoint 13).

**Очікуваний результат при відмові гейта, якщо не вказано інше:**
- `submit` → `422`, тіло `{failing_gate, rule_reference, message}`; заявка в `REJECTED`; `failing_gate`, `rule_reference`, `rejection_reason` заповнені і повертаються в `GET /flight-requests/{id}` (`01` FlightRequest; `02` States; `03` «The 422 body»).
- `validate` → вердикт «fail» з тими самими `failing_gate`/`rule_reference`; стан не змінюється. Точна форма відповіді `validate` (код і тіло) — див. OQ-2.

**Очікуваний результат при проходженні:** `submit` → заявка в `UNDER_REVIEW`, `submitted_at` заповнено, `failing_gate`/`rule_reference`/`rejection_reason` порожні; `validate` → усі гейти «pass», стан `DRAFT`.

**Статуси кейсів:** `Ready` — очікуваний результат повністю визначено документацією; `Needs clarification` — частину очікуваного результату docs не визначають (посилання на OQ), кейс виконується, але цей аспект фіксується як спостереження, а не pass/fail; `Blocked` — виконання неможливе до відповіді на відкрите питання.

**Пріоритети:** P1 — критичні межі і контракт; P2 — додаткові межі і комбінації; P3 — регресійні «пастки» та UI.

---

## 6. Тест-кейси

### 6.1 Гейт `PILOT`

#### 6.1.1 Рейтинг vs клас зони (`06` §1.1, §4; `rule_reference` = `OH-1 §2.1`)

Повна матриця 3 × 3. Зона: `OPEN` = `DNI-OPN-1`, `CONTROLLED` = `KBP-CTR-3`, `RESTRICTED` = `LVV-RST-2`. Решта — `BASE`.

| ID | Пілот | Зона (клас) | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|---|
| GP-PIL-001 | `P-BASIC` | `DNI-OPN-1` (`OPEN`) | pass | P1 | Ready |
| GP-PIL-002 | `P-BASIC` | `KBP-CTR-3` (`CONTROLLED`) | fail `PILOT`, `OH-1 §2.1` | P1 | Ready |
| GP-PIL-003 | `P-BASIC` | `LVV-RST-2` (`RESTRICTED`) | fail `PILOT`, `OH-1 §2.1` | P1 | Ready |
| GP-PIL-004 | `P-ADV` | `DNI-OPN-1` (`OPEN`) | pass | P2 | Ready |
| GP-PIL-005 | `P-ADV` | `KBP-CTR-3` (`CONTROLLED`) | pass | P1 | Ready |
| GP-PIL-006 | `P-ADV` | `LVV-RST-2` (`RESTRICTED`) | fail `PILOT`, `OH-1 §2.1` | P1 | Ready |
| GP-PIL-007 | `P-BVLOS` | `DNI-OPN-1` (`OPEN`) | pass | P2 | Ready |
| GP-PIL-008 | `P-BVLOS` | `KBP-CTR-3` (`CONTROLLED`) | pass | P2 | Ready |
| GP-PIL-009 | `P-BVLOS` | `LVV-RST-2` (`RESTRICTED`) | pass | P1 | Ready |
| GP-PIL-010 | `P-BASIC` | `ODS-CTR-5`, `ZTM-RST-7`, `KHR-OPN-4` (по одному прогону) | `ODS` fail, `ZTM` fail, `KHR` pass (`OH-1 §2.1`) | P2 | Ready |

Примітка: для GP-PIL-002/003/006 решта гейтів проходить завдяки `BASE` (`OP-L3`, `C0`, 30 м), тому падіння саме `PILOT` однозначне.

#### 6.1.2 Актуальність сертифікації (`06` §1.1, §2; `rule_reference` = `OH-1 §2.3`)

Позначимо `E` = `certified_on` + 18 календарних місяців. Правило: актуальна 18 місяців, без пільгового періоду, «one day past the 18-month mark is not current», перевіряється на `WS`, а не на `T`.

| ID | Умова (`certified_on` відносно дати `WS`) | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-PIL-020 | `E` = дата `WS` + 1 день (до межі лишається 1 день) | pass | P1 | Ready |
| GP-PIL-021 | `E` = дата `WS` (рівно на 18-місячній позначці) | не визначено: чи є день `E` ще актуальним | P1 | Needs clarification (OQ-3) |
| GP-PIL-022 | `E` = дата `WS` − 1 день (один день після позначки) | fail `PILOT`, `OH-1 §2.3` | P1 | Ready |
| GP-PIL-023 | `certified_on` = дата `WS` − 18 міс − 1 рік (давно прострочено) | fail `PILOT`, `OH-1 §2.3` | P2 | Ready |
| GP-PIL-024 | **Протухає між `T` і `WS`:** `E` = `T` + 1 день, `WS` = `T` + 3 доби | fail `PILOT`, `OH-1 §2.3` (актуальна на момент подання, але не на `WS`) | P1 | Ready |
| GP-PIL-025 | Щойно сертифікований: `certified_on` = сьогодні | pass | P3 | Ready |
| GP-PIL-026 | Регрес `OH-1` v1.4 (24 місяці): `certified_on` = дата `WS` − 20 місяців | fail `PILOT`, `OH-1 §2.3` (якщо pass — реалізовано застаріле правило 24 міс) | P2 | Ready |
| GP-PIL-027 | Кінець місяця: `certified_on` = 2025-08-31, `WS` = 2027-02-28 / 2027-03-01 | не визначено арифметику «+18 місяців» для 31-го числа | P3 | Needs clarification (OQ-3) |
| GP-PIL-028 | `certified_on` у майбутньому відносно `WS` | не визначено (чи це «не актуальна», чи помилка реєстрації) | P3 | Needs clarification (OQ-3) |
| GP-PIL-029 | `GET /pilots/{id}` для пілотів з GP-PIL-020/022 | похідний статус актуальності повертається; він обчислюється на момент читання і не є збереженим полем (`01` «Derived, never stored») | P2 | Ready |

Примітка до GP-PIL-020/022: порівняння `date` (`certified_on`) з `timestamptz` (`WS`, UTC) — у якому часовому поясі береться «дата `WS`» не визначено (OQ-3). Щоб уникнути впливу, у кейсах 020/022 обирати `WS` о 12:00 UTC.

#### 6.1.3 Незалежність умов і статус пілота

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-PIL-040 | `P-BASIC` прострочений (обидві умови `PILOT` порушені) у `KBP-CTR-3` | fail `PILOT`; яке з двох `rule_reference` (`OH-1 §2.1` чи `§2.3`) — не визначено | P2 | Needs clarification (OQ-4) |
| GP-PIL-041 | `P-INACTIVE` у `BASE` | pass — `Pilot.status` не прив'язаний до жодного гейта (`05` рядок 2, `07` п.2) | P2 | Ready |
| GP-PIL-042 | `P-BASIC` у `KBP-CTR-3` + оператор `OP-L3` (рівень зони виконано) | fail `PILOT` — рейтинг перевіряється незалежно від гейта `ZONE` (`06` §1.1) | P2 | Ready |

### 6.2 Гейт `AIRCRAFT` (`06` §1.2, §3; `07` п.1; `rule_reference` = `ZC-2026.1 §2`)

#### 6.2.1 Граничні значення ваги × дозволені категорії зони

Кожна межа категорії перевіряється в зоні, де вона є межею дозволу. Пілот `P-BVLOS`, оператор `OP-L3`, висота 30 м.

| ID | Зона (`permitted_bands`) | Дрон (`mtom_grams` → категорія) | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|---|
| GP-AIR-001 | `ZTM-RST-7` (`C0`) | `AC-1` (1 → `C0`) | pass | P2 | Ready |
| GP-AIR-002 | `ZTM-RST-7` (`C0`) | `AC-900` (900 → `C0`) | pass | P1 | Ready |
| GP-AIR-003 | `ZTM-RST-7` (`C0`) | `AC-901` (901 → `C1`) | fail `AIRCRAFT`, `ZC-2026.1 §2` | P1 | Ready |
| GP-AIR-004 | `LVV-RST-2` (`C0`–`C1`) | `AC-901` (901 → `C1`) | pass | P2 | Ready |
| GP-AIR-005 | `LVV-RST-2` (`C0`–`C1`) | `AC-3500` (3 500 → `C1`) | pass | P1 | Ready |
| GP-AIR-006 | `LVV-RST-2` (`C0`–`C1`) | `AC-3501` (3 501 → `C2`) | fail `AIRCRAFT`, `ZC-2026.1 §2` | P1 | Ready |
| GP-AIR-007 | `KBP-CTR-3` (`C0`–`C2`) | `AC-3501` (3 501 → `C2`) | pass | P2 | Ready |
| GP-AIR-008 | `KBP-CTR-3` (`C0`–`C2`) | `AC-12000` (12 000 → `C2`) | pass | P1 | Ready |
| GP-AIR-009 | `KBP-CTR-3` (`C0`–`C2`) | `AC-12001` (12 001 → `C3`) | fail `AIRCRAFT`, `ZC-2026.1 §2` | P1 | Ready |
| GP-AIR-010 | `ODS-CTR-5` (`C0`–`C2`) | `AC-12000` / `AC-12001` | pass / fail `AIRCRAFT` | P2 | Ready |
| GP-AIR-011 | `DNI-OPN-1` (`C0`–`C3`) | `AC-12001` (12 001 → `C3`) | pass | P2 | Ready |
| GP-AIR-012 | `DNI-OPN-1` (`C0`–`C3`) | `AC-30000` (30 000 → `C3`) | pass | P1 | Ready |
| GP-AIR-013 | `KHR-OPN-4` (`C0`–`C3`) | `AC-30000` | pass | P2 | Ready |
| GP-AIR-014 | `ZTM-RST-7` (`C0`) | `AC-30000` (`C3`, найдальша від дозволеної) | fail `AIRCRAFT`, `ZC-2026.1 §2` | P2 | Ready |

#### 6.2.2 Правильність гейта (виправлення `07` п.1) і пов'язані правила

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-AIR-020 | Повтор GP-AIR-003: перевірити саме значення `failing_gate` | `AIRCRAFT`, **не** `ZONE` (якщо `ZONE` — реалізовано дослівний помилковий текст `ZC-2026.1` §2) | P1 | Ready |
| GP-AIR-021 | Вага не дозволена **і** висота понад стелю (`ZTM-RST-7`, `AC-901`, 46 м) | fail `AIRCRAFT` (іде раніше за `ZONE`); обидві причини мають однаковий `rule_reference` `ZC-2026.1 §2`, розрізняються лише `failing_gate` (`06` §1.2 примітка) | P1 | Ready |
| GP-AIR-022 | `AC-RETIRED` у `BASE` | pass — `Aircraft.status` не прив'язаний до гейта (`05` рядок 2) | P2 | Ready |
| GP-AIR-023 | Регрес `OH-1` v1.4 (межа `C1`/`C2` = 4 000 г): `AC-3501` у `LVV-RST-2`; додатково дрон 4 000 г у `LVV-RST-2` | обидва fail `AIRCRAFT` (3 501 і 4 000 г — `C2`); pass означає застарілу межу | P2 | Ready |
| GP-AIR-024 | `GET /aircraft/{id}` для всіх дронів §4.4 | похідна категорія відповідає таблиці; поле не зберігається (`01`) | P1 | Ready |

#### 6.2.3 Верхня межа реєстрації (не гейт; `06` §1.2, §3; `rule_reference` = `OH-1 §3`)

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-AIR-030 | `POST /aircraft`, `mtom_grams` = 30 000 | дрон зареєстровано, категорія `C3` | P1 | Ready |
| GP-AIR-031 | `POST /aircraft`, `mtom_grams` = 30 001 | `422`, дрон не створено; форма тіла (яке `failing_gate`, якщо поле обов'язкове) — не визначено | P1 | Needs clarification (OQ-5) |
| GP-AIR-032 | `POST /aircraft`, `mtom_grams` = 0 / від'ємне / дробове | не визначено (нижня межа `C0` = 0 г, але валідність 0 і типу не описано) | P3 | Needs clarification (OQ-6) |

### 6.3 Гейт `ZONE` (`06` §1.3; `07` п.4)

#### 6.3.1 Висота vs стеля (`rule_reference` = `ZC-2026.1 §2`)

Правило: `requested_altitude_metres` ≤ `ceiling_metres`. Пілот `P-BVLOS`, дрон `AC-500`, `OP-L3`.

| ID | Зона (стеля) | Висота, м | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|---|
| GP-ZON-001 | `ZTM-RST-7` (45) | 44 | pass | P2 | Ready |
| GP-ZON-002 | `ZTM-RST-7` (45) | 45 | pass | P1 | Ready |
| GP-ZON-003 | `ZTM-RST-7` (45) | 46 | fail `ZONE`, `ZC-2026.1 §2` | P1 | Ready |
| GP-ZON-004 | `LVV-RST-2` (60) | 59 / 60 / 61 | pass / pass / fail `ZONE` | P1 | Ready |
| GP-ZON-005 | `ODS-CTR-5` (90) | 89 / 90 / 91 | pass / pass / fail `ZONE` | P1 | Ready |
| GP-ZON-006 | `KBP-CTR-3` (120) | 119 / 120 / 121 | pass / pass / fail `ZONE` | P1 | Ready |
| GP-ZON-007 | `DNI-OPN-1` (120) | 120 / 121 | pass / fail `ZONE` | P2 | Ready |
| GP-ZON-008 | `KHR-OPN-4` (120) | 120 / 121 | pass / fail `ZONE` | P2 | Ready |
| GP-ZON-009 | `ZTM-RST-7` (45) | 45.5 (дробове) | не визначено: тип поля (ціле/дробове) не описано | P3 | Needs clarification (OQ-6) |
| GP-ZON-010 | будь-яка | 0, −1 | не визначено: нижня межа висоти не описана | P3 | Needs clarification (OQ-6) |

#### 6.3.2 Рівень авторизації оператора (`rule_reference` = `OH-1 §4`)

Правило: `authorisation_level` оператора ≥ `required_authorisation_level` зони, `L1 < L2 < L3`. Пілот `R_BVLOS`, дрон `C0` — під тим самим оператором.

| ID | Оператор | Зона (вимога) | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|---|
| GP-ZON-020 | `OP-L1` | `DNI-OPN-1` (`L1`) | pass (рівно межа) | P1 | Ready |
| GP-ZON-021 | `OP-L1` | `KBP-CTR-3` (`L2`) | fail `ZONE`, `OH-1 §4` | P1 | Ready |
| GP-ZON-022 | `OP-L1` | `LVV-RST-2` (`L3`) | fail `ZONE`, `OH-1 §4` | P2 | Ready |
| GP-ZON-023 | `OP-L2` | `KHR-OPN-4` (`L1`) | pass | P2 | Ready |
| GP-ZON-024 | `OP-L2` | `ODS-CTR-5` (`L2`) | pass (рівно межа) | P1 | Ready |
| GP-ZON-025 | `OP-L2` | `ZTM-RST-7` (`L3`) | fail `ZONE`, `OH-1 §4` | P1 | Ready |
| GP-ZON-026 | `OP-L3` | `DNI-OPN-1` (`L1`) | pass | P2 | Ready |
| GP-ZON-027 | `OP-L3` | `KBP-CTR-3` (`L2`) | pass | P2 | Ready |
| GP-ZON-028 | `OP-L3` | `LVV-RST-2` (`L3`) | pass (рівно межа) | P1 | Ready |
| GP-ZON-029 | Новозареєстрований оператор (завжди `L1`, `03`) з пілотом `R_BVLOS` у `KBP-CTR-3` | fail `ZONE`, `OH-1 §4` | P2 | Ready |

#### 6.3.3 Активність зони (`rule_reference` = `ZC-2026.1 §4`, цитата не підтверджена)

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-ZON-040 | `BASE` у зоні з `active = false` (ENV-3) | fail `ZONE` (`05` §2: правило встановлене); `rule_reference` = `ZC-2026.1 §4` фіксується як спостереження, не як критерій pass/fail | P1 | Needs clarification (OQ `05` рядок 5) |
| GP-ZON-041 | Неактивна зона + висота понад стелю | fail `ZONE`; яке з двох `rule_reference` — не визначено | P3 | Needs clarification (OQ-4, OQ `05` рядок 5) |

#### 6.3.4 Комбінації всередині `ZONE`

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-ZON-050 | `OP-L1` у `KBP-CTR-3` + висота 121 м | fail `ZONE`; порядок умов усередині гейта не визначено → `rule_reference` (`ZC-2026.1 §2` чи `OH-1 §4`) фіксується | P2 | Needs clarification (OQ-4) |
| GP-ZON-051 | `OP-L3`, `P-ADV`, `AC-500`, `KBP-CTR-3`, 120 м | pass (усі межі виконано одночасно) | P1 | Ready |

### 6.4 Гейт `TIME_WINDOW` (`06` §1.4)

#### 6.4.1 Мінімальний lead time ≥ 6 год (`rule_reference` = `TSD-4 §2.1`)

Вимір: `WS − T`, де `T` — момент подання. Для зменшення впливу затримки — значення ±1 хв від межі; точно на межі — з допуском ENV-5.

| ID | `WS − T` | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-TW-001 | 5 год 59 хв | fail `TIME_WINDOW`, `TSD-4 §2.1` | P1 | Ready |
| GP-TW-002 | 6 год 00 хв (≥, межа включно) | pass | P1 | Ready (точність див. OQ-6) |
| GP-TW-003 | 6 год 01 хв | pass | P1 | Ready |
| GP-TW-004 | 0 хв (`WS` = `T`) | fail `TIME_WINDOW`, `TSD-4 §2.1` | P2 | Ready |
| GP-TW-005 | `WS` у минулому (`T` − 1 год) | fail `TIME_WINDOW`, `TSD-4 §2.1` | P2 | Ready |
| GP-TW-006 | Чернетку створено при `WS − now` = 7 год, `submit` виконано при `WS − T` = 5 год 59 хв | fail `TIME_WINDOW` — рахується від подання, а не від створення | P1 | Ready |
| GP-TW-007 | Те саме, але через `validate` | не визначено, від якого моменту `validate` рахує lead time (час виклику `validate`?) | P2 | Needs clarification (OQ-7) |

#### 6.4.2 Максимум 60 днів наперед (`rule_reference` = `TSD-4 §2.2`)

| ID | `WS − T` | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-TW-010 | 59 діб 23 год 59 хв | pass | P1 | Ready |
| GP-TW-011 | рівно 60 діб (≤, межа включно) | pass | P1 | Needs clarification (OQ-8: календарні дні vs 60 × 24 год; перехід DST) |
| GP-TW-012 | 60 діб 00 год 01 хв | fail `TIME_WINDOW`, `TSD-4 §2.2` | P1 | Needs clarification (OQ-8) |
| GP-TW-013 | 61 доба | fail `TIME_WINDOW`, `TSD-4 §2.2` | P1 | Ready |
| GP-TW-014 | 365 діб | fail `TIME_WINDOW`, `TSD-4 §2.2` | P3 | Ready |

#### 6.4.3 Тривалість 15–240 хв включно; `WE` строго після `WS` (`rule_reference` = `TSD-4 §3`)

`WS` як у `BASE`, змінюється лише `WE`.

| ID | `WE − WS` | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-TW-020 | 14 хв | fail `TIME_WINDOW`, `TSD-4 §3` | P1 | Ready |
| GP-TW-021 | 15 хв | pass | P1 | Ready |
| GP-TW-022 | 16 хв | pass | P2 | Ready |
| GP-TW-023 | 239 хв | pass | P2 | Ready |
| GP-TW-024 | 240 хв | pass | P1 | Ready |
| GP-TW-025 | 241 хв | fail `TIME_WINDOW`, `TSD-4 §3` | P1 | Ready |
| GP-TW-026 | 14 хв 59 с / 240 хв 01 с | не визначено (точність до секунд) | P3 | Needs clarification (OQ-6) |
| GP-TW-027 | 0 хв (`WE` = `WS`) | відмова; не визначено, на якому кроці: `400`/відмова при створенні (`01`: «must be after `window_start`») чи `422 TIME_WINDOW` при `validate`/`submit` | P1 | Needs clarification (OQ-9) |
| GP-TW-028 | −30 хв (`WE` < `WS`) | як GP-TW-027 | P2 | Needs clarification (OQ-9) |

#### 6.4.4 Комендантська година (curfew) (`rule_reference` = `A-7 §1` для `KBP-CTR-3`/`ODS-CTR-5`, `ZC-2026.1 §3` для інших)

Чинні значення (`06` §1.4): `KBP-CTR-3`, `ODS-CTR-5` — 23:00–05:00 (за `A-7`, перекриває базові 22:00–06:00); `LVV-RST-2`, `ZTM-RST-7` — 22:00–06:00; `DNI-OPN-1`, `KHR-OPN-4` — немає. Час вікна в таблиці — **локальний час зони**; у запиті передається відповідне UTC-значення (зміщення — OQ-1). Решта — `BASE` з придатним для зони пілотом/оператором (для `ZTM`/`LVV` — `P-BVLOS`, `OP-L3`, висота ≤ 45 м).

**`KBP-CTR-3` (23:00–05:00, `A-7 §1`)** — аналогічний прогін повторити для `ODS-CTR-5` (GP-TW-049).

| ID | Вікно (локальний час) | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-TW-030 | 21:30–22:30 | pass | P2 | Ready |
| GP-TW-031 | 22:00–23:00 (закінчується рівно на початку curfew) | не визначено, чи хвилина 23:00 входить у curfew | P1 | Needs clarification (OQ-10) |
| GP-TW-032 | 22:00–23:01 (заходить на 1 хв) | fail `TIME_WINDOW`, `A-7 §1` | P1 | Ready |
| GP-TW-033 | 22:15–22:59 | pass | P1 | Ready |
| GP-TW-034 | 23:30–00:30 (через північ, усередині) | fail `TIME_WINDOW`, `A-7 §1` — curfew через північ є одним періодом | P1 | Ready |
| GP-TW-035 | 01:00–02:00 (повністю всередині) | fail `TIME_WINDOW`, `A-7 §1` | P2 | Ready |
| GP-TW-036 | 04:59–05:30 (починається за 1 хв до кінця) | fail `TIME_WINDOW`, `A-7 §1` | P1 | Ready |
| GP-TW-037 | 05:00–05:30 (починається рівно на кінці curfew) | не визначено, чи хвилина 05:00 входить у curfew | P1 | Needs clarification (OQ-10) |
| GP-TW-038 | 05:01–05:30 | pass | P1 | Ready |
| GP-TW-039 | **Перевірка `A-7` vs база:** 22:00–22:45 | pass (за базою `ZC-2026.1` 22:00–06:00 був би fail) | P1 | Ready |
| GP-TW-040 | **Перевірка `A-7` vs база:** 05:15–05:59 | pass (за базою — fail) | P1 | Ready |
| GP-TW-041 | 21:00–01:00 (240 хв, перекриває початок curfew і північ) | fail `TIME_WINDOW`, `A-7 §1` | P2 | Ready |

**`LVV-RST-2` (22:00–06:00, `ZC-2026.1 §3`)** — аналогічно для `ZTM-RST-7` (GP-TW-058).

| ID | Вікно (локальний час) | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-TW-050 | 21:00–21:59 | pass | P2 | Ready |
| GP-TW-051 | 21:00–22:00 (кінець рівно на початку) | не визначено | P1 | Needs clarification (OQ-10) |
| GP-TW-052 | 21:00–22:01 | fail `TIME_WINDOW`, `ZC-2026.1 §3` | P1 | Ready |
| GP-TW-053 | 22:00–22:45 (те саме вікно, що pass у `KBP` — GP-TW-039) | fail `TIME_WINDOW`, `ZC-2026.1 §3` | P1 | Ready |
| GP-TW-054 | 23:30–00:30 (через північ) | fail `TIME_WINDOW`, `ZC-2026.1 §3` | P2 | Ready |
| GP-TW-055 | 05:59–06:30 | fail `TIME_WINDOW`, `ZC-2026.1 §3` | P1 | Ready |
| GP-TW-056 | 06:00–06:30 (початок рівно на кінці) | не визначено | P1 | Needs clarification (OQ-10) |
| GP-TW-057 | 06:01–06:30 | pass | P1 | Ready |
| GP-TW-058 | Повтор GP-TW-052/055/057 для `ZTM-RST-7` | ті самі вердикти, `ZC-2026.1 §3` | P2 | Ready |
| GP-TW-049 | Повтор GP-TW-032/034/036/038/039 для `ODS-CTR-5` | ті самі вердикти, `A-7 §1` | P2 | Ready |

**Зони без curfew і перетворення часу**

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-TW-060 | `DNI-OPN-1`, вікно 00:00–04:00 локального | pass | P1 | Ready |
| GP-TW-061 | `KHR-OPN-4`, вікно 23:00–03:00 локального | pass | P2 | Ready |
| GP-TW-062 | `KBP-CTR-3`, вікно, що в **UTC** поза 23:00–05:00, але в локальному часі всередині (наприклад, UTC 20:30–21:30 при зміщенні +2/+3) | fail — curfew оцінюється в локальному часі зони, а не в UTC | P1 | Needs clarification (OQ-1: конкретне зміщення) |
| GP-TW-063 | Вікно, що в UTC всередині 23:00–05:00, але в локальному поза ним | pass | P1 | Needs clarification (OQ-1) |
| GP-TW-064 | Вікно в ніч переходу на зимовий/літній час | не визначено (часовий пояс і правило DST не задокументовані) | P3 | Needs clarification (OQ-1) |
| GP-TW-065 | «Minute-by-minute»: вікно з обома кінцями поза curfew, але серединою всередині | з чинними даними **неможливо** (найкоротший curfew 6 год > максимальної тривалості 240 хв); фіксується як непокрите, перевіряється лише в unit-рівні розробки | P3 | Blocked (обмеження даних) |

#### 6.4.5 Перетин з іншими заявками того ж дрона (`rule_reference` = `TSD-4 §5`)

Передумова: заявка `R0` для того ж дрона `AC-500`, вікно 12:00–13:00 (UTC, дата як у `BASE`), у заданому стані. Тестова заявка `R1` — `BASE` з вікном нижче. Правило: перетин з `APPROVED`/`UNDER_REVIEW` → fail; дотик на межі не є перетином; `DRAFT`, `REJECTED`, `CANCELLED`, `COMPLETED` не враховуються.

| ID | Стан `R0` | Вікно `R1` | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|---|
| GP-TW-070 | `APPROVED` | 13:00–14:00 (дотик: `R1.WS` = `R0.WE`) | pass | P1 | Ready |
| GP-TW-071 | `APPROVED` | 11:00–12:00 (дотик: `R1.WE` = `R0.WS`) | pass | P1 | Ready |
| GP-TW-072 | `APPROVED` | 12:59–14:00 (перетин 1 хв) | fail `TIME_WINDOW`, `TSD-4 §5` | P1 | Needs clarification (OQ-11: `422 TIME_WINDOW` чи `409`) |
| GP-TW-073 | `APPROVED` | 11:00–12:01 (перетин 1 хв) | fail, як GP-TW-072 | P1 | Needs clarification (OQ-11) |
| GP-TW-074 | `APPROVED` | 12:15–12:45 (усередині `R0`) | fail, як GP-TW-072 | P2 | Needs clarification (OQ-11) |
| GP-TW-075 | `APPROVED` | 11:00–14:00 (охоплює `R0`) | fail, як GP-TW-072 | P2 | Needs clarification (OQ-11) |
| GP-TW-076 | `APPROVED` | 12:00–13:00 (ідентичне) | fail, як GP-TW-072 | P2 | Needs clarification (OQ-11) |
| GP-TW-077 | `UNDER_REVIEW` | 12:30–13:30 | fail, як GP-TW-072 | P1 | Needs clarification (OQ-11) |
| GP-TW-078 | `UNDER_REVIEW` | 13:00–14:00 (дотик) | pass | P2 | Ready |
| GP-TW-079 | `DRAFT` | 12:30–13:30 | pass (не враховується) | P1 | Ready |
| GP-TW-080 | `REJECTED` | 12:30–13:30 | pass | P1 | Ready |
| GP-TW-081 | `CANCELLED` | 12:30–13:30 | pass | P1 | Ready |
| GP-TW-082 | `COMPLETED` | 12:30–13:30 | pass | P2 | Blocked (OQ-12: ендпоінт maintenance-completion відсутній у контракті) |
| GP-TW-083 | `APPROVED`, **інший** дрон, той самий пілот | 12:30–13:30 | pass — перевіряється лише той самий дрон | P2 | Ready |
| GP-TW-084 | `R1` пройшов `validate` (стан `DRAFT`); потім `R0` переведено в `UNDER_REVIEW`/`APPROVED` з перетином; `submit R1` | `validate` — pass, `submit` — fail; очікувана задокументована поведінка, не дефект (`02` «Aircraft schedule overlap») | P1 | Needs clarification (OQ-11 щодо коду) |
| GP-TW-085 | `validate` для заявки, яка сама в `UNDER_REVIEW` (перевірка, що заявка не «перетинається сама з собою») | не визначено, чи дозволено `validate` поза `DRAFT` і чи виключається сама заявка | P3 | Needs clarification (OQ-13) |

#### 6.4.6 Комбінації всередині `TIME_WINDOW`

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-TW-090 | Тривалість 241 хв + curfew (`KBP`, 21:00–01:01) | fail `TIME_WINDOW`; `rule_reference` (`TSD-4 §3` чи `A-7 §1`) — порядок умов не визначено | P2 | Needs clarification (OQ-4) |
| GP-TW-091 | Lead time 5 год 59 хв + перетин | fail `TIME_WINDOW`; `rule_reference` не визначено | P3 | Needs clarification (OQ-4) |
| GP-TW-092 | Кілька меж одночасно на «зеленому» боці: `DNI-OPN-1`, `WS` = `T` + 6 год 01 хв, тривалість 240 хв, висота 120 м, `OP-L1` з власними пілотом `R_BASIC` і дроном 30 000 г | pass | P2 | Ready |

### 6.5 Порядок гейтів і перший `failing_gate` (`06` «Evaluation order»; `03` «Gate pipeline»)

Таблиця рішень: ✗ — гейт порушено, ✓ — виконано.

| ID | `PILOT` | `AIRCRAFT` | `ZONE` | `TIME_WINDOW` | Конкретні дані | Очікуваний `failing_gate` | Пріоритет | Статус |
|---|---|---|---|---|---|---|---|---|
| GP-ORD-001 | ✗ | ✗ | ✗ | ✗ | `P-BASIC`, `AC-901`, `ZTM-RST-7`, `OP-L1`, 46 м, тривалість 14 хв | `PILOT` | P1 | Ready |
| GP-ORD-002 | ✗ | ✗ | ✓ | ✓ | `P-BASIC`, `AC-901`, `ZTM-RST-7`, `OP-L3`, 30 м | `PILOT` | P1 | Ready |
| GP-ORD-003 | ✓ | ✗ | ✗ | ✗ | `P-BVLOS`, `AC-901`, `ZTM-RST-7`, 46 м, 241 хв | `AIRCRAFT` | P1 | Ready |
| GP-ORD-004 | ✓ | ✓ | ✗ | ✗ | `P-BVLOS`, `AC-500`, `ZTM-RST-7`, 46 м, 14 хв | `ZONE` | P1 | Ready |
| GP-ORD-005 | ✓ | ✓ | ✓ | ✗ | `BASE` з 14 хв | `TIME_WINDOW` | P1 | Ready |
| GP-ORD-006 | ✗ | ✓ | ✓ | ✗ | `P-BASIC` у `KBP-CTR-3`, 241 хв | `PILOT` | P2 | Ready |
| GP-ORD-007 | ✓ | ✗ | ✓ | ✗ | `AC-12001` у `KBP-CTR-3`, 241 хв | `AIRCRAFT` | P2 | Ready |
| GP-ORD-008 | ✗ | ✓ | ✗ | ✓ | `P-ADV` у `LVV-RST-2`, `OP-L2` | `PILOT` | P2 | Ready |
| GP-ORD-009 | ✓ | ✓ | ✓ | ✓ | `BASE` | — (pass) | P1 | Ready |
| GP-ORD-010 | Для GP-ORD-001: тіло `422` | — | — | — | — | Рівно один `failing_gate` (не масив, не всі гейти) | P1 | Ready |

Для кожного рядка перевірити: (а) `failing_gate`; (б) `rule_reference` відповідає умові саме цього гейта за `06` §6; (в) `message` зрозумілий оператору і стосується саме цієї умови (`03` «The 422 body»).

### 6.6 Контракт відповіді `422` (`03` «The 422 body»; `06` §6)

| ID | Перевірка | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-422-001 | Для кожного з 12 рядків `06` §6 (гейтових) отримати `422` через `submit` | `rule_reference` дослівно: `OH-1 §2.1`, `OH-1 §2.3`, `ZC-2026.1 §2` (AIRCRAFT), `ZC-2026.1 §2` (ZONE), `OH-1 §4`, `ZC-2026.1 §4`*, `TSD-4 §2.1`, `TSD-4 §2.2`, `TSD-4 §3`, `A-7 §1`, `ZC-2026.1 §3`, `TSD-4 §5`* | P1 | Ready (* — див. GP-ZON-040, OQ-11) |
| GP-422-002 | Жодна відповідь не містить ілюстративного `TSD-4 §3.2` | відсутній | P1 | Ready |
| GP-422-003 | `failing_gate` ∈ {`PILOT`, `AIRCRAFT`, `ZONE`, `TIME_WINDOW`} у верхньому регістрі | так | P1 | Ready |
| GP-422-004 | Поля `failing_gate`, `rule_reference`, `message` присутні і непорожні | так | P1 | Ready |
| GP-422-005 | Після `422` від `submit`: `GET /flight-requests/{id}` повертає ті самі `failing_gate` і `rule_reference`, заповнений `rejection_reason` | так (`01`, міграція `0002`) | P1 | Ready |
| GP-422-006 | Регрес-пастки `CR-1`: значення `rule_reference` з `CR-1` або категорії `C4`/`C5` ніде не з'являються | так | P3 | Ready |

### 6.7 Ендпоінт `validate` (`POST /api/v1/flight-requests/{id}/validate`; `03` endpoint 13)

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-VAL-001 | `validate` для «зеленої» `DRAFT`-заявки | усі гейти pass; стан `DRAFT` | P1 | Needs clarification (OQ-2: код і форма тіла) |
| GP-VAL-002 | `validate` для заявки, що падає на гейті (будь-який кейс §6.1–6.4) | вердикт fail з тим самим `failing_gate`/`rule_reference`, що дав би `submit`; стан `DRAFT` | P1 | Needs clarification (OQ-2) |
| GP-VAL-003 | Відсутність побічних ефектів: порівняти `GET /flight-requests/{id}` до і після | `state`, `submitted_at`, `decided_at`, `failing_gate`, `rule_reference`, `rejection_reason` не змінились; у `AuditLog` немає нових записів для заявки | P1 | Ready |
| GP-VAL-004 | Ідемпотентність: 5 викликів поспіль для одних даних | однаковий результат щоразу, без змін стану | P1 | Ready |
| GP-VAL-005 | `validate` не «резервує» вікно: після `validate R1` створити й подати `R2` того ж дрона з тим самим вікном | `R2` не отримує перетину від `R1` (`DRAFT` невидимий для перевірки перетину) | P2 | Ready |
| GP-VAL-006 | `validate` з заголовком `X-Reviewer-Id`, без сесії оператора | дозволено; результат як у власника (`03` «Reviewer-or-owner reads») | P1 | Ready |
| GP-VAL-007 | `validate` без сесії і без `X-Reviewer-Id` | `401` | P1 | Ready |
| GP-VAL-008 | `validate` сесією `OP-OTHER` для заявки `OP-L3` | `403` | P1 | Ready |
| GP-VAL-009 | `validate` для неіснуючого `id` | `404` | P2 | Ready |
| GP-VAL-010 | `validate` для заявок у `UNDER_REVIEW`, `REJECTED`, `APPROVED`, `CANCELLED` | не визначено (дозволено / `409`); рев'юерська детальна сторінка викликає `validate` для `UNDER_REVIEW` (`03`, `04`), отже для `UNDER_REVIEW` очікується дозволено | P2 | Needs clarification (OQ-13) |
| GP-VAL-011 | Чи `validate` при відмові повертає результати всіх чотирьох гейтів, чи лише перший провал | не визначено: `03` «stopping at the first failure» vs `04` «per-gate pass/fail», «all four gate results» | P1 | Needs clarification (OQ-2) |
| GP-VAL-012 | `validate` від `OP-SUSP` | `validate` не є записом; поведінка для `SUSPENDED` описана лише для create/submit/approve | P3 | Needs clarification (OQ `05` рядок 4) |
| GP-VAL-013 | Диференційний прогін: усі Ready-кейси §6.1–6.5 через `validate` і `submit` | вердикти та `failing_gate`/`rule_reference` збігаються (виняток — GP-TW-084) | P1 | Ready |

### 6.8 Ендпоінт `submit` (`POST /api/v1/flight-requests/{id}/submit`; `02`; `03` endpoint 10)

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-SUB-001 | `submit` «зеленої» `DRAFT` | стан `UNDER_REVIEW`; `submitted_at` заповнено; `failing_gate`/`rule_reference`/`rejection_reason` порожні | P1 | Ready (код успіху `200`/`201` не зафіксовано — OQ-2) |
| GP-SUB-002 | `submit` `DRAFT`, що падає на гейті | `422` з контрактним тілом; стан `REJECTED`; поля відмови збережено (GP-422-005) | P1 | Ready |
| GP-SUB-003 | Стан `SUBMITTED` ніколи не спостерігається: `GET` одразу після `submit` і в списку `GET /flight-requests?state=SUBMITTED` | `SUBMITTED` відсутній | P1 | Ready |
| GP-SUB-004 | `AuditLog` після GP-SUB-001/002 | є запис(и) переходу з `DRAFT`; чи логується проміжний `SUBMITTED` окремим рядком — не визначено | P2 | Needs clarification (OQ-14) |
| GP-SUB-005 | Повторний `submit` заявки в `UNDER_REVIEW` | `409`, стан не змінився (`02`: явний приклад) | P1 | Ready |
| GP-SUB-006 | `submit` з `REJECTED` (після відмови гейта) | `409` | P1 | Ready |
| GP-SUB-007 | `submit` з `APPROVED` | `409` | P2 | Ready |
| GP-SUB-008 | `submit` з `CANCELLED` | `409` | P2 | Ready |
| GP-SUB-009 | `submit` з `COMPLETED` | `409` | P3 | Blocked (OQ-12) |
| GP-SUB-010 | `409` має пріоритет над гейтами: заявка в `UNDER_REVIEW`, дані якої тепер порушують гейт (наприклад, пілот прострочився) → `submit` | `409`, а не `422` (`03`: «`409` — cannot be attempted at all») | P2 | Ready |
| GP-SUB-011 | `submit` без сесії і без `X-Reviewer-Id` | `401` | P1 | Ready |
| GP-SUB-012 | `submit` сесією `OP-OTHER` | `403` | P1 | Ready |
| GP-SUB-013 | `submit` лише з `X-Reviewer-Id` (без сесії) | відмова (ендпоінт «owning operator only»); код `401` чи `403` не визначено | P2 | Needs clarification (OQ-15) |
| GP-SUB-014 | `submit` неіснуючого `id` | `404` | P2 | Ready |
| GP-SUB-015 | `submit` від `OP-SUSP` («зелена» заявка) | не `422` і без `failing_gate` (`05` §1); точний код і тіло не визначені | P1 | Blocked (OQ `05` рядок 4) |
| GP-SUB-016 | Два паралельні `submit` двох заявок одного дрона з перетином вікон | щонайбільше одна в `UNDER_REVIEW` — очікування не задокументоване (конкурентність) | P3 | Needs clarification (OQ-16) |
| GP-SUB-017 | Після `submit` → `REJECTED` змінити дані (наприклад, пілота) і подати знову ту саму заявку | `409` — `REJECTED` термінальний; потрібна нова заявка | P2 | Ready |

### 6.9 UI оператора (`04` Operator portal)

| ID | Сценарій | Очікувано | Пріоритет | Статус |
|---|---|---|---|---|
| GP-UI-001 | На формі створення заявки обрати несумісну комбінацію (`P-BASIC` + `LVV-RST-2` + `AC-30000`) | пікери (zone, pilot, aircraft) **не** фільтрують варіанти; комбінацію можна обрати і зберегти | P1 | Ready |
| GP-UI-002 | Натиснути «Check before submitting» для заявки з GP-UI-001 | показано результат по гейтах (pass/fail); стан заявки в «My requests» лишається `DRAFT` | P1 | Needs clarification (OQ-2, OQ-17) |
| GP-UI-003 | Натиснути «Check before submitting» кілька разів | результат стабільний, стан не змінюється | P2 | Ready |
| GP-UI-004 | Подати заявку з гейтовою відмовою, відкрити детальну сторінку (Request detail view) | відображено `rejection_reason` і `rule_reference` | P1 | Ready |
| GP-UI-005 | «My requests», фільтр за станом `REJECTED` / `UNDER_REVIEW` після кейсів §6.8 | заявки у відповідних фільтрах | P3 | Ready |
| GP-UI-006 | Рев'юер: детальна сторінка заявки в черзі | показано «all four gate results» | P2 | Needs clarification (OQ-2) |
| GP-UI-007 | Інвентаризація `data-testid`: кнопка submit (`flight-request-submit-button` — приклад з `04`), кнопка «Check before submitting», блок результатів гейтів | кожен інтерактивний елемент має `data-testid`; точні значення для validate-кнопки і блоку результатів у `04` не наведені — зафіксувати фактичні як базову лінію | P2 | Needs clarification (OQ-17) |

---

## 7. Зведена матриця граничних значень

| Параметр | Правило | Нижче межі | На межі | Вище межі | Кейси |
|---|---|---|---|---|---|
| Актуальність пілота | 18 міс від `certified_on` на `WS`, без grace | `E` − 1 день: pass | `E`: **?** (OQ-3) | `E` + 1 день: fail | GP-PIL-020…022 |
| `C0`/`C1` | ≤ 900 г — `C0` | 900: `C0` | — | 901: `C1` | GP-AIR-002/003 |
| `C1`/`C2` | ≤ 3 500 г — `C1` | 3 500: `C1` | — | 3 501: `C2` | GP-AIR-005/006 |
| `C2`/`C3` | ≤ 12 000 г — `C2` | 12 000: `C2` | — | 12 001: `C3` | GP-AIR-008/009 |
| Реєстрація | ≤ 30 000 г | 30 000: OK | — | 30 001: `422` | GP-AIR-030/031 |
| Висота | ≤ стелі (45/60/90/120) | стеля − 1: pass | стеля: pass | стеля + 1: fail | GP-ZON-001…008 |
| Рівень авторизації | оператор ≥ зона | на рівень нижче: fail | рівний: pass | вищий: pass | GP-ZON-020…028 |
| Lead time | ≥ 6 год | 5:59: fail | 6:00: pass | 6:01: pass | GP-TW-001…003 |
| Макс. наперед | ≤ 60 діб | 59д 23:59: pass | 60д: pass (OQ-8) | 60д 00:01: fail | GP-TW-010…012 |
| Тривалість (мін.) | ≥ 15 хв | 14: fail | 15: pass | 16: pass | GP-TW-020…022 |
| Тривалість (макс.) | ≤ 240 хв | 239: pass | 240: pass | 241: fail | GP-TW-023…025 |
| Curfew початок `KBP`/`ODS` 23:00 | жодна хвилина в curfew | кінець 22:59: pass | кінець 23:00: **?** (OQ-10) | кінець 23:01: fail | GP-TW-031…033 |
| Curfew кінець `KBP`/`ODS` 05:00 | — | старт 04:59: fail | старт 05:00: **?** (OQ-10) | старт 05:01: pass | GP-TW-036…038 |
| Curfew початок `LVV`/`ZTM` 22:00 | — | кінець 21:59: pass | кінець 22:00: **?** | кінець 22:01: fail | GP-TW-050…052 |
| Curfew кінець `LVV`/`ZTM` 06:00 | — | старт 05:59: fail | старт 06:00: **?** | старт 06:01: pass | GP-TW-055…057 |
| Перетин | спільна мить з `APPROVED`/`UNDER_REVIEW` | перетин 1 хв: fail | дотик: pass | проміжок: pass | GP-TW-070…078 |

---

## 8. Відкриті питання та прогалини, виявлені під час планування

Питання з `05-open-questions.md` (канонічний статус):

| # | Питання | Статус у `05` | Вплив |
|---|---|---|---|
| `05`-4 | `Operator.status = SUSPENDED` — код і тіло відповіді | Open | GP-SUB-015 `Blocked`, GP-VAL-012 |
| `05`-5 | Цитата для `active` зони (`ZC-2026.1 §4`) | Open | GP-ZON-040/041: правило перевіряється, цитата — спостереження |

Нові питання, виявлені цим планом (у `05` відсутні; потрібно передати власнику документації):

| # | Питання / суперечність | Де | Кейси |
|---|---|---|---|
| OQ-1 | Не задокументовано часовий пояс зон (IANA-ідентифікатор) і правила переходу на літній/зимовий час. `06` каже лише «zone-local time». | `06` §1.4; `01` Time handling | GP-TW-030…064 |
| OQ-2 | Форма відповіді `validate` і `submit` не визначена: `03` — «stopping at the first failure», `04` — «per-gate pass/fail result» / «all four gate results», `08` — `validate` повертає «200 with per-gate results», але на тій самій діаграмі пайплайн повертає `422`. Невідомо: чи `validate` повертає `422` при відмові чи `200` з результатами; чи показує результати гейтів після першого провалу; код успіху `submit` (`200`/`201`). | `03` endpoint 13; `04`; `08` Gate Pipeline | GP-VAL-001/002/011, GP-SUB-001, GP-UI-002/006 |
| OQ-3 | Точна межа 18 місяців: чи сам день `certified_on` + 18 міс ще актуальний; арифметика для 29–31 числа; в якому часовому поясі `WS` (UTC timestamp) порівнюється з `certified_on` (date); `certified_on` у майбутньому. | `06` §2 | GP-PIL-021, 027, 028 |
| OQ-4 | Порядок умов **усередині** гейта (`PILOT`: рейтинг/актуальність; `ZONE`: висота/рівень/активність; `TIME_WINDOW`: 5 умов) не визначений → невідомо, який `rule_reference` повертається при кількох порушеннях одного гейта. | `06` §1; `03` | GP-PIL-040, GP-ZON-041/050, GP-TW-090/091 |
| OQ-5 | `422` при реєстрації дрона > 30 000 г: контракт `422` вимагає `failing_gate` з чотирьох значень, але це не гейт (`06` §1.2). Форма тіла не визначена. | `03` «The 422 body»; `06` §1.2 | GP-AIR-031 |
| OQ-6 | Тип і діапазон числових полів: `requested_altitude_metres` (ціле/дробове, ≥ 0?), `mtom_grams` (0, дробове); точність часу (секунди) для lead time і тривалості. | `01` | GP-AIR-032, GP-ZON-009/010, GP-TW-002/026 |
| OQ-7 | Від якого моменту `validate` рахує lead time і 60 днів — «submission» для `validate` не визначено. | `06` §1.4; `03` endpoint 13 | GP-TW-007 |
| OQ-8 | «≤ 60 days» — календарні дні чи 60 × 24 год; як рахувати через перехід DST. | `06` §1.4 | GP-TW-011/012 |
| OQ-9 | `WE` ≤ `WS`: відхиляється при створенні (`01`: «must be after») з `400`, чи доходить до `TIME_WINDOW` (`06` §1.4, `TSD-4 §3`) як `422`. | `01`; `06` §1.4 | GP-TW-027/028 |
| OQ-10 | Включність меж curfew: чи вікно, що закінчується рівно на `curfew_start` або починається рівно на `curfew_end`, «falls inside» curfew. Для перетину заявок дотик явно дозволений, для curfew — ні. | `06` §1.4 | GP-TW-031/037/051/056 |
| OQ-11 | Перетин розкладу: `06` §1.4 відносить його до `TIME_WINDOW` (`422`, `TSD-4 §5`), а `03` «Error semantics» називає «scheduling conflict between requests» підставою для `409`; `02` пише «overlap check inside the `AIRCRAFT` / `TIME_WINDOW` gates». Потрібно підтвердити: код (`422` vs `409`) і гейт (`TIME_WINDOW` vs `AIRCRAFT`). | `02`; `03`; `06` §1.4 | GP-TW-072…077, 084 |
| OQ-12 | Ендпоінт maintenance-completion (єдиний шлях до `COMPLETED`, `02`) відсутній у переліку ендпоінтів `03` → неможливо підготувати заявку в `COMPLETED`. | `02`; `03` | GP-TW-082, GP-SUB-009 |
| OQ-13 | Чи дозволено `validate` для заявок поза `DRAFT` і чи виключає перевірка перетину саму заявку (для `UNDER_REVIEW`, яку переглядає рев'юер). | `03` endpoint 13; `04` Reviewer queue | GP-VAL-010, GP-TW-085 |
| OQ-14 | Чи логується транзитний стан `SUBMITTED` в `AuditLog` окремим записом (`DRAFT→SUBMITTED`, `SUBMITTED→UNDER_REVIEW`) чи одним (`DRAFT→UNDER_REVIEW`). | `01` AuditLog; `02` | GP-SUB-004 |
| OQ-15 | `submit` лише з `X-Reviewer-Id`: `401` чи `403`. | `03` Authentication | GP-SUB-013 |
| OQ-16 | Поведінка при одночасних `submit` заявок одного дрона з перетином. | `02`; `06` §1.4 | GP-SUB-016 |
| OQ-17 | `data-testid` для кнопки «Check before submitting» і блоку результатів гейтів у `04` не наведені. | `04` | GP-UI-002/007 |
| OQ-18 | Посилання на пілота/дрона іншого оператора при створенні заявки — поведінка не описана (поза обсягом цього плану, зафіксовано для повноти). | `03` | — |

---

## 9. Ризики

| Ризик | Ймовірність | Вплив | Пом'якшення |
|---|---|---|---|
| Реалізовано дослівний текст `ZC-2026.1` §2 (вага → `ZONE`) | Середня | Високий | GP-AIR-020, GP-AIR-021 |
| Використано застарілі `OH-1` v1.4 (24 міс, 4 000 г) або базовий curfew замість `A-7` | Середня | Високий | GP-PIL-026, GP-AIR-023, GP-TW-039/040/053 |
| Логіку гейтів продубльовано між `validate` і `submit` — розбіжність вердиктів | Середня | Високий | GP-VAL-013 |
| Curfew порівнюється в UTC, а не в локальному часі | Середня | Високий | GP-TW-062/063 (після OQ-1) |
| Перевірка curfew лише по кінцях вікна, а не похвилинно | Низька | Середній | GP-TW-065 — з чинними даними не відтворюється, передати на unit-рівень |
| Нестабільність граничних кейсів за часом (lead time, 60 днів) | Висока | Середній | ENV-5, кейси ±1 хв, фіксація фактичного `T` з відповіді/логів |
| Неможливість підготувати дані (`L2`/`L3`, неактивна зона, `COMPLETED`) | Середня | Середній | ENV-2/3, OQ-12 |

## 10. Критерії входу і виходу

**Вхід:**
- Розгорнуто збірку з усіма чотирма гейтами і ендпоінтами 9–14; дані §4 підготовлено (ENV-1…ENV-6).
- Відповіді на OQ-1 і OQ-2 бажані до старту (без них частина кейсів виконується як спостереження).

**Вихід:**
- 100 % кейсів P1 зі статусом `Ready` виконано; дефектів рівня Critical/High немає відкритих.
- ≥ 90 % кейсів P2 `Ready` виконано.
- Для кожного `Needs clarification` зафіксовано фактичну поведінку, а питання передано власнику docs; кейси `Blocked` задокументовано з посиланням на питання.
- Диференційний прогін GP-VAL-013 без розбіжностей.

## 11. Трасованість (вимога → кейси)

| Вимога | Джерело | Кейси |
|---|---|---|
| Рейтинг vs клас зони | `06` §1.1, `OH-1 §2.1` | GP-PIL-001…010, 042 |
| Актуальність 18 міс на `WS` | `06` §1.1, §2, `OH-1 §2.3` | GP-PIL-020…029 |
| Статуси пілота/дрона не гейтові | `05` рядок 2; `07` п.2 | GP-PIL-041, GP-AIR-022 |
| Вага vs `permitted_bands` → `AIRCRAFT` | `06` §1.2, §3; `07` п.1 | GP-AIR-001…024 |
| Реєстрація ≤ 30 000 г | `06` §1.2, §3, `OH-1 §3` | GP-AIR-030…032 |
| Висота ≤ стелі | `06` §1.3, §4 | GP-ZON-001…010 |
| Рівень авторизації → `ZONE`, `OH-1 §4` | `06` §1.3; `07` п.4; `05` рядок 3 | GP-ZON-020…029 |
| Неактивна зона | `06` §1.3; `05` рядок 5 | GP-ZON-040/041 |
| Lead time ≥ 6 год | `06` §1.4, `TSD-4 §2.1` | GP-TW-001…007 |
| ≤ 60 днів | `06` §1.4, `TSD-4 §2.2` | GP-TW-010…014 |
| Тривалість 15–240 | `06` §1.4, `TSD-4 §3`; `01` | GP-TW-020…028 |
| Curfew, `A-7` | `06` §1.4, §4 | GP-TW-030…065 |
| Перетин | `06` §1.4, `TSD-4 §5`; `02` | GP-TW-070…085 |
| Порядок гейтів | `06`; `03` | GP-ORD-001…010 |
| Контракт `422` | `03`; `06` §6 | GP-422-001…006 |
| `validate` без побічних ефектів, доступ | `03` endpoint 13 | GP-VAL-001…013 |
| `submit`, переходи, `409` | `02`; `03` endpoint 10 | GP-SUB-001…017 |
| `SUSPENDED` — не гейт | `05` §1; `07` п.3 | GP-SUB-015, GP-VAL-012 |
| UI: без префільтрації, «Check before submitting», детальна сторінка | `04` | GP-UI-001…007 |
