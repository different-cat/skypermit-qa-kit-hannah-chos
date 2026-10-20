# TP-gate-pipeline: Гейти PILOT / AIRCRAFT / ZONE / TIME_WINDOW та ендпоінти validate / submit

| Поле | Значення |
|---|---|
| ID | `TP-gate-pipeline` |
| Тип | Фічевий |
| Версія | 0.1 |
| Дата | 2026-10-20 |
| Автор | згенеровано Claude |
| Джерела | `../skypermit/docs/` @ `78407f1`: `01-data-model.md`, `02-state-machine.md`, `03-api-contract.md`, `04-frontend-spec.md`, `05-open-questions.md`, `06-business-rules.md`, `07-clarifications.md`; `08-uml-diagrams.md` — лише як допоміжний |

## 1. Мета

Підтвердити, що gate pipeline (`PILOT → AIRCRAFT → ZONE → TIME_WINDOW`, перша відмова) приймає й відхиляє заявки рівно за правилами `06`/`07`. Окремо — що `submit` правильно переводить стан, а `validate` дає той самий вердикт і не має побічних ефектів. Граничні значення кожного порогу розписані з точністю до 1 г / 1 м / 1 хв / 1 день. План використовується для приймання фічі «gate pipeline» перед релізом.

## 2. Обсяг

### 2.1 В обсязі

| AREA | Що саме |
|---|---|
| `GP` | рейтинг × `zone_class`, currency 18 міс. відносно `window_start` |
| `GA` | weight band ∈ `permitted_bands`, межі бендів ±1 г, регрес `OH-1` v1.4 |
| `GZ` | висота ≤ `ceiling_metres`, `authorisation_level` ≥ required, зона `active` |
| `GT` | lead time, advance booking, тривалість, curfew (включно з `A-7`), overlap |
| `PIPE` | порядок гейтів, first failure, тіло `422`, дослівний `rule_reference`, що зберігається в заявці після відмови |
| `VAL` | `POST /flight-requests/{id}/validate`: той самий вердикт, без зміни стану, ідемпотентність, доступ owner/reviewer, кнопка «Check before submitting» |
| `SM` (частково) | лише перехід `submit`: `DRAFT → UNDER_REVIEW/REJECTED`, `409` на submit з усіх інших станів |
| `AUTH` (частково) | доступ до `submit` (`401`/`403`/`404`) |
| `SUSP` (частково) | `SUSPENDED`-оператор на `submit` |

Ендпоінти: `POST /flight-requests/{id}/submit` (10), `POST /flight-requests/{id}/validate` (13), допоміжні `GET /flight-requests/{id}` (11), `GET /pilots/{id}` (4), `GET /aircraft/{id}` (6), `GET /zones` (7). Як підготовчі кроки: `POST /flight-requests` (9), `approve`/`reject` (15/16), `cancel` (14), maintenance completion.

### 2.2 Поза обсягом

- Реєстрація дрона > 30 000 г (`422` на `/aircraft`, `OH-1 §3`). Це не гейт: вона належить до `REG` і тут потрібна лише як передумова (дрон на 30 001 г неможливо використати в заявці).
- Валідація тіла `POST /flight-requests` (`400`, `flight_purpose` ≤ 2000) — `FR`. Виняток: межа `window_end ≤ window_start`, бо вона перетинається з `TSD-4 §3` (див. `TS-GT-10`).
- Решта переходів state machine (approve/reject/cancel/complete) і дедлайн скасування — `SM`, `CAN`, `REV`, `CMP`.
- Черга рев'юера як UI (`UIR`). Повна перевірка контракту `data-testid` (`TID`).
- Автотести, навантаження, безпека поза `401`/`403`.
- Скасування з `SUBMITTED`: стан транзитний, мануально недосяжний.

## 3. Об'єкт тестування

- Сутності: `FlightRequest` (`state`, `failing_gate`, `rule_reference`, `rejection_reason`, `submitted_at`), `Pilot` (`rating`, `certified_on`, `status`), `Aircraft` (`mtom_grams`, `status`), `Operator` (`authorisation_level`, `status`), `Zone` (усі поля), `AuditLog`, `ReviewAction` — `01`.
- Стани й переходи: `DRAFT --submit--> SUBMITTED --> UNDER_REVIEW | REJECTED` — `02`, таблиця переходів.
- Логіка гейтів і значення `rule_reference` — `06` §1–§6 з виправленням `07` п.1.
- UI: форма заявки, кнопка «Check before submitting», «My requests», деталі заявки — `04` Operator portal.

## 4. Підхід

- **Рівні.** Основна маса — мануальне функціональне тестування API через REST-клієнт (Postman/Bruno/curl) із сесійною cookie оператора або заголовком `X-Reviewer-Id`. Поверх цього — короткий UI-прохід порталу оператора для `validate` і перегляду відмови.
- **Шаблон перевірки гейту.** Для кожного сценарію одна змінна — та, що перевіряється. Усе інше налаштоване як «золота» валідна комбінація (див. §6.5), щоб відмова гарантовано йшла від цієї змінної, а не від іншого гейту. Кожен граничний набір проганяється двічі: спершу `validate`, потім `submit` на копії заявки. Обидва виклики мають дати однаковий вердикт.
- **Техніки:**
  - *Boundary value analysis* — маса (межа і +1 г для кожного бенду в зоні, де ця межа розділяє pass/fail), висота (стеля і +1 м), currency (мітка 18 міс. −1 день / +1 день), lead time (6 год ±1 хв), advance (60 днів поза зоною двозначності), тривалість (14/15/16 і 239/240/241 хв), curfew (±1 хв від меж локального часу), overlap (дотик і перетин на 1 хв).
  - *Decision table* — рейтинг × `zone_class` (3×3), `authorisation_level` × required level (3×3), weight band × `permitted_bands` (4 бенди × 6 зон).
  - *Equivalence partitioning* — стани іншої заявки для overlap (враховуються / не враховуються), класи зон, статуси пілота й дрона.
  - *Pairwise / комбінації відмов* — усі 6 пар гейтів і всі 4 гейти разом, перевірка first failure.
  - *State transition* — `submit` з кожного з 6 спостережуваних станів.
  - *Negative* — `401`/`403`/`404`/`409`.
- **Побічні ефекти.** Після кожного `submit` і `validate` виконується `GET /flight-requests/{id}`: перевіряються `state`, `failing_gate`, `rule_reference` і `rejection_reason`. `ReviewAction` і `AuditLog` у API не читаються (див. `F-13`), тому для них потрібен read-only доступ до БД стенду. Якщо його немає, ці перевірки позначаються «не виконано — немає доступу».
- **Curfew і повне покриття.** Найкоротший curfew у каталозі триває 6 год (`23:00–05:00`), а найдовше вікно — 240 хв. Тому вікно, яке накриває curfew цілком і має обидва кінці поза ним, побудувати неможливо. Вимога «minute-by-minute» перевіряється вікнами, що зачіпають curfew на 1 хв, вікнами через локальну північ і вікнами повністю всередині curfew.

## 5. Тестові сценарії та матриця покриття

Пріоритети: **P1** — хибний дозвіл/відмова на політ, порушення доступу; **P2** — коди, тіла відповідей, побічні ефекти; **P3** — косметика, допоміжні похідні поля.
`422`-відповідь у колонці «Сценарій» скорочено як `422 GATE / rule_reference`; тіло завжди містить `failing_gate`, `rule_reference`, `message` (`03` «The 422 body»).

### 5.1 `GP` — гейт PILOT

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-GP-01` | Рейтинг покриває клас зони — 6 комбінацій: `R_BASIC`/`OPEN`, `R_ADVANCED`/`OPEN`, `R_ADVANCED`/`CONTROLLED`, `R_BVLOS`/`OPEN`, `R_BVLOS`/`CONTROLLED`, `R_BVLOS`/`RESTRICTED` → `PILOT` не падає (за золотої комбінації решти → `UNDER_REVIEW`) | Decision table | P1 | `06` §1.1, §4 (останній абзац) | Ready |
| `TS-GP-02` | Рейтинг не покриває клас — 3 комбінації: `R_BASIC`/`CONTROLLED`, `R_BASIC`/`RESTRICTED`, `R_ADVANCED`/`RESTRICTED` → `422 PILOT / OH-1 §2.1` | Decision table, Negative | P1 | `06` §1.1, §6 | Ready |
| `TS-GP-03` | Currency: `window_start` припадає на день **перед** 18-місячною міткою від `certified_on` → pass | BVA | P1 | `06` §1.1, §2 | Ready |
| `TS-GP-04` | Currency: `window_start` припадає на день **після** мітки (+1 день) → `422 PILOT / OH-1 §2.3`. Grace period немає | BVA | P1 | `06` §2 («one day past … is not current») | Ready |
| `TS-GP-05` | Currency: `window_start` припадає **рівно на день мітки** (дата `certified_on` + 18 міс.) | BVA | P1 | `06` §2 | Needs clarification (`F-4`) |
| `TS-GP-06` | Currency для `certified_on`, після якого «+18 міс.» дає неіснуючу дату (31-ше число; 29 лютого) | BVA | P2 | `06` §2 | Needs clarification (`F-4`) |
| `TS-GP-07` | Пілот current на момент submit, але мітка 18 міс. настає до `window_start` → `422 PILOT / OH-1 §2.3` (перевіряється проти `window_start`, не часу подачі) | BVA, EP | P1 | `06` §1.1, §2 | Ready |
| `TS-GP-08` | Регрес `OH-1` v1.4: від `certified_on` до `window_start` минуло 20 міс. (між 18 і 24) → `422 PILOT / OH-1 §2.3`, а не pass за старими 24 міс. | Regression, BVA | P1 | `06` §2 (⚠️ v1.4) | Ready |
| `TS-GP-09` | Пілот зі `status = INACTIVE`, рейтинг і currency валідні → `PILOT` не падає | EP | P2 | `05` рядок 2; `07` п.2 | Ready |
| `TS-GP-10` | Одночасно не підходить рейтинг і минула currency — яку `rule_reference` повертає `422` | Pairwise | P2 | `06` §1.1, §6 | Needs clarification (`F-9`) |
| `TS-GP-11` | `GET /pilots/{id}` показує currency «зараз» (current), а `submit` із `window_start` після мітки падає на `PILOT`. Це очікувана розбіжність, не дефект | EP | P3 | `01` «Derived, never stored»; `06` §2 | Ready |

### 5.2 `GA` — гейт AIRCRAFT

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-GA-01` | Межа `C0`/`C1` у `ZTM-RST-7` (лише `C0`): 900 г → pass; 901 г → `422 AIRCRAFT / ZC-2026.1 §2` | BVA | P1 | `06` §1.2, §3, §4 | Ready |
| `TS-GA-02` | Межа `C1`/`C2` у `LVV-RST-2` (`C0`–`C1`): 3 500 г → pass; 3 501 г → `422 AIRCRAFT / ZC-2026.1 §2` | BVA | P1 | `06` §1.2, §3, §4 | Ready |
| `TS-GA-03` | Межа `C2`/`C3` у `KBP-CTR-3` і `ODS-CTR-5` (`C0`–`C2`): 12 000 г → pass; 12 001 г → `422 AIRCRAFT / ZC-2026.1 §2` | BVA | P1 | `06` §1.2, §3, §4 | Ready |
| `TS-GA-04` | Верх `C3` у `DNI-OPN-1` і `KHR-OPN-4` (`C0`–`C3`): 30 000 г і 12 001 г → pass | BVA | P1 | `06` §3, §4 | Ready |
| `TS-GA-05` | Weight band не дозволений → `failing_gate` = **`AIRCRAFT`**, а **не** `ZONE`, попри дослівне «fails the zone gate» у `ZC-2026.1 §2` | Negative | P1 | `07` п.1; `06` §1.2 ⚠️ | Ready |
| `TS-GA-06` | Регрес `OH-1` v1.4: 3 800 г тепер `C2`, а не `C1` → у `LVV-RST-2` `422 AIRCRAFT / ZC-2026.1 §2`; у `KBP-CTR-3` pass | Regression, BVA | P1 | `06` §3 (⚠️ v1.4) | Ready |
| `TS-GA-07` | Повна таблиця бенд × зона на середніх значеннях (450 / 2 000 / 8 000 / 20 000 г × 6 зон): pass тоді й лише тоді, коли бенд ∈ `permitted_bands` | Decision table, EP | P2 | `06` §3, §4 | Ready |
| `TS-GA-08` | Дрон зі `status = RETIRED`, валідний бенд → `AIRCRAFT` не падає | EP | P2 | `05` рядок 2; `07` п.2 | Ready |
| `TS-GA-09` | `GET /aircraft/{id}` для 900 / 901 / 3 500 / 3 501 / 12 000 / 12 001 / 30 000 г показує `C0`/`C1`/`C1`/`C2`/`C2`/`C3`/`C3`, і це збігається з рішенням гейту | BVA | P3 | `01` «Derived»; `06` §3 | Ready |

### 5.3 `GZ` — гейт ZONE

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-GZ-01` | Висота = стеля → pass: `KBP-CTR-3` 120, `ODS-CTR-5` 90, `LVV-RST-2` 60, `ZTM-RST-7` 45, `DNI-OPN-1` 120, `KHR-OPN-4` 120 м | BVA | P1 | `06` §1.3, §4 | Ready |
| `TS-GZ-02` | Висота = стеля + 1 м (121 / 91 / 61 / 46 / 121 / 121) → `422 ZONE / ZC-2026.1 §2` | BVA, Negative | P1 | `06` §1.3, §6 | Ready |
| `TS-GZ-03` | Дробова висота (напр. 120.5 при стелі 120), 0 м, від'ємна висота — прийом і результат | BVA | P2 | `01` FlightRequest (тип не вказано) | Needs clarification (`F-14`) |
| `TS-GZ-04` | Рівень оператора нижчий за потрібний — `L1`→`L2` (`KBP-CTR-3`, `ODS-CTR-5`), `L1`→`L3` і `L2`→`L3` (`LVV-RST-2`, `ZTM-RST-7`) → `422 ZONE / OH-1 §4` | Decision table, Negative | P1 | `06` §1.3, §6; `07` п.4; `05` рядок 3 | Ready |
| `TS-GZ-05` | Рівень ≥ потрібного — `L1`/`L2`/`L3` у `DNI-OPN-1`; `L2`, `L3` у `KBP-CTR-3`; `L3` у `LVV-RST-2` → pass (≥, вищий рівень теж підходить) | Decision table | P1 | `06` §1.3 | Ready |
| `TS-GZ-06` | Зона з `active = false` (решта валідна) → заявка відхиляється з `failing_gate: ZONE` | EP, Negative | P1 | `06` §1.3; `05` §2 (правило settled) | Ready |
| `TS-GZ-07` | Неактивна зона: дослівне значення `rule_reference` у `422` | Negative | P2 | `05` §2, рядок 5 | Blocked (OQ-5) — не визначено, див. `05` §2 |
| `TS-GZ-08` | В одній зоні одночасно перевищено висоту і замалий рівень авторизації — яку `rule_reference` повертає `422` | Pairwise | P2 | `06` §1.3, §6 | Needs clarification (`F-9`) |
| `TS-GZ-09` | Однакова цитата `ZC-2026.1 §2` для двох різних правил: заявка А (бенд не дозволено) → `AIRCRAFT`, заявка Б (висота > стелі) → `ZONE`. `rule_reference` однаковий, правило розрізняється тільки за `failing_gate` | EP | P2 | `06` §1.2 ⚠️ «Same citation» | Ready |

### 5.4 `GT` — гейт TIME_WINDOW

Час подачі `T` — момент виклику `submit`. Curfew задано в **локальному** часі зони; UTC-значення вікна обчислюються з часового поясу зони (див. `F-3` і §6.6).

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-GT-01` | Lead time: `window_start` = `T` + 6 год 00 хв (і `T` + 6 год 01 хв) → pass (≥ 6 год) | BVA | P1 | `06` §1.4 (`TSD-4` §2.1) | Ready |
| `TS-GT-02` | Lead time: `window_start` = `T` + 5 год 59 хв → `422 TIME_WINDOW / TSD-4 §2.1` | BVA, Negative | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-03` | `window_start` у минулому → `422 TIME_WINDOW / TSD-4 §2.1` | EP, Negative | P2 | `06` §1.4 | Ready |
| `TS-GT-04` | Точка відліку lead time для `validate` і для повторної оцінки, а також точність (секунди) на межі 6 год | BVA | P2 | `06` §1.4; `03` endpoint 13 | Needs clarification (`F-5`) |
| `TS-GT-05` | Advance booking поза зоною двозначності: `window_start` = `T` + 30 днів → pass; `T` + 61 день → `422 TIME_WINDOW / TSD-4 §2.2` | EP, BVA | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-06` | Точна межа advance: `T` + 60 днів 00 хв → pass?; `T` + 60 днів + 1 хв → fail? (що таке «день» і звідки рахувати) | BVA | P2 | `06` §1.4 (`TSD-4` §2.2) | Needs clarification (`F-6`) |
| `TS-GT-07` | Тривалість 15, 16, 239, 240 хв → pass (межі включні) | BVA | P1 | `06` §1.4 (`TSD-4` §3) | Ready |
| `TS-GT-08` | Тривалість 14 і 241 хв → `422 TIME_WINDOW / TSD-4 §3` | BVA, Negative | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-09` | Тривалість 1 хв (≥ 1, < 15) → `422 TIME_WINDOW / TSD-4 §3` | EP | P3 | `06` §1.4 | Ready |
| `TS-GT-10` | `window_end` = `window_start` або `window_end` < `window_start`: де це відхиляється (`400` на create чи `422 TIME_WINDOW / TSD-4 §3` на submit) | BVA, Negative | P2 | `01` FlightRequest; `06` §1.4; `03` Error semantics | Needs clarification (`F-7`) |
| `TS-GT-11` | Регрес `A-7`, `KBP-CTR-3`: локально 22:00–22:45 і 05:15–06:00 → pass. За базовим 22:00–06:00 обидва вікна впали б | Regression, BVA | P1 | `06` §1.4 (curfew table), `A-7` §1, §4 | Ready |
| `TS-GT-12` | Регрес `A-7`, `ODS-CTR-5`: локально 22:00–22:45 і 05:15–06:00 → pass | Regression, BVA | P1 | `06` §1.4 | Ready |
| `TS-GT-13` | `KBP-CTR-3`/`ODS-CTR-5`, вікно заходить у curfew на 1 хв: локально 22:01–23:01 (кінець усередині) і 04:59–05:59 (початок усередині) → `422 TIME_WINDOW / A-7 §1` | BVA (minute-by-minute) | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-14` | `KBP-CTR-3`/`ODS-CTR-5`, curfew через північ як один період: локально 23:30–00:30, 00:30–02:30, 03:00–04:59 → `422 TIME_WINDOW / A-7 §1` | EP, BVA | P1 | `06` §1.4 («Curfews crossing midnight are one continuous period») | Ready |
| `TS-GT-15` | `LVV-RST-2`/`ZTM-RST-7` (22:00–06:00): локально 21:00–21:59 і 06:01–07:00 → pass; 21:01–22:01, 05:59–06:59, 23:00–00:30, 02:00–03:00 → `422 TIME_WINDOW / ZC-2026.1 §3` | BVA, EP | P1 | `06` §1.4, §6 | Ready |
| `TS-GT-16` | Різні `rule_reference` для curfew: однакове вікно 23:30–00:30 у `KBP-CTR-3` → `A-7 §1`, у `LVV-RST-2` → `ZC-2026.1 §3` | EP | P2 | `06` §6 | Ready |
| `TS-GT-17` | Зони без curfew (`DNI-OPN-1`, `KHR-OPN-4`): локально 23:00–02:00 і 02:00–05:00 → pass | EP | P2 | `06` §1.4, §4 | Ready |
| `TS-GT-18` | Вікно лише торкається межі curfew: кінець рівно в 23:00 (22:00–23:00) або початок рівно в 05:00 (05:00–06:00) для `KBP-CTR-3`; 21:00–22:00 / 06:00–07:00 для `LVV-RST-2` | BVA | P1 | `06` §1.4 | Needs clarification (`F-8`) |
| `TS-GT-19` | Конвертація UTC → локальний час зони: зміщення, перехід на літній/зимовий час (ніч переходу), вікно, у якого UTC-дата і локальна дата різні | BVA | P1 | `06` §1.4 «On UTC vs. zone-local time»; `01` Time handling | Needs clarification (`F-3`) |
| `TS-GT-20` | Overlap: той самий дрон, інша заявка в `UNDER_REVIEW` перетинається частково → поточна заявка **не** переходить в `UNDER_REVIEW` | EP | P1 | `06` §1.4 (`TSD-4` §5); `02` | Ready |
| `TS-GT-21` | Overlap: той самий дрон, інша заявка `APPROVED` перетинається → заявка не переходить в `UNDER_REVIEW` | EP | P1 | `06` §1.4 | Ready |
| `TS-GT-22` | Overlap на 1 хв (інша 10:00–14:00, ця 13:59–15:00) і повне вкладення / ідентичне вікно → не переходить в `UNDER_REVIEW` | BVA | P1 | `06` §1.4 | Ready |
| `TS-GT-23` | Дотик меж (інша 10:00–14:00, ця 14:00–15:00; і навпаки 09:00–10:00) → pass | BVA | P1 | `06` §1.4 («only touch at a boundary … do not overlap») | Ready |
| `TS-GT-24` | Інша заявка того ж дрона з тим самим вікном у `DRAFT`, `REJECTED`, `CANCELLED`, `COMPLETED` → pass (ці стани не враховуються) | EP | P1 | `06` §1.4; `02` overlap note | Ready |
| `TS-GT-25` | Інший дрон (той самий пілот, той самий оператор) з тим самим вікном у `UNDER_REVIEW` → pass (overlap лише для того самого дрона) | EP | P2 | `06` §1.4 («same aircraft») | Ready |
| `TS-GT-26` | Відповідь на overlap: статус-код (`422` чи `409`), `failing_gate` (`TIME_WINDOW` чи `AIRCRAFT`), `rule_reference` `TSD-4 §5`, кінцевий стан (`REJECTED` чи `DRAFT`) | Negative | P1 | `06` §1.4, §6 vs `02` overlap note vs `03` Error semantics | Needs clarification (`F-1`, `F-2`) |
| `TS-GT-27` | У межах `TIME_WINDOW` одночасно порушено кілька умов (напр. тривалість 241 хв + curfew; lead time + overlap) — яку `rule_reference` повертає `422` | Pairwise | P2 | `06` §1.4, §6 | Needs clarification (`F-9`) |

### 5.5 `PIPE` — порядок гейтів, тіло `422`, збереження результату

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-PIPE-01` | Падають `PILOT` (рейтинг) + `AIRCRAFT` (бенд) → `failing_gate: PILOT` | Pairwise | P1 | `03` Gate pipeline; `06` Evaluation order | Ready |
| `TS-PIPE-02` | Падають `PILOT` + `ZONE` (висота) → `PILOT` | Pairwise | P1 | те саме | Ready |
| `TS-PIPE-03` | Падають `PILOT` (currency) + `TIME_WINDOW` (тривалість) → `PILOT` | Pairwise | P1 | те саме | Ready |
| `TS-PIPE-04` | Падають `AIRCRAFT` + `ZONE` (рівень авторизації) → `AIRCRAFT` | Pairwise | P1 | те саме | Ready |
| `TS-PIPE-05` | Падають `AIRCRAFT` + `TIME_WINDOW` (curfew або overlap) → `AIRCRAFT` | Pairwise | P1 | те саме | Ready |
| `TS-PIPE-06` | Падають `ZONE` (висота) + `TIME_WINDOW` (lead time) → `ZONE` | Pairwise | P1 | те саме | Ready |
| `TS-PIPE-07` | Падають усі чотири гейти → лише `PILOT`; тіло містить рівно один `failing_gate` | Pairwise | P1 | `03` «The 422 body» | Ready |
| `TS-PIPE-08` | Рейтинг і рівень авторизації оцінюються незалежно (`LVV-RST-2`): `R_BVLOS` + оператор `L1` → `422 ZONE / OH-1 §4`; `R_ADVANCED` + оператор `L3` → `422 PILOT / OH-1 §2.1` | Decision table | P1 | `06` §1.1, §1.3 | Ready |
| `TS-PIPE-09` | Тіло `422`: `failing_gate` ∈ {`PILOT`,`AIRCRAFT`,`ZONE`,`TIME_WINDOW`}, `rule_reference` дослівно з `06` §6, `message` непорожній і зрозумілий оператору | Negative | P2 | `03` «The 422 body»; `06` §6 | Ready |
| `TS-PIPE-10` | Жодна відповідь не містить ілюстративну цитату `TSD-4 §3.2` (зокрема на відмову за тривалістю має бути `TSD-4 §3`) | Negative | P2 | `03` «The 422 body»; `06` §6 | Ready |
| `TS-PIPE-11` | Після відмови на `submit`: `GET /flight-requests/{id}` → `state: REJECTED`, `failing_gate` і `rule_reference` дорівнюють тілу `422`, `rejection_reason` заповнено | EP | P1 | `02` States; `01` FlightRequest | Ready |
| `TS-PIPE-12` | Який саме текст потрапляє в `rejection_reason` при відмові гейту (= `message`?) | EP | P3 | `01`; `02` | Needs clarification (`F-12`) |
| `TS-PIPE-13` | Після успішного `submit`: `failing_gate`, `rule_reference`, `rejection_reason` порожні | EP | P2 | `01` FlightRequest («populated only on …») | Ready |
| `TS-PIPE-14` | Позитивний наскрізний прохід: одна золота комбінація для кожної з 6 зон → `UNDER_REVIEW` | EP | P1 | `06` §4; `02` | Ready |
| `TS-PIPE-15` | `AuditLog` після `submit`: скільки записів (`DRAFT→SUBMITTED` + `SUBMITTED→UNDER_REVIEW/REJECTED` чи один) і що в полі `actor` | EP | P2 | `01` AuditLog; `02` | Needs clarification (`F-13`) |
| `TS-PIPE-16` | `submit` не створює `ReviewAction` (ні при pass, ні при fail) | EP | P2 | `01` ReviewAction; `03` Reviewer | Ready |

### 5.6 `VAL` — `POST /flight-requests/{id}/validate`

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-VAL-01` | `validate` на `DRAFT`, усі гейти pass → результат «усі pass»; після виклику `state` = `DRAFT`, `failing_gate`/`rule_reference`/`rejection_reason` порожні, `ReviewAction` немає | EP | P1 | `03` endpoint 13 | Ready |
| `TS-VAL-02` | `validate` на `DRAFT`, що падає на кожному з гейтів (по одному кейсу з §5.1–5.4) → названо той самий перший гейт і ту саму `rule_reference`, що повернув би `submit`; стан лишається `DRAFT`, поля відмови не записуються | EP | P1 | `03` endpoint 13 («exactly as submit would, in the same order») | Ready |
| `TS-VAL-03` | Ідемпотентність: 5 поспіль `validate` на тій самій заявці дають ідентичний результат, стан не змінюється | EP | P1 | `03` endpoint 13 («safe to call repeatedly») | Ready |
| `TS-VAL-04` | Узгодженість: для кожного граничного набору з §5.1–5.4 `validate`, а потім `submit` (без змін даних і без перетину часової межі між викликами) дають однаковий вердикт | EP | P1 | `03` endpoint 13 | Ready |
| `TS-VAL-05` | `validate` заявки А = pass → заявку Б на той самий дрон і перетинне вікно подано (`UNDER_REVIEW`) або затверджено → `submit` А не дає `UNDER_REVIEW`. Це очікувана поведінка, не дефект | State transition | P1 | `02` overlap note | Ready |
| `TS-VAL-06` | `validate` не «бронює» вікно: перевірена, але не подана заявка (`DRAFT`) не блокує `submit` іншої заявки того ж дрона з тим самим вікном | EP | P2 | `06` §1.4; `02` | Ready |
| `TS-VAL-07` | Формат відповіді `validate`: HTTP-статус при відмові (`422` чи `200` з результатами), чи повертаються результати всіх чотирьох гейтів, чи тільки до першої відмови | Negative | P1 | `03` endpoint 13 vs `04` Reviewer queue / Operator portal; `08` (похідний) | Needs clarification (`F-10`) |
| `TS-VAL-08` | `validate` рев'юером (`X-Reviewer-Id`) на `UNDER_REVIEW`: стан лишається `UNDER_REVIEW`, `ReviewAction` не створюється | EP | P1 | `03` Reviewer-or-owner reads, endpoint 13 | Ready |
| `TS-VAL-09` | Результат гейтів для `UNDER_REVIEW`: чи виключається сама заявка з overlap-перевірки, від якого моменту рахується lead time | EP | P1 | `03` endpoint 13; `06` §1.4 | Needs clarification (`F-11`) |
| `TS-VAL-10` | `validate` на `REJECTED`, `CANCELLED`, `APPROVED`, `COMPLETED` — дозволено чи `409`, і що повертається | State transition | P2 | `03` endpoint 13; `02` | Needs clarification (`F-11`) |
| `TS-VAL-11` | Доступ: власник (сесія) і будь-який `X-Reviewer-Id` → виклик виконується | Negative | P1 | `03` Reviewer-or-owner reads | Ready |
| `TS-VAL-12` | Доступ: сесія іншого оператора без reviewer-заголовка → `403` | Negative | P1 | `03` Enforcement | Ready |
| `TS-VAL-13` | Без сесії і без `X-Reviewer-Id` → `401` | Negative | P1 | `03` Error semantics | Ready |
| `TS-VAL-14` | Неіснуючий `id` → `404` | Negative | P2 | `03` Error semantics | Ready |
| `TS-VAL-15` | UI: у формі заявки «Check before submitting» показує результат pass/fail для кожного гейту. Після цього «My requests» і деталі заявки показують `DRAFT` | EP | P2 | `04` Operator portal | Ready |
| `TS-VAL-16` | UI: значення `data-testid` для кнопки «Check before submitting» і блоку результатів | — | P3 | `04` Two contracts | Needs clarification (`F-15`) |
| `TS-VAL-17` | `validate` від `SUSPENDED`-оператора: `validate` не є write-операцією зі списку `05` §1 — блокується чи ні | Negative | P2 | `05` §1; `03` endpoint 13 | Needs clarification (`F-18`) |

### 5.7 `SM` / `AUTH` / `SUSP` — ендпоінт `submit`

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-SM-01` | `submit` з `DRAFT`, гейти pass → `UNDER_REVIEW`; `SUBMITTED` ніколи не видно ні у відповіді, ні в `GET` одразу після | State transition | P1 | `02` таблиця переходів, States | Ready |
| `TS-SM-02` | `submit` з `DRAFT`, гейт падає → `422`, стан `REJECTED` (термінальний) | State transition | P1 | `02`; `03` | Ready |
| `TS-SM-03` | Повторний `submit` заявки в `UNDER_REVIEW` → `409`, стан не змінюється | State transition, Negative | P1 | `02` («re-submitting something already `UNDER_REVIEW`») | Ready |
| `TS-SM-04` | `submit` з `REJECTED`, `CANCELLED`, `APPROVED`, `COMPLETED` → `409` | State transition, Negative | P1 | `02` таблиця переходів | Ready |
| `TS-SM-05` | `409` має пріоритет над гейтами: `submit` на `REJECTED`-заявці, дані якої й зараз валять гейт, → `409`, а не `422` | Negative | P2 | `03` Error semantics («`409` means the request cannot be attempted at all») | Ready |
| `TS-SM-06` | Статус-код і тіло успішного `submit` (`200` чи `201`, що в тілі) | — | P3 | `03` (мовчить); `08` («200/201») | Needs clarification (`F-16`) |
| `TS-AUTH-01` | `submit` заявки іншого оператора → `403`, стан не змінюється | Negative | P1 | `03` Enforcement; endpoint 10 («owning operator only») | Ready |
| `TS-AUTH-02` | `submit` без сесії і без `X-Reviewer-Id` → `401` | Negative | P1 | `03` Error semantics | Ready |
| `TS-AUTH-03` | `submit` лише з `X-Reviewer-Id` (без сесії): `401` чи `403` | Negative | P2 | `03` endpoint 10 vs Error semantics | Needs clarification (`F-17`) |
| `TS-AUTH-04` | `submit` неіснуючого `id` → `404` | Negative | P2 | `03` Error semantics | Ready |
| `TS-SUSP-01` | `submit` від `SUSPENDED`-оператора **не** повертає `422` з `failing_gate` | Negative | P1 | `05` §1; `07` п.3 | Ready |
| `TS-SUSP-02` | `submit` від `SUSPENDED`-оператора: статус-код, тіло, чи змінюється стан | Negative | P1 | `05` §1, рядок 4 | Blocked (OQ-4) — не визначено, див. `05` §1 |

## 6. Тестові дані

### 6.1 Зони (з `06` §4, curfew з урахуванням `A-7`)

| Code | Class | Ceiling | Bands | Curfew (local) | Req. level | `rule_reference` curfew | Граничні значення для плану |
|---|---|---|---|---|---|---|---|
| `KBP-CTR-3` | `CONTROLLED` | 120 м | C0–C2 | 23:00–05:00 | `L2` | `A-7 §1` | 120/121 м; 12 000/12 001 г; 22:59/23:00/23:01, 04:59/05:00/05:01 |
| `ODS-CTR-5` | `CONTROLLED` | 90 м | C0–C2 | 23:00–05:00 | `L2` | `A-7 §1` | 90/91 м; 12 000/12 001 г; ті самі хвилини |
| `LVV-RST-2` | `RESTRICTED` | 60 м | C0–C1 | 22:00–06:00 | `L3` | `ZC-2026.1 §3` | 60/61 м; 3 500/3 501 г; 3 800 г (регрес); 21:59/22:00/22:01, 05:59/06:00/06:01 |
| `ZTM-RST-7` | `RESTRICTED` | 45 м | C0 | 22:00–06:00 | `L3` | `ZC-2026.1 §3` | 45/46 м; 900/901 г |
| `DNI-OPN-1` | `OPEN` | 120 м | C0–C3 | none | `L1` | — | 120/121 м; 30 000 г |
| `KHR-OPN-4` | `OPEN` | 120 м | C0–C3 | none | `L1` | — | 120/121 м; 30 000 г |
| `ZZZ-INACTIVE` *(підготовлені дані)* | будь-який | — | — | — | `L1` | — | для `TS-GZ-06/07`: копія однієї з зон з `active = false`. Через API створити неможливо — див. §7 |

### 6.2 Оператори

| Ключ | `authorisation_level` | `status` | Як отримати |
|---|---|---|---|
| `OP-L1` | `L1` | `ACTIVE` | `POST /operators` (реєстрація завжди дає `L1`) |
| `OP-L2` | `L2` | `ACTIVE` | реєстрація + out-of-band підвищення адміністратором |
| `OP-L3` | `L3` | `ACTIVE` | реєстрація + out-of-band підвищення |
| `OP-OTHER` | `L1` | `ACTIVE` | реєстрація (для `403`) |
| `OP-SUSP` | `L3` | `SUSPENDED` | out-of-band зміна статусу |

### 6.3 Пілоти (у кожного оператора-власника потрібного рівня)

`E` — дата виконання тесту, `W` — дата `window_start`. Приклад розраховано для `E = 2026-10-20`; при виконанні перерахувати.

| Ключ | `rating` | `certified_on` | Мета | Приклад (E = 2026-10-20) |
|---|---|---|---|---|
| `PL-BASIC` | `R_BASIC` | E − 1 міс. | current, decision table | 2026-09-20 |
| `PL-ADV` | `R_ADVANCED` | E − 1 міс. | current | 2026-09-20 |
| `PL-BVLOS` | `R_BVLOS` | E − 1 міс. | current, золота комбінація для всіх зон | 2026-09-20 |
| `PL-EDGE` | `R_BVLOS` | (E + 25 днів) − 18 міс. | мітка = 2026-11-14 → `W` 2026-11-13 pass (`GP-03`), `W` 2026-11-15 fail (`GP-04`), `W` 2026-11-14 (`GP-05`, F-4) | 2025-05-14 |
| `PL-LAPSE` | `R_BVLOS` | E − 18 міс. + 5 днів | current на момент submit, мітка 2026-10-25 < `W` 2026-11-10 (`GP-07`) | 2025-04-25 |
| `PL-V14` | `R_BVLOS` | W − 20 міс. | регрес 24 міс. (`GP-08`); `W` 2026-11-10 | 2025-03-10 |
| `PL-EOM` | `R_BVLOS` | 2025-05-31 | «2026-11-31» не існує; `W` 2026-11-30 / 2026-12-01 (`GP-06`, F-4) | 2025-05-31 |
| `PL-INACT` | `R_BVLOS` | E − 1 міс., `status = INACTIVE` | `GP-09`; спосіб встановити статус — out-of-band (немає API) | 2026-09-20 |

Усі `W` у прикладах лежать у межах 6 год – 60 днів від `E`, щоб `TIME_WINDOW` не заважав.

### 6.4 Дрони

| `mtom_grams` | Очікуваний бенд | Де використовується |
|---|---|---|
| 450 | `C0` | золота комбінація, `GA-07` |
| 900 / 901 | `C0` / `C1` | `GA-01`, `GA-09` |
| 2 000 | `C1` | `GA-07` |
| 3 500 / 3 501 | `C1` / `C2` | `GA-02`, `GA-09` |
| 3 800 | `C2` (у v1.4 був би `C1`) | `GA-06` |
| 8 000 | `C2` | `GA-07` |
| 12 000 / 12 001 | `C2` / `C3` | `GA-03`, `GA-04`, `GA-09` |
| 20 000 | `C3` | `GA-07` |
| 30 000 | `C3` | `GA-04`, `GA-09` |
| 450, `status = RETIRED` | `C0` | `GA-08`; статус — out-of-band (немає API) |
| `AC-OVL` 450 (окремий дрон) | `C0` | усі overlap-сценарії; `AC-OVL-2` — другий дрон для `GT-25` |

Маса не редагується через API (у `03` немає update для aircraft), тому для кожної маси потрібен окремий дрон.

### 6.5 Золота комбінація (усі гейти pass)

Пілот `PL-BVLOS`, дрон 450 г (`C0`), висота 30 м, оператор рівня ≥ required зони, вікно 60 хв о 12:00 локального часу, `W` = `T` + 3 дні, без інших заявок цього дрона в `UNDER_REVIEW`/`APPROVED`. Вона проходить у всіх 6 зонах. У кожному сценарії змінюється рівно один параметр.

### 6.6 Часові вікна

| Параметр | Pass | Fail | Двозначно |
|---|---|---|---|
| Lead time (`W − T`) | 6 год 00 хв, 6 год 01 хв | 5 год 59 хв; `W` < `T` | секунди на межі (F-5) |
| Advance (`W − T`) | 30 днів | 61 день | рівно 60 днів / +1 хв (F-6) |
| Тривалість | 15, 16, 239, 240 хв | 1, 14, 241 хв | 0 хв і від'ємна (F-7) |
| Curfew `KBP`/`ODS` (local) | 22:00–22:45, 05:15–06:00, 12:00–13:00 | 22:01–23:01, 04:59–05:59, 23:30–00:30, 00:30–02:30, 03:00–04:59 | 22:00–23:00, 05:00–06:00 (F-8) |
| Curfew `LVV`/`ZTM` (local) | 21:00–21:59, 06:01–07:00 | 21:01–22:01, 05:59–06:59, 23:00–00:30, 02:00–03:00 | 21:00–22:00, 06:00–07:00 (F-8) |
| Overlap (інша заявка 10:00–14:00 UTC) | 14:00–15:00, 09:00–10:00 | 13:59–15:00, 11:00–12:00, 10:00–14:00 | — |

**UTC для curfew.** Часовий пояс зон у docs не вказано (`F-3`). Перед виконанням `GT-11…GT-17` власник має підтвердити пояс, і лише після цього UTC-значення переносяться в тест-кейси. Ілюстрація під *припущенням* «Europe/Kyiv, зимовий час UTC+2» (не з docs): локальне 22:00–22:45 = 20:00–20:45 UTC. Дати curfew-тестів брати поза ніччю переходу DST (ніч переходу — окремий `GT-19`).

## 7. Середовище та передумови

- Розгорнутий стенд SkyPermit із засіяним каталогом зон (`supabase/seed.sql`), значення збігаються з §6.1. Перевіряється через `GET /zones`.
- REST-клієнт із підтримкою cookie-сесії; довільний рядок для `X-Reviewer-Id` (напр. `qa-reviewer-1`).
- **Out-of-band підготовка даних** (через продукт неможлива): оператори `L2`/`L3`, оператор `SUSPENDED`, пілот `INACTIVE`, дрон `RETIRED`, неактивна зона. Потрібен адміністратор стенду з правом змінювати ці поля. QA-кіт у Supabase не звертається.
- Read-only доступ до таблиць `audit_log` і `review_action` стенду для `PIPE-15/16` і `VAL-01/08` (у API читання немає).
- Доступ до maintenance completion endpoint, щоб отримати заявку в `COMPLETED` (`GT-24`, `SM-04`, `VAL-10`).
- Синхронізований годинник тестувальника із сервером. Для точних меж lead time (`GT-01/02`, ±1 хв) бажано керувати серверним часом. Без цього беруться значення з запасом ±2 хв, і це фіксується в результатах.

## 8. Критерії входу, виходу, призупинення

- **Вхід:** стенд доступний; `GET /zones` повертає 6 зон з §6.1; дані §6.2–6.4 підготовлені; є підтвердження поясу зон (`F-3`) — інакше curfew-група `GT-11…GT-19` не стартує, решта стартує.
- **Вихід:** виконано і пройдено всі P1 `Ready`; щонайменше 90 % P2 `Ready` пройдено, решта має заведені дефекти; немає відкритих дефектів P1; для кожного `Blocked`/`Needs clarification` є посилання на питання власнику.
- **Призупинення:** відсутні або неправильні засіяні зони; `422` з `failing_gate`, якого немає в enum; `submit` з будь-якого стану, крім `DRAFT`, змінює стан (тоді результати решти сценаріїв ненадійні). **Відновлення** — після виправлення і повторного проходу `TS-PIPE-14` та `TS-SM-01…04`.

## 9. Ризики та припущення

| Ризик / припущення | Вплив | Пом'якшення |
|---|---|---|
| Пояс зон невідомий (`F-3`) | curfew-тести можуть дати хибні pass/fail через зсув на 2–3 год | вхідний критерій; ілюстрація в §6.6 явно позначена як припущення |
| Неможливо точно влучити в межу lead time вручну | межі 6 год ±1 хв перевіряються з похибкою | керування часом на стенді або запас ±2 хв з позначкою в звіті |
| Out-of-band дані (`L2`/`L3`, `SUSPENDED`, `INACTIVE`, `RETIRED`, неактивна зона) не підготовлено | блокуються `GZ-04/05/06`, `GP-09`, `GA-08`, `SUSP-01`, сценарії `CONTROLLED`/`RESTRICTED` | заявка на підготовку даних до старту |
| Немає читання `AuditLog`/`ReviewAction` через API | побічні ефекти `validate` не перевірити | read-only доступ до БД стенду або позначка «не виконано» |
| Overlap: код і гейт суперечать між `02`/`03`/`06` (`F-1`, `F-2`) | дефект можуть завести на коректну реалізацію | фактичну поведінку перевіряти лише після відповіді власника; до того фіксувати як спостереження |
| Дані з `OH-1` v1.4 / `CR-1` у старих фікстурах | хибні очікування (24 міс., межа 4 000 г, C4/C5) | регресійні сценарії `GP-08`, `GA-06`; бенди тільки `C0`–`C3` |
| Тестові дати прив'язані до `E` | застарілі абсолютні дати в кейсах | у кейсах зберігати формули відносно `E` |

## 10. Заблоковані області (відкриті питання з `05`)

| OQ | Питання (з `05`) | Які сценарії заблоковано | Що потрібно для розблокування |
|---|---|---|---|
| OQ-4 | `Operator.status = SUSPENDED` — статус-код і форма відповіді | `TS-SUSP-02` (і частково `TS-VAL-17` через `F-18`) | рішення власника API-контракту щодо коду й тіла; `403` у `03` — лише пропозиція-заглушка |
| OQ-5 | Zone `active` — цитата | `TS-GZ-07` (правило — `TS-GZ-06` — тестується) | підтвердження `ZC-2026.1 §4` як `rule_reference` |

## 11. Знахідки в документації

| ID | Опис | Де (файл + розділ, обидві сторони) | Вплив на тестування | Питання власнику |
|---|---|---|---|---|
| `F-1` | Overlap віднесено до різних гейтів | `02` «Rules that affect the state machine» («inside the `AIRCRAFT` / `TIME_WINDOW` gates») vs `06` §1.4, §6 (лише `TIME_WINDOW`, `TSD-4 §5`) | `failing_gate` для overlap невідомий; залежить і first failure при одночасній відмові `AIRCRAFT`/`ZONE` | Overlap — це завжди `TIME_WINDOW`? |
| `F-2` | Overlap як `409` чи `422` | `03` Error semantics («`409` … scheduling conflict between requests») vs `06` §1.4/§6 (умова гейту, `rule_reference` `TSD-4 §5`) | від цього залежить код, тіло і кінцевий стан (`REJECTED` при `422` чи `DRAFT` при `409`) | Який код і стан на `submit`/`validate` з overlap? |
| `F-3` | Не вказано часовий пояс зон; у `Zone` немає поля поясу | `01` Zone (лише `curfew_start`/`curfew_end`), `06` §1.4 «On UTC vs. zone-local time» | неможливо обчислити UTC-вікна для curfew; ризик DST | Який пояс у кожної зони (Europe/Kyiv?) і як враховується DST? |
| `F-4` | «18 місяців від `certified_on`» не формалізовано: чи current сам день мітки; що робити з неіснуючою датою (31-ше, 29.02); `certified_on` — дата, `window_start` — UTC timestamp, у якому поясі порівнюються | `06` §2; `01` Pilot | межа currency визначена лише ±1 день | Чи current пілот, якщо `window_start` припадає на день мітки? Як зсувається кінець місяця? |
| `F-5` | Точка відліку lead time для `validate` (і для повторного `validate` на `UNDER_REVIEW`) не визначена; точність порівняння (хв/с) не вказана | `06` §1.4 («between submission and `window_start`»); `03` endpoint 13 | `validate` і `submit` можуть законно розійтися біля межі 6 год | Від чого рахується lead time у `validate`? |
| `F-6` | «≤ 60 days before `window_start`» — не названо точку відліку (подача?) і що таке день (24 год чи календарний, DST) | `06` §1.4 (`TSD-4` §2.2) | точна межа advance не тестується | Від якого моменту і в яких одиницях рахувати 60 днів? |
| `F-7` | `window_end ≤ window_start`: обмеження сутності (→ `400` на create?) чи умова гейту `TSD-4 §3` (→ `422` на submit)? | `01` FlightRequest («must be after»); `06` §1.4 Duration («strictly after»); `03` Error semantics | невідомо, на якому кроці і з яким кодом очікувати відмову | Де валідується порядок кінців вікна? |
| `F-8` | Включність меж curfew не визначена: чи 23:00 і 05:00 належать curfew | `06` §1.4 Curfew («No part of the window may fall inside») | дотичні вікна (22:00–23:00, 05:00–06:00) не мають очікуваного результату | Чи вважається вікно, що закінчується рівно на початку curfew / починається рівно на його кінці, допустимим? (для overlap docs це явно визначають, для curfew — ні) |
| `F-9` | Порядок умов **усередині** гейту не визначено (рейтинг vs currency; висота vs рівень vs active; lead/advance/duration/curfew/overlap) | `06` Evaluation order (лише між гейтами); `03` «The 422 body» (один `rule_reference`) | `rule_reference` при кількох відмовах в одному гейті невизначений | Який фіксований порядок умов усередині кожного гейту? |
| `F-10` | Контракт відповіді `validate` не описано: код при відмові; повертаються всі 4 результати чи тільки до першої відмови. `04` вимагає «per-gate pass/fail» і «all four gate results», а `03` — «exactly as submit would… stopping at first failure». `08` (похідний) показує і `422`, і «200 with per-gate results» | `03` endpoint 13, Gate pipeline; `04` Operator portal, Reviewer queue; `08` Gate Pipeline | неможливо перевірити формат і повноту результату `validate` | Що повертає `validate`, коли гейт падає, і чи оцінюються гейти після першої відмови? |
| `F-11` | `validate` для не-`DRAFT` станів: рев'юер викликає його на `UNDER_REVIEW` (`03`), але не сказано, чи заявка виключає себе з overlap і від якого моменту рахується lead time; для термінальних станів — `409` чи результат | `03` endpoint 13, Reviewer-or-owner reads; `02`; `06` §1.4 | на `UNDER_REVIEW` результат може бути хибно-негативним (самоперетин) | Як `validate` поводиться на кожному стані? |
| `F-12` | Не визначено зміст `rejection_reason` при відмові гейту (≠/= `message`) | `01` FlightRequest; `02` States | поле перевіряється лише на непорожність | Що записується в `rejection_reason`? |
| `F-13` | `AuditLog`: чи логується транзитний `SUBMITTED` окремим записом; значення `actor`. Ендпоінта читання `AuditLog` у `03` немає | `01` AuditLog; `02` States; `03` (відсутність) | побічні ефекти перевіряються тільки через БД | Скільки записів `AuditLog` на один `submit`? Чи є спосіб читати журнал без БД? |
| `F-14` | Тип і одиниці `requested_altitude_metres` не вказані (ціле/дробове, допустимість 0 і від'ємних) | `01` FlightRequest; `06` §1.3 | невідомо, чи 120.5 м — `400` чи `422 ZONE` | Які значення висоти допустимі? |
| `F-15` | `data-testid` для «Check before submitting», результатів гейтів і полів відмови не задано (у `04` є лише приклад і кілька auth-значень) | `04` Two contracts | UI-кейси не мають еталонних селекторів | Який перелік `data-testid` для форми заявки? |
| `F-16` | Код і тіло успішного `submit` не описані в `03`; `08` (похідний) каже «200/201» | `03` endpoint 10; `08` Gate Pipeline | очікуваний статус успіху невідомий | `200` чи `201`, і що в тілі? |
| `F-17` | `submit` — «owning operator only», а `401` означає «немає сесії **і** немає `X-Reviewer-Id`». Що повертати при наявному reviewer-заголовку без сесії — не визначено | `03` endpoint 10 vs Error semantics | `TS-AUTH-03` без очікування | `401` чи `403`? |
| `F-18` | `SUSPENDED` — precondition на create/submit/approve; `validate` не згадано | `05` §1; `07` п.3; `03` endpoint 13 | невідомо, чи блокується `validate` | Чи блокує `SUSPENDED` `validate`? |
| `F-19` | Не описано, що буде, якщо в заявці вказано пілота/дрон іншого оператора (у UI pickers скоуплені, у API обмеження не описане), а отже — які дані бачать гейти | `03` endpoint 9, Enforcement; `04` Request creation form | можна обійти гейти через чужі дані; поза обсягом `FR`, але впливає на `GP`/`GA` | Чи перевіряється належність пілота/дрона оператору на create/submit? |
| `F-20` | `03` подає `403` як відповідь для `SUSPENDED`, хоча `05` вважає код невизначеним | `03` Error semantics vs `05` §1 | за пріоритетом статус береться з `05` → `TS-SUSP-02` Blocked | Оновити `03` або закрити OQ-4 |

Додатково для `references/coverage-areas.md` скіла: гаряча точка «вікно охоплює curfew повністю, з кінцями поза ним» нездійсненна з поточними даними (найкоротший curfew — 6 год, найдовше вікно — 240 хв). Розбіжностей значень між coverage-areas і docs не знайдено.

## 12. Підсумок

| Область | Всього | Ready | Blocked | Needs clarification |
|---|---|---|---|---|
| `GP` | 11 | 8 | 0 | 3 |
| `GA` | 9 | 9 | 0 | 0 |
| `GZ` | 9 | 6 | 1 | 2 |
| `GT` | 27 | 20 | 0 | 7 |
| `PIPE` | 16 | 14 | 0 | 2 |
| `VAL` | 17 | 12 | 0 | 5 |
| `SM`/`AUTH`/`SUSP` | 12 | 9 | 1 | 2 |
| **Разом** | **101** | **78** | **2** | **21** |

За пріоритетом: P1 — 65 (Ready 58, Blocked 1, Needs clarification 6); P2 — 30 (Ready 17, Blocked 1, Needs clarification 12); P3 — 6 (Ready 3, Needs clarification 3).
