# TP-operator-access: Акаунт оператора — реєстрація, сесія, ізоляція даних, `SUSPENDED`

| Поле | Значення |
|---|---|
| ID | `TP-operator-access` |
| Тип | Фічевий |
| Версія | 0.1 |
| Дата | 2026-10-20 |
| Автор | згенеровано Claude |
| Джерела | `../skypermit/docs/` — `01-data-model.md`, `02-state-machine.md`, `03-api-contract.md`, `04-frontend-spec.md`, `05-open-questions.md`, `06-business-rules.md`, `07-clarifications.md`; допоміжно `../skypermit/CLAUDE.md`. Коміт `../skypermit`: `78407f1` |

## 1. Мета

Підтвердити, що оператор може зареєструватися, увійти й вийти, що всі operator-scoped дані (пілоти, дрони, заявки, профіль) ізольовані між операторами на сервері (`401`/`403`), і що `SUSPENDED` оператор не може виконувати заборонені записи. План також фіксує, яку частину поведінки для `SUSPENDED` поки тестувати не можна (`05` п.4), і прогалини в docs щодо auth. Результат — рішення про приймання фічі «operator auth».

## 2. Обсяг

### 2.1 В обсязі

| AREA | Що саме |
|---|---|
| `AUTH` | `POST /operators` (реєстрація = створення оператора), `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`; `401` vs `403`; доступ до чужих `pilots`/`aircraft`/`flight-requests`/`operators`; reviewer-or-owner reads; scoping списків; server-derived `operator_id`; `authorisation_level = L1` на реєстрації |
| `SUSP` | `Operator.status = SUSPENDED` на create / submit / approve, а також неописані операції (виносяться як прогалини) |
| `UIO` (частково) | `/login`, `/register`, стан хедера, доступ до порталу без сесії, pickers і «My requests» лише зі своїми даними |
| `TID` (частково) | `operator-login-link`, `operator-register-link`, `operator-session-info`, `operator-logout-button`; відсутність `operator-id-input` |
| `AUD` (частково) | відсутність переходу/`AuditLog`/`ReviewAction` після відмовленого запису (чужий ресурс, `SUSPENDED`) |

### 2.2 Поза обсягом

- Логіка гейтів `PILOT`/`AIRCRAFT`/`ZONE`/`TIME_WINDOW`, BVA часових меж, повна матриця переходів `SM` — окремі плани. Тут заявки свідомо будуються так, щоб проходити всі гейти (див. §6), щоб відмова гарантовано йшла від auth/`SUSPENDED`, а не від гейту.
- Підвищення `authorisation_level` і зміна `Operator.status` — out-of-band адмін-дії, API для них немає (`03` Authentication & authorisation). Тут вони лише підготовка даних.
- Внутрішня реалізація Supabase Auth (RLS, service-role key), стійкість до brute force, продуктивність, безпека на рівні пентесту. Ризики, які видно з docs, винесено в §9 і §11.
- Автотести.

## 3. Об'єкт тестування

- Сутність `Operator` (`01`): `id` = id користувача Supabase Auth, `email` і `national_id` унікальні, `authorisation_level` (`L1`–`L3`), `status` (`ACTIVE`, `SUSPENDED`).
- Ендпоінти (`03`): auth-четвірка без номерів; #2–#6 (registry), #9–#14 (flight requests), #15–#16 (reviewer, `X-Reviewer-Id`).
- Правило enforcement (`03` Authentication & authorisation): `operator_id` береться із сесії і ніколи з тіла чи query; без сесії → `401`, чужий ресурс → `403`; GET одного ресурсу і `validate` дозволені власнику або будь-якому `X-Reviewer-Id`.
- UI (`04` Operator portal; Two contracts): сторінки `/login`, `/register`, хедер з auth-елементами, pickers тільки з поточного оператора.

## 4. Підхід

- **API, мануально** через REST-клієнт з cookie jar (Postman/Insomnia/curl). Для кожного оператора своє cookie-середовище, для рев'юера — окремий профіль тільки з `X-Reviewer-Id`.
- **UI**: браузер, два профілі (або incognito) для двох операторів.
- **Техніки:**
  - *Decision table «хто × що»*: стовпці — без сесії / власник / інший оператор / тільки `X-Reviewer-Id` / сесія + `X-Reviewer-Id`; рядки — кожен ендпоінт. Це основна матриця §5.3.
  - *Equivalence partitioning* для ідентифікаторів: свій id / чужий id / неіснуючий UUID / не-UUID.
  - *Negative testing*: `400`/`401`/`403`/`404`, підкладений `operator_id`, підкладений `authorisation_level`, дублікати `email`/`national_id`.
  - *State-based*: стан сесії (немає → логін → логаут) і `Operator.status` (`ACTIVE` → `SUSPENDED` → `ACTIVE`).
- **Побічні ефекти**: після кожної відмовленої мутації перевіряти через `GET` від власника або рев'юера, що `state`, `submitted_at`, `decided_at`/`decided_by`, `failing_gate`/`rule_reference` не змінилися. `AuditLog` і `ReviewAction` в API не мають read-ендпоінтів, тому їх перевіряють читанням з БД стенду (див. §7).

## 5. Тестові сценарії та матриця покриття

Пріоритети: P1 — порушення доступу або недозволений запис, втрата аудиту; P2 — некоректні коди/повідомлення, UI-флоу; P3 — косметика/регрес селекторів.

### 5.1 `AUTH` — реєстрація (`POST /operators`, `/register`)

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-AUTH-01` | Валідна реєстрація створює оператора і сесію: відповідь успішна, cookie встановлено, одразу після цього `GET /auth/me` повертає профіль з тими самими `email`, `legal_name`, `national_id` | EP (valid) | P1 | `03` Authentication & authorisation (`POST /operators`); `04` Operator portal | Ready |
| `TS-AUTH-02` | `id` оператора з `/auth/me` можна використати в `GET /operators/{id}` від себе → `200` (оператор keyed by auth identity, окремого `user_id` немає) | EP | P2 | `01` Operator.`id`; `03` #2 | Ready |
| `TS-AUTH-03` | Новий оператор завжди `authorisation_level = L1`, зокрема коли в тіло реєстрації підкладено `"authorisation_level": "L3"` або `"L2"`. Оператор з рівнем вище `L1` не створюється в жодному разі | Negative | P1 | `03` «`authorisation_level` is no longer self-service»; `04` Operator portal | Ready |
| `TS-AUTH-04` | Відповідь на підкладене поле `authorisation_level`: його проігнорують чи повернуть `400` | Negative | P3 | — | Needs clarification (F-10) |
| `TS-AUTH-05` | Початковий `status` нового оператора (очікувано `ACTIVE`, але docs цього не кажуть) | EP | P2 | `01` Operator.`status` | Needs clarification (F-3) |
| `TS-AUTH-06` | Повторна реєстрація з наявним `email` не створює другого оператора: `GET /auth/me` старого акаунта не змінився, нова сесія для другого профілю не з'являється | Negative | P1 | `01` Operator.`email` unique | Ready |
| `TS-AUTH-07` | Повторна реєстрація з наявним `national_id` (інший `email`) не створює другого оператора | Negative | P1 | `01` Operator.`national_id` unique | Ready |
| `TS-AUTH-08` | Статус-код і тіло для дубліката `email` / `national_id` (`400`? `409`? `422`?) | Negative | P2 | `03` Error semantics не покриває | Needs clarification (F-4) |
| `TS-AUTH-09` | Атомарність: реєстрація, що провалилася на профілі (дубль `national_id` з новим `email`), не лишає «осиротілого» auth-користувача: `POST /auth/login` з цими `email`/паролем не дає сесії | Negative | P1 | `03` «creates the Supabase Auth user *and* the Operator profile in one request»; поведінка при збої не описана | Needs clarification (F-4) |
| `TS-AUTH-10` | Синтаксично зламане тіло (не JSON; немає `email` або `password`) → `400`, оператора не створено | Negative | P2 | `03` Error semantics (`400` Malformed request body) | Ready |
| `TS-AUTH-11` | Обов'язковість окремих полів профілю (`phone`, `address_line`, `city`, `postal_code` …), формат `email`, політика пароля | EP/BVA | P2 | `01` Operator (nullable не вказано) | Needs clarification (F-4) |
| `TS-AUTH-12` | UI `/register`: одна комбінована форма (email, password + поля профілю), поля `authorisation_level` немає; після submit оператор одразу залогінений: видно `operator-session-info` і `operator-logout-button` | UI | P2 | `04` Operator portal; Two contracts | Ready |

### 5.2 `AUTH` — логін, логаут, `/auth/me`

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-AUTH-13` | `POST /auth/login` з валідними email + password ставить session cookie; `GET /auth/me` → профіль саме цього оператора | EP (valid) | P1 | `03` Authentication & authorisation | Ready |
| `TS-AUTH-14` | Логін з неправильним паролем / неіснуючим email не створює сесію: наступний `GET /auth/me` → `401` | Negative | P1 | `03` (`/auth/me` → профіль або `401`) | Ready |
| `TS-AUTH-15` | Статус-код і тіло відповіді `POST /auth/login` на невірні облікові дані; чи однакова відповідь для «неправильний пароль» і «немає такого email» | Negative | P2 | `03` не описує | Needs clarification (F-5) |
| `TS-AUTH-16` | `GET /auth/me` без сесії → `401` | Negative | P1 | `03` Authentication & authorisation | Ready |
| `TS-AUTH-17` | Після `POST /auth/logout`: `GET /auth/me` → `401`; `GET /pilots`, `POST /flight-requests` → `401` | State-based | P1 | `03` (`/auth/logout` clears the session; no session → `401`) | Ready |
| `TS-AUTH-18` | Повторне використання cookie, збереженого до логауту (replay), після `POST /auth/logout` | Negative | P1 | `03` не описує інвалідацію на сервері | Needs clarification (F-5) |
| `TS-AUTH-19` | `POST /auth/logout` без сесії; закінчення терміну сесії; логін іншим оператором поверх активної сесії | Negative | P3 | `03` не описує | Needs clarification (F-5) |
| `TS-AUTH-20` | UI, логаут-стан: є `operator-login-link` і `operator-register-link`, немає `operator-session-info` / `operator-logout-button`; у залогіненому стані — навпаки | UI | P2 | `04` Two contracts (виняток для auth) | Ready |
| `TS-AUTH-21` | UI: натискання `operator-logout-button` повертає хедер у логаут-стан, сторінки порталу більше не показують даних оператора | UI | P2 | `04` Operator portal («Requires login») | Ready |
| `TS-AUTH-22` | UI: прямий перехід на сторінки порталу (реєстр пілотів/дронів, форма заявки, «My requests») без сесії не показує даних жодного оператора | UI, Negative | P1 | `04` Operator portal («Requires login»); `03` (`401` без сесії) | Ready |
| `TS-AUTH-23` | UI: яка саме реакція на сторінку порталу без сесії (redirect на `/login`, повідомлення, порожня сторінка) і як перемикач ролей у хедері взаємодіє з сесією оператора | UI | P3 | `04` не описує | Needs clarification (F-11) |

### 5.3 `AUTH` — доступ до чужих ресурсів (decision table)

Позначення: **A** — власник ресурсу (сесія оператора A), **B** — інший `ACTIVE` оператор (сесія B), **∅** — без сесії й без заголовка, **R** — тільки `X-Reviewer-Id`, без сесії.

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-AUTH-24` | **∅ → `401`** на кожному operator-scoped ендпоінті: `POST`/`GET /pilots`, `GET /pilots/{id}`, `POST`/`GET /aircraft`, `GET /aircraft/{id}`, `POST`/`GET /flight-requests`, `GET /flight-requests/{id}`, `.../submit`, `.../cancel`, `.../validate`, `GET /operators/{id}`. Жоден запис не створюється, стан не змінюється | Decision table, Negative | P1 | `03` Authentication & authorisation (Enforcement); Error semantics (`401`) | Ready |
| `TS-AUTH-25` | **B → `GET /pilots/{id_A}` → `403`**; у тілі тільки error detail, жодного поля пілота A | Decision table | P1 | `03` Enforcement; Registry #4 | Ready |
| `TS-AUTH-26` | **B → `GET /aircraft/{id_A}` → `403`**, без даних дрона A | Decision table | P1 | `03` Enforcement; Registry #6 | Ready |
| `TS-AUTH-27` | **B → `GET /flight-requests/{id_A}` → `403`**, без даних заявки A | Decision table | P1 | `03` Enforcement; #11 | Ready |
| `TS-AUTH-28` | **B → `GET /operators/{id_A}` → `403`** (дозволено лише «self, or any reviewer») | Decision table | P1 | `03` Registry #2 | Ready |
| `TS-AUTH-29` | **B → `POST /flight-requests/{id_A}/submit` (A в `DRAFT`) → `403`**; заявка лишається `DRAFT`, `submitted_at` порожній, нових записів `AuditLog` немає | Decision table, Negative | P1 | `03` #10 («owning operator only»); Enforcement; `01` AuditLog | Ready |
| `TS-AUTH-30` | **B → `POST /flight-requests/{id_A}/cancel`** для заявки A в `DRAFT`, `UNDER_REVIEW` і `APPROVED` (> 2 год до `window_start`) → `403`; стан не змінюється, `AuditLog` без нових записів | Decision table, Negative | P1 | `03` #14 («owning operator only»); `02` таблиця переходів | Ready |
| `TS-AUTH-31` | **B → `POST /flight-requests/{id_A}/validate` → `403`**; стан заявки A не змінився | Decision table | P1 | `03` Reviewer-or-owner reads; #13 | Ready |
| `TS-AUTH-32` | `GET /pilots` і `GET /aircraft` від A містять тільки записи A (жодного запису B), у кожному рядку є похідне поле (currency / weight band) | EP | P1 | `03` Registry, «Extension beyond the original 16» | Ready |
| `TS-AUTH-33` | `GET /flight-requests` від A повертає тільки заявки A; з `?operator_id=<id_B>` — так само тільки заявки A (scope примусово з сесії) | Negative | P1 | `03` Reviewer-or-owner reads («forced to the session's operator_id»); #12 | Ready |
| `TS-AUTH-34` | Підкладений `operator_id=<id_B>` у тілі `POST /pilots`, `POST /aircraft`, `POST /flight-requests` від A: запис або належить A, або запит відхилено; запису, що належить B, ніколи не створено | Negative | P1 | `03` Enforcement («none of them accept it as a request field») | Ready |
| `TS-AUTH-35` | Реакція на підкладений `operator_id` (ігнор чи `400`) | Negative | P3 | `03` не уточнює | Needs clarification (F-10) |
| `TS-AUTH-36` | A створює заявку з `pilot_id` та/або `aircraft_id`, що належать B: результат (`403` на create? прийнято і падає на гейті? прийнято без помилки?) | Negative | P1 | `03` Enforcement (загальне правило), `06` §1 гейтів для цього немає | Needs clarification (F-9) |
| `TS-AUTH-37` | Неіснуючий UUID у `GET /pilots/{id}`, `/aircraft/{id}`, `/flight-requests/{id}`, `/operators/{id}` від A → `404` | EP | P2 | `03` Error semantics (`404`) | Ready |
| `TS-AUTH-38` | **R** читає ресурси будь-якого оператора: `GET /pilots/{id}`, `/aircraft/{id}`, `/operators/{id}`, `/flight-requests/{id}` → `200`; `POST .../validate` → результат гейтів, стан не змінився | Decision table | P2 | `03` Reviewer-or-owner reads | Ready |
| `TS-AUTH-39` | **R** → `GET /flight-requests` повертає заявки всіх операторів; фільтр `?operator_id=` звужує до одного оператора | EP | P2 | `03` Reviewer-or-owner reads; #12 | Ready |
| `TS-AUTH-40` | **R** без сесії → `POST .../submit` / `.../cancel` (owner-only): `401`, `403` чи інше | Decision table | P2 | `03` #10, #14 vs Error semantics (`401` = «no session, no reviewer header») | Needs clarification (F-7) |
| `TS-AUTH-41` | **R** без сесії → `GET /pilots`, `GET /aircraft` (списки) | Decision table | P3 | `03` Registry extension описує тільки «logged-in operator» | Needs clarification (F-7) |
| `TS-AUTH-42` | **A** без `X-Reviewer-Id` → `POST /flight-requests/{id_A}/approve` і `.../reject` → `400`; заявка лишається `UNDER_REVIEW`, `ReviewAction` не створено. Оператор не може схвалити свою заявку сесією | Negative | P1 | `03` Reviewer («Return `400` if `X-Reviewer-Id` is missing»); `02` (approve/reject reviewer-only) | Ready |
| `TS-AUTH-43` | **∅** → `approve`/`reject`: `400` (розділ Reviewer) чи `401` (Error semantics) | Negative | P2 | `03` Reviewer vs `03` Error semantics | Needs clarification (F-2) |
| `TS-AUTH-44` | Сесія A + довільний `X-Reviewer-Id`: A читає дані B, бачить усі заявки в `GET /flight-requests`, схвалює власну заявку. За буквою docs заголовок приймається від будь-кого | Negative, security | P1 | `03` Reviewer-or-owner reads («any `X-Reviewer-Id` header»); Authentication history | Needs clarification (F-8) |
| `TS-AUTH-45` | UI: pickers пілота і дрона у формі заявки A не містять пілотів/дронів B (scope поточного оператора); при цьому немає фільтрації за бізнес-правилами (видно й «невалідні» свої) | UI | P2 | `04` Operator portal (pickers scoped), «The form does not pre-filter» | Ready |
| `TS-AUTH-46` | UI: «My requests», реєстри пілотів і дронів A не показують записів B | UI | P2 | `04` Operator portal | Ready |
| `TS-AUTH-47` | Чи вимагає `GET /zones` / `/zones/{code}` сесії | EP | P3 | `03` Reference data не каже | Needs clarification (F-12) |

### 5.4 `SUSP` — оператор зі `status = SUSPENDED`

Що вже вирішено (`05` §1, `07` п.3): `SUSPENDED` — це precondition на create / submit / approve, а не гейт; запит блокується і **не** має бути `422` з `failing_gate`. Що відкрито: статус-код і тіло відповіді (`05` п.4). Тому нижче «блокування відбулося» і «це не `422`/`failing_gate`» — `Ready`, а точний код і тіло — `Blocked (OQ-4)`.

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-SUSP-01` | `SUSPENDED` оператор S → `POST /flight-requests` з валідним тілом: заявка не створюється (немає в `GET /flight-requests` від рев'юера з `operator_id=S`), відповідь не 2xx і **не** `422` з `failing_gate` | Negative | P1 | `05` §1; `07` п.3 | Ready |
| `TS-SUSP-02` | Статус-код і тіло відповіді для create від S | Negative | P1 | `05` §1 | Blocked (OQ-4) — не визначено, див. `05` §1 |
| `TS-SUSP-03` | S → `POST .../submit` на своїй `DRAFT` (створеній до блокування), яка проходить усі гейти: заявка лишається `DRAFT`, `submitted_at` порожній, `failing_gate`/`rule_reference`/`rejection_reason` не заповнені, у `AuditLog` немає переходу; відповідь не `422` з `failing_gate` | Negative | P1 | `05` §1; `07` п.3; `03` Gate pipeline (SUSPENDED не частина pipeline) | Ready |
| `TS-SUSP-04` | Статус-код і тіло відповіді для submit від S | Negative | P1 | `05` §1 | Blocked (OQ-4) — не визначено, див. `05` §1 |
| `TS-SUSP-05` | Рев'юер → `POST .../approve` (з `X-Reviewer-Id` і `note`) заявки S, яка була в `UNDER_REVIEW` до блокування: стан лишається `UNDER_REVIEW`, `ReviewAction` не створено, `decided_at`/`decided_by` порожні; відповідь не `422` з `failing_gate` | Negative | P1 | `05` §1 (approve у переліку write operations); `07` п.3 | Ready |
| `TS-SUSP-06` | Статус-код і тіло відповіді для approve заявки S | Negative | P1 | `05` §1 | Blocked (OQ-4) — не визначено, див. `05` §1 |
| `TS-SUSP-07` | Docs-регрес: блокування S не повертає `403`, описаний у `03` Error semantics, *як вирішену поведінку*. Якщо система повертає `403`, у звіті відзначити це як placeholder, а не як «pass» | Negative | P2 | `03` Error semantics vs `05` §1 | Blocked (OQ-4) — не визначено, див. `05` §1 |
| `TS-SUSP-08` | S → `submit` на заявці, що вже `UNDER_REVIEW` (нелегальний перехід): `409` (`02`) чи відповідь для `SUSPENDED` — що перевіряється першим | Negative | P2 | `02` таблиця переходів; `05` §1 | Blocked (OQ-4) — не визначено, див. `05` §1 |
| `TS-SUSP-09` | S → `submit` на `DRAFT`, яка ще й валить гейт (напр. висота > стелі): заявка не має стати `REJECTED` з `failing_gate`; precondition перевіряється до pipeline? | Negative | P1 | `05` §1; порядок перевірок не описаний | Needs clarification (F-6) |
| `TS-SUSP-10` | Рев'юер → `reject` заявки S | Negative | P2 | `05` §1 називає тільки create / submit / approve | Needs clarification (F-6) |
| `TS-SUSP-11` | S → `cancel` своєї заявки (`DRAFT` / `UNDER_REVIEW` / `APPROVED`) | Negative | P2 | `05` §1 (cancel не названо) | Needs clarification (F-6) |
| `TS-SUSP-12` | S → `validate`; рев'юер → `validate` заявки S (чи показує UI рев'юера щось про блокування) | Negative | P3 | `05` §1 (validate не write) | Needs clarification (F-6) |
| `TS-SUSP-13` | S → `POST /pilots`, `POST /aircraft` | Negative | P2 | `05` §1 (не названо) | Needs clarification (F-6) |
| `TS-SUSP-14` | S → `POST /auth/login`, `GET /auth/me`, GET-читання своїх ресурсів і списків | EP | P2 | `05` §1 стосується тільки записів | Needs clarification (F-6) |
| `TS-SUSP-15` | Доля заявок S, що на момент блокування `UNDER_REVIEW`/`APPROVED`: чи лишаються в черзі рев'юера, чи враховуються в overlap для того ж дрона | EP | P2 | `02` overlap note; `05` §1 не описує | Needs clarification (F-6) |
| `TS-SUSP-16` | Після повернення S у `ACTIVE` (out-of-band) create і submit працюють звичайно: заявка створюється в `DRAFT`, submit веде до `UNDER_REVIEW` | State-based | P2 | `05` §1 (precondition тільки для `SUSPENDED`); `02` | Ready |
| `TS-SUSP-17` | Чи пишеться зміна `Operator.status` (`ACTIVE` ↔ `SUSPENDED`) в `AuditLog` | EP | P3 | `01` AuditLog («every state transition, on any entity») | Needs clarification (F-13) |
| `TS-SUSP-18` | UI порталу для S: що бачить оператор при спробі створити/подати заявку | UI | P2 | `04` не описує | Needs clarification (F-6) |
| `TS-SUSP-19` | Блокування S не впливає на інших операторів: A в той самий час створює і подає заявку успішно | EP | P2 | `05` §1 (precondition про оператора, що виконує запис) | Ready |

### 5.5 `TID` — селектори auth

| ID | Сценарій (test condition) | Техніка | Пріоритет | Джерело | Статус |
|---|---|---|---|---|---|
| `TS-TID-01` | Наявність і точні значення `operator-login-link`, `operator-register-link`, `operator-session-info`, `operator-logout-button` | UI | P2 | `04` Two contracts | Ready |
| `TS-TID-02` | `operator-id-input` відсутній на всіх сторінках (елемент видалено, не перейменовано) | UI, регрес | P3 | `04` Two contracts | Ready |
| `TS-TID-03` | Значення `data-testid` для полів і кнопок форм `/login` і `/register` (наявність можна перевірити, еталонних значень немає) | UI | P3 | `04` Two contracts | Needs clarification (F-11) |

## 6. Тестові дані

**Оператори**

| Ключ | Як отримати | `status` | `authorisation_level` | Навіщо |
|---|---|---|---|---|
| A | реєстрація через `/register` | `ACTIVE` | `L1` | власник ресурсів |
| B | реєстрація через `/register`, інші `email` і `national_id` | `ACTIVE` | `L1` | «чужий» оператор |
| S | реєстрація, далі **out-of-band** переведення в `SUSPENDED` (в API/UI немає способу) | `SUSPENDED` | `L1` | §5.4 |
| A-dup | `email` = A, новий `national_id`; і окремо `national_id` = A, новий `email` | — | — | `TS-AUTH-06`…`09` |

**Ресурси кожного з A, B, S** (створювати до блокування S):

| Ресурс | Значення | Обґрунтування |
|---|---|---|
| Пілот | `rating = R_BASIC`, `certified_on` в межах 18 міс. до `window_start` (напр. 2026-06-01 для вікон у листопаді 2026) | проходить `PILOT` у зоні `OPEN` (`06` §1.1, §2) |
| Дрон | `mtom_grams = 500` → `C0` | проходить `AIRCRAFT` в будь-якій зоні (`06` §3, §4) |
| Зона | `DNI-OPN-1` (`OPEN`, стеля 120 м, C0–C3, без curfew, `L1`) | проходить `ZONE` для `L1` (`06` §4) |
| Заявка | `requested_altitude_metres = 50`; `window_start` = зараз + 24 год, тривалість 60 хв; для кожного оператора окремий дрон, щоб не було overlap | проходить `TIME_WINDOW` (`06` §1.4); для cancel `APPROVED` — > 2 год до старту (`06` §5) |
| Стани заявок | для A і S: по одній `DRAFT`, `UNDER_REVIEW`, `APPROVED` | `TS-AUTH-29`…`31`, `TS-SUSP-03`, `-05`, `-11` |
| Заявка, що валить гейт | як вище, але `requested_altitude_metres = 121` | `TS-SUSP-09` (`ZONE`, `ZC-2026.1 §2`) |

**Інше:** значення `X-Reviewer-Id`, напр. `qa-reviewer-1`; неіснуючий UUID `00000000-0000-4000-8000-000000000000`.

## 7. Середовище та передумови

- Розгорнутий стенд SkyPermit з застосованими міграціями `0001`–`0004` і засіяним каталогом зон (`03` Reference data).
- REST-клієнт з окремим cookie jar на кожного оператора.
- Out-of-band доступ адміністратора стенду, щоб перевести S у `SUSPENDED` і назад. Без нього §5.4 не виконується.
- Read-доступ до БД стенду для `AuditLog` і `ReviewAction`: у API немає ендпоінтів для їх читання (`01` AuditLog; `03`).
- Два браузерні профілі для UI-сценаріїв.

## 8. Критерії входу, виходу, призупинення

- **Вхід:** стенд доступний; A, B, S і їхні ресурси підготовлені (§6); S переведено в `SUSPENDED`; є доступ до БД для перевірки аудиту.
- **Вихід:** усі P1 `Ready` виконано й пройдено; жодного відкритого дефекту P1 щодо доступу до чужих даних; кожен `Blocked`/`Needs clarification` має відкрите питання до власника (§10, §11).
- **Призупинення:** будь-який витік даних чужого оператора (`TS-AUTH-25`…`34`) або успішний запис S (`TS-SUSP-01`, `-03`, `-05`) — зупинити виконання цієї області й ескалювати як P1. Відновлення — після фіксу і повторного прогону всього §5.3.

## 9. Ризики та припущення

| Ризик / припущення | Вплив | Пом'якшення |
|---|---|---|
| `X-Reviewer-Id` — статичний рядок без автентифікації, і будь-хто може його надіслати (`03` Authentication history) | Ізоляція з §5.3 обходиться одним заголовком; оператор, можливо, схвалює власну заявку | `TS-AUTH-44` + F-8; рішення за власником, до цього результат не трактувати як pass/fail |
| Команда сприйме `403` з `03` як очікуваний код для `SUSPENDED` | Хибний pass для placeholder-поведінки | Тільки `Blocked (OQ-4)`, `TS-SUSP-07` |
| Немає API для `SUSPENDED` і для читання `AuditLog` | §5.4 і перевірки аудиту залежать від доступу до БД | Зафіксовано в §7 як залежність середовища |
| Припущення: у `TS-SUSP-05` precondition для approve стосується оператора-власника заявки (інший оператор в approve не бере участі) | Якщо власник docs мав на увазі інше, сценарій змінюється | Підтвердити в межах F-6 |
| Заявки в тестових даних завалять гейт через час виконання (lead time, currency) | Відмова через гейт замість auth плутає результат | Створювати вікна відносно дня прогону (§6) |

## 10. Заблоковані області (відкриті питання з `05`)

| OQ | Питання (з `05`) | Які сценарії заблоковано | Що потрібно для розблокування |
|---|---|---|---|
| OQ-4 | `Operator.status = SUSPENDED` — status code and response shape | `TS-SUSP-02`, `-04`, `-06`, `-07`, `-08` | Власник API contract фіксує код і тіло для create / submit / approve і порядок відносно `409` |

OQ-5 (zone `active` citation) не стосується цього обсягу.

## 11. Знахідки в документації

| ID | Опис | Де (файл + розділ, обидві сторони) | Вплив на тестування | Питання власнику |
|---|---|---|---|---|
| `F-1` | `03` описує `403` як відповідь для `SUSPENDED` («or the pre-existing `SUSPENDED`-operator precondition»), а `05` каже, що код не визначено, і `403` — лише placeholder | `03` Error semantics vs `05` §1, `07` п.3 | `403` не можна брати як очікуваний; `TS-SUSP-02/04/06/07` — Blocked | Прибрати згадку `SUSPENDED` з `403` в `03` або закрити OQ-4 |
| `F-2` | `approve`/`reject` без `X-Reviewer-Id` → `400` (розділ Reviewer), але `401` означає «no operator session and no `X-Reviewer-Id`» | `03` Reviewer vs `03` Error semantics | Невідомий код для анонімного approve/reject (`TS-AUTH-43`) | Який код при відсутності і сесії, і заголовка? |
| `F-3` | Початковий `Operator.status` при реєстрації не вказано (для `authorisation_level` сказано явно — `L1`) | `01` Operator; `03` Authentication & authorisation | `TS-AUTH-05` | Чи завжди новий оператор `ACTIVE`? |
| `F-4` | Помилки реєстрації не описані: код для дубля `email`/`national_id`, обов'язкові поля профілю, валідація email, політика пароля, атомарність «auth-user + профіль» при збої | `01` Operator (unique); `03` `POST /operators`, Error semantics | `TS-AUTH-08`, `-09`, `-11` | Описати контракт помилок `POST /operators` і rollback |
| `F-5` | Не описано: відповідь логіну на невірні облікові дані, logout без сесії, серверна інвалідація cookie, термін сесії | `03` Authentication & authorisation | `TS-AUTH-15`, `-18`, `-19` | Визначити коди й поведінку сесії |
| `F-6` | `SUSPENDED` описано тільки для create / submit / approve. Не сказано: reject, cancel, validate, `POST /pilots`/`aircraft`, login і читання; порядок відносно гейтів і `409`; доля заявок, що вже `UNDER_REVIEW`/`APPROVED`; UI | `05` §1; `07` п.3; `03` Gate pipeline; `04` | `TS-SUSP-09`…`15`, `-18` | Повний перелік операцій, які блокує `SUSPENDED`, і порядок перевірок |
| `F-7` | Owner-only ендпоінти (`submit`, `cancel`) і списки `GET /pilots`/`/aircraft` з `X-Reviewer-Id`, але без сесії: за визначенням `401` заголовок уже «автентифікує», а ресурс йому не дозволено | `03` #10, #14, Registry extension vs Error semantics | `TS-AUTH-40`, `-41` | `401` чи `403`? Чи доступні рев'юеру списки пілотів/дронів? |
| `F-8` | Будь-який `X-Reviewer-Id` дає читання всіх даних і дозвіл на approve/reject; нічого не заважає оператору додати його до своєї сесії й схвалити власну заявку | `03` Reviewer-or-owner reads, Reviewer, Authentication history | Ізоляція з §5.3 обходиться за дизайном; `TS-AUTH-44` | Це прийнятний ризик чи треба заборонити поєднання сесії оператора з заголовком рев'юера? |
| `F-9` | Не визначено, що буває при `POST /flight-requests` з `pilot_id`/`aircraft_id` іншого оператора. UI pickers обмежують вибір, API — невідомо; серед гейтів (`06` §1) немає перевірки належності | `03` Enforcement; `04` pickers; `06` §1 | `TS-AUTH-36` (P1: заявка на чужому дроні) | `403` на create, `422` з гейтом чи інше? |
| `F-10` | «Doesn't accept `operator_id`» / «`authorisation_level` is not a field» — не сказано, ігнорувати чи `400` | `03` Enforcement; `04` Operator portal | `TS-AUTH-04`, `-35` (P3; суть ізоляції тестується як Ready) | Ігнор чи `400`? |
| `F-11` | `04` не описує поведінку сторінки порталу без сесії (redirect?) і взаємодію перемикача ролей із сесією; `data-testid` для полів форм `/login`/`/register` не задано | `04` Operator portal, Two contracts | `TS-AUTH-23`, `TS-TID-03` — немає еталона для UI-кейсів | Додати значення селекторів і поведінку без сесії |
| `F-12` | Чи потрібна сесія для `GET /zones` — не сказано (у переліку operator-scoped ендпоінтів зон немає) | `03` Authentication & authorisation (Enforcement), Reference data | `TS-AUTH-47` | Зони публічні? |
| `F-13` | `AuditLog` пише «every state transition, on any entity», але незрозуміло, чи `Operator.status` вважається станом; механізму зміни статусу в API немає | `01` AuditLog, Operator; `03` | `TS-SUSP-17` | Чи логується `ACTIVE` ↔ `SUSPENDED`? |

## 12. Підсумок

Всього **69** сценаріїв.

| Статус | Кількість | P1 | P2 | P3 |
|---|---|---|---|---|
| Ready | 38 | 24 | 13 | 1 |
| Blocked (OQ-4) | 5 | 3 | 2 | 0 |
| Needs clarification | 26 | 5 | 12 | 9 |
| **Разом** | **69** | 32 | 27 | 10 |

Найбільший ризик для релізу — не відсутні відповіді по `SUSPENDED` (вони чесно заблоковані), а F-8 і F-9: за поточними docs ізоляцію між операторами можна обійти заголовком `X-Reviewer-Id` або посиланням на чужий дрон/пілота в новій заявці.
