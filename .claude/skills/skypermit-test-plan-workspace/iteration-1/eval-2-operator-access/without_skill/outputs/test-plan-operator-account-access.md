# Тест-план: акаунт оператора — реєстрація, логін/логаут, ізоляція даних, SUSPENDED

| | |
|---|---|
| Проєкт | SkyPermit |
| Тип тестування | Мануальне (UI + API через HTTP-клієнт, напр. Postman / curl) |
| Версія документації | `../skypermit/docs/01`–`08` станом на 2026-10-20 |
| Автор | Hanna (QA) |
| Статус документа | Чернетка v1 |

---

## 1. Мета та обсяг

Перевірити, що:

1. Оператор може зареєструватися (один запит створює Supabase Auth-користувача і профіль `Operator`), одразу отримує сесію, а `authorisation_level` завжди `L1`.
2. Логін, логаут і `GET /auth/me` працюють відповідно до контракту, а UI показує правильні елементи для залогіненого й незалогіненого стану.
3. Авторизація серверна: `operator_id` береться тільки із сесії. Без сесії оператор отримує `401`, до чужих пілотів, дронів, заявок і профілю оператора — `403`.
4. Ревʼюер (`X-Reviewer-Id`) читає дані будь-якого оператора, але читання не дає йому прав власника.
5. Поведінку для `Operator.status = SUSPENDED` перевірено в межах уже визначеного. Не визначене відкрите питання позначене як заблоковане.

### 1.1 Поза обсягом

- Логіка гейтів (`PILOT`/`AIRCRAFT`/`ZONE`/`TIME_WINDOW`), окрім того, що SUSPENDED **не** є гейтом.
- Повна матриця переходів state machine: тут лише ті переходи, що потрібні для перевірок доступу.
- Підвищення `authorisation_level` і зміна `status`: за docs це out-of-band адмін-дії, для них немає API.
- Навантажувальне тестування, безпека на рівні інфраструктури (RLS, service-role key), пряма робота з Supabase.

---

## 2. Джерела вимог

| Скорочення | Файл / розділ |
|---|---|
| DM | `../skypermit/docs/01-data-model.md`: Entities → Operator, Pilot, Aircraft, FlightRequest; Enumerations |
| SM | `../skypermit/docs/02-state-machine.md`: таблиця переходів |
| API-AUTH | `../skypermit/docs/03-api-contract.md`: «Authentication & authorisation» (включно з «Enforcement», «Reviewer-or-owner reads», «authorisation_level is no longer self-service») |
| API-REG | `../skypermit/docs/03-api-contract.md`: «Registry» і примітка «Extension beyond the original 16» |
| API-FR | `../skypermit/docs/03-api-contract.md`: «Flight requests», «Reviewer — requires X-Reviewer-Id» |
| API-ERR | `../skypermit/docs/03-api-contract.md`: «Error semantics» |
| FE | `../skypermit/docs/04-frontend-spec.md`: «Operator portal», «Two contracts to hold exactly» |
| OQ | `../skypermit/docs/05-open-questions.md`: рядок 4, §1 (канонічний статус) |
| CL | `../skypermit/docs/07-clarifications.md`: п. 3 |

Пріоритет джерел (згідно з `CLAUDE.md` цього репо): 07 > 06; статус питань береться тільки з 05; 01/02 > 08.

---

## 3. Тестове середовище та дані

### 3.1 Середовище
- Розгорнутий стенд SkyPermit (UI оператора, `/api/v1`), з наповненим каталогом зон (`seed.sql`).
- Браузер (будь-який сучасний) з DevTools, щоб бачити cookie сесії. HTTP-клієнт, що зберігає cookies між запитами.

### 3.2 Тестові дані (готуються до початку виконання)

| Позначення | Опис | Як отримати |
|---|---|---|
| **OP-A** | Оператор `ACTIVE`, зареєстрований через `/register` | Через UI/API у межах TC-REG-01 |
| **OP-B** | Другий оператор `ACTIVE` («чужий») | Через UI/API |
| **OP-S** | Оператор зі `status = SUSPENDED` | ⚠️ API для зміни статусу немає. Потрібен адміністратор стенду (див. **Q-10**) |
| PIL-A, AC-A | Пілот і дрон оператора OP-A | `POST /pilots`, `POST /aircraft` під сесією OP-A |
| PIL-B, AC-B | Пілот і дрон оператора OP-B | Те саме під сесією OP-B |
| FR-B-DRAFT | Заявка OP-B у стані `DRAFT` | `POST /flight-requests` під сесією OP-B |
| FR-B-UR | Заявка OP-B у стані `UNDER_REVIEW` | Валідна заявка в `KHR-OPN-4` / `DNI-OPN-1` (зони `L1`), `submit` |
| FR-S-UR | Заявка OP-S у стані `UNDER_REVIEW` (створена до призупинення) | Створити й засабмітити до переведення OP-S у `SUSPENDED` |
| FR-S-DRAFT | Заявка OP-S у `DRAFT` (створена до призупинення) | Те саме |
| REV | Значення заголовка `X-Reviewer-Id`, напр. `qa-reviewer-01` | Будь-який рядок |
| ID-NX | UUID, якого немає в БД | Згенерувати випадковий |

---

## 4. Підхід

- **API-рівень** є основним для перевірок `401`/`403`/`404`: UI за дизайном не дає вибрати чужі сутності (пікери вже відфільтровані за оператором), тому обхід можна перевірити тільки прямим запитом.
- **UI-рівень**: реєстрація, логін, логаут, видимість елементів за `data-testid`, «My requests», пікери.
- Кожен кейс має статус:
  - **Ready**: очікуваний результат повністю визначений у docs;
  - **Blocked**: залежить від відкритого питання в `05-open-questions.md`;
  - **Needs clarification**: docs поведінку не описують або описують суперечливо. Питання винесено в розділ 6.
- Пріоритет: **P1** (критичний, безпека/ізоляція), **P2** (основна функціональність), **P3** (граничні випадки).

---

## 5. Тест-кейси

### 5.1 Реєстрація (`POST /operators`, сторінка `/register`)

| ID | Пріор. | Назва | Передумови | Кроки | Очікуваний результат | Джерело | Статус |
|---|---|---|---|---|---|---|---|
| TC-REG-01 | P1 | Успішна реєстрація через UI | Користувач не залогінений; email і `national_id` ще не використані | 1. Відкрити головну сторінку, натиснути `operator-register-link`. 2. На `/register` заповнити email, password, `legal_name`, `national_id`, `phone`, `address_line`, `city`, `postal_code`. 3. Відправити форму | Профіль створено **одним** сабмітом форми; оператор одразу залогінений: у хедері є `operator-session-info` і `operator-logout-button`, немає `operator-login-link`/`operator-register-link` | FE «Operator portal», «Two contracts»; API-AUTH | Ready |
| TC-REG-02 | P1 | Реєстрація через API встановлює сесію | — | 1. `POST /api/v1/operators` з валідним тілом (credentials і поля профілю). 2. Тим самим клієнтом (з cookies) `GET /api/v1/auth/me` | Крок 1: у відповіді встановлено session cookie. Крок 2: повертається профіль щойно створеного оператора (не `401`) | API-AUTH (таблиця auth-ендпоінтів) | Ready (код успіху кроку 1 див. **Q-01**) |
| TC-REG-03 | P1 | `authorisation_level` завжди `L1` | — | 1. Зареєструвати оператора. 2. `GET /auth/me` або `GET /operators/{id}` | `authorisation_level = L1` | API-AUTH «authorisation_level is no longer self-service»; FE | Ready |
| TC-REG-04 | P1 | Спроба передати `authorisation_level` у тілі | — | 1. `POST /operators` з валідним тілом і `"authorisation_level": "L3"`. 2. Перевірити створений профіль (якщо створено) | Оператора з рівнем `L3` **не** створено. Якщо профіль створено, у нього `L1`. Чи поле ігнорується, чи запит відхиляється з `400`, у docs не сказано | API-AUTH; FE | Needs clarification (**Q-02**) |
| TC-REG-05 | P2 | На формі немає поля `authorisation_level` | Сторінка `/register` | Оглянути форму реєстрації | Поля для вибору `authorisation_level` немає | FE «Operator portal» | Ready |
| TC-REG-06 | P2 | `id` оператора збігається з auth-ідентичністю | Зареєстрований OP-A | 1. `GET /auth/me`. 2. `GET /operators/{id}` з `id` із кроку 1 | Обидва запити повертають той самий профіль; `id` один і той самий (окремого `user_id` у профілі немає) | DM «Operator» → `id` | Ready |
| TC-REG-07 | P2 | Дублікат email | Існує OP-A | `POST /operators` з email оператора OP-A та іншим `national_id` | Реєстрацію відхилено, другого оператора не створено. Код і тіло помилки в docs не визначені | DM «Operator» (`email` unique) | Needs clarification (**Q-03**) |
| TC-REG-08 | P2 | Дублікат `national_id` | Існує OP-A | `POST /operators` з новим email і `national_id` оператора OP-A | Реєстрацію відхилено, оператора не створено. Код не визначено | DM «Operator» (`national_id` unique) | Needs clarification (**Q-03**) |
| TC-REG-09 | P2 | Після невдалої реєстрації (дублікат `national_id`) не лишається «осиротілого» auth-користувача | Виконано TC-REG-08 | `POST /auth/login` з email і паролем із TC-REG-08 | Docs кажуть, що auth-користувач і профіль створюються «together in one request», але атомарність при помилці явно не описано | API-AUTH; FE | Needs clarification (**Q-04**) |
| TC-REG-10 | P2 | Некоректне тіло запиту | — | `POST /operators` з невалідним JSON / без обовʼязкових полів | `400` з validation detail | API-ERR (`400`) | Ready (перелік обовʼязкових полів і правил валідації див. **Q-05**) |
| TC-REG-11 | P3 | Початкові `status` і `registered_on` | Зареєстрований оператор | `GET /auth/me` | `status`, `registered_on` заповнені. Конкретні значення при реєстрації (`ACTIVE`? сьогоднішня дата?) у docs явно не вказано | DM «Operator» | Needs clarification (**Q-06**) |
| TC-REG-12 | P2 | На `/register` немає поля operator ID | — | Оглянути хедер і сторінки до та після логіну | Елемента `operator-id-input` немає ніде | FE «Two contracts to hold exactly» | Ready |

### 5.2 Логін / логаут / `GET /auth/me`

| ID | Пріор. | Назва | Передумови | Кроки | Очікуваний результат | Джерело | Статус |
|---|---|---|---|---|---|---|---|
| TC-AUTH-01 | P1 | Успішний логін через UI | Існує OP-A; користувач не залогінений | 1. Натиснути `operator-login-link`. 2. На `/login` ввести email і пароль OP-A. 3. Увійти | Залогінено: видно `operator-session-info` і `operator-logout-button`, посилань логіну та реєстрації немає | FE | Ready |
| TC-AUTH-02 | P1 | Логін через API встановлює сесію | Існує OP-A | 1. `POST /auth/login` (email і password). 2. `GET /auth/me` | Крок 1: встановлено session cookie. Крок 2: профіль OP-A | API-AUTH | Ready (код успіху кроку 1 див. **Q-01**) |
| TC-AUTH-03 | P1 | Логін із неправильним паролем | Існує OP-A | `POST /auth/login` з правильним email і хибним паролем | Сесію не встановлено; наступний `GET /auth/me` повертає `401`. Код відповіді самого `/auth/login` у docs не визначено | API-AUTH; API-ERR | Needs clarification (**Q-07**) |
| TC-AUTH-04 | P2 | Логін із неіснуючим email | — | `POST /auth/login` з незареєстрованим email | Як у TC-AUTH-03 | API-AUTH | Needs clarification (**Q-07**) |
| TC-AUTH-05 | P2 | Логін із некоректним тілом | — | `POST /auth/login` без поля `password` / з невалідним JSON | `400` | API-ERR (`400`) | Ready |
| TC-AUTH-06 | P1 | `GET /auth/me` без сесії | Нова сесія клієнта без cookies | `GET /auth/me` | `401` | API-AUTH («or `401`») | Ready |
| TC-AUTH-07 | P1 | Логаут через UI | OP-A залогінений | 1. Натиснути `operator-logout-button` | Сесію завершено: знову видно `operator-login-link` і `operator-register-link`, а `operator-session-info` і `operator-logout-button` зникли | FE | Ready |
| TC-AUTH-08 | P1 | Після логауту захищені ендпоінти дають `401` | OP-A залогінений (API) | 1. `POST /auth/logout`. 2. `GET /auth/me`. 3. `GET /pilots`. 4. `POST /flight-requests` з валідним тілом | Кроки 2–4: `401` | API-AUTH «Enforcement» | Ready |
| TC-AUTH-09 | P2 | Сторінки порталу оператора вимагають логіну | Не залогінений | Відкрити напряму сторінки реєстру пілотів/дронів, форму заявки, «My requests» | Доступ без логіну неможливий (portal «Requires login»). Конкретну поведінку (редірект на `/login` чи повідомлення) docs не описують | FE «Operator portal» | Needs clarification (**Q-08**) |
| TC-AUTH-10 | P2 | Повторний логін іншим оператором замінює сесію | OP-A залогінений | 1. `POST /auth/login` як OP-B. 2. `GET /auth/me`. 3. `GET /pilots` | Кроки 2–3 повертають дані OP-B, не OP-A | API-AUTH «Enforcement» (operator_id із сесії) | Ready |
| TC-AUTH-11 | P3 | Сесія переживає перезавантаження сторінки | OP-A залогінений в UI | Перезавантажити сторінку, відкрити нову вкладку | Оператор лишається залогіненим (`operator-session-info` видно), бо сесія тримається в cookie, а не у введеному вручну ID | FE (session cookie, «identity now comes from the session») | Ready |

### 5.3 Ізоляція даних: доступ до чужих пілотів, дронів, заявок і профілю

Для кожного кейсу ресурси оператора OP-B запитуються під сесією OP-A, якщо не вказано інше.

#### 5.3.1 Без автентифікації (`401`)

| ID | Пріор. | Назва | Кроки (без cookies і без `X-Reviewer-Id`) | Очікуваний результат | Джерело | Статус |
|---|---|---|---|---|---|---|
| TC-ACC-01 | P1 | Створення без сесії | `POST /pilots`, `POST /aircraft`, `POST /flight-requests` з валідними тілами | `401` на кожен; нічого не створено | API-AUTH «Enforcement»; API-ERR | Ready |
| TC-ACC-02 | P1 | Списки без сесії | `GET /pilots`, `GET /aircraft`, `GET /flight-requests` | `401` | API-AUTH; API-ERR | Ready |
| TC-ACC-03 | P1 | Читання одиничного ресурсу без сесії | `GET /pilots/{PIL-B}`, `/aircraft/{AC-B}`, `/flight-requests/{FR-B-DRAFT}`, `/operators/{OP-B}` | `401` | API-AUTH «Reviewer-or-owner reads»; API-ERR | Ready |
| TC-ACC-04 | P1 | Дії над заявкою без сесії | `POST /flight-requests/{FR-B-DRAFT}/submit`, `/validate`, `/cancel` | `401` | API-AUTH; API-ERR | Ready |

#### 5.3.2 Чужі ресурси (`403`)

| ID | Пріор. | Назва | Кроки (сесія OP-A) | Очікуваний результат | Джерело | Статус |
|---|---|---|---|---|---|---|
| TC-ACC-10 | P1 | Чужий пілот | `GET /pilots/{PIL-B}` | `403`; даних PIL-B у тілі немає | API-AUTH «Enforcement»; API-REG #4 | Ready |
| TC-ACC-11 | P1 | Чужий дрон | `GET /aircraft/{AC-B}` | `403` | API-AUTH; API-REG #6 | Ready |
| TC-ACC-12 | P1 | Чужий профіль оператора | `GET /operators/{OP-B}` | `403` | API-REG #2 («self, or any reviewer»); API-ERR | Ready |
| TC-ACC-13 | P1 | Свій профіль оператора | `GET /operators/{OP-A}` | Профіль OP-A | API-REG #2 | Ready |
| TC-ACC-14 | P1 | Чужа заявка: читання | `GET /flight-requests/{FR-B-DRAFT}` | `403` | API-FR #11 | Ready |
| TC-ACC-15 | P1 | Чужа заявка: submit | `POST /flight-requests/{FR-B-DRAFT}/submit` | `403`; стан FR-B-DRAFT лишається `DRAFT` (перевірити під OP-B або ревʼюером) | API-FR #10 («owning operator only») | Ready |
| TC-ACC-16 | P1 | Чужа заявка: validate | `POST /flight-requests/{FR-B-DRAFT}/validate` | `403` | API-FR #13 | Ready |
| TC-ACC-17 | P1 | Чужа заявка: cancel | `POST /flight-requests/{FR-B-UR}/cancel` | `403`; FR-B-UR лишається `UNDER_REVIEW` | API-FR #14 («owning operator only») | Ready |
| TC-ACC-18 | P1 | Заявка з чужим пілотом/дроном | `POST /flight-requests` під OP-A з `pilot_id = PIL-B`, `aircraft_id = AC-B` | Заявку з чужими сутностями не створено; очікувано `403` («operator reaching for another operator's resource»). Явно для полів тіла `pilot_id`/`aircraft_id` docs цього не формулюють | API-AUTH «Enforcement» | Needs clarification (**Q-09**) |
| TC-ACC-19 | P2 | Неіснуючий ресурс | `GET /pilots/{ID-NX}`, `/aircraft/{ID-NX}`, `/flight-requests/{ID-NX}` | `404` | API-ERR (`404`) | Ready |

#### 5.3.3 `operator_id` від клієнта ігнорується (server-derived)

| ID | Пріор. | Назва | Кроки (сесія OP-A) | Очікуваний результат | Джерело | Статус |
|---|---|---|---|---|---|---|
| TC-ACC-20 | P1 | `operator_id` у тілі `POST /pilots` | Створити пілота з `"operator_id": "<OP-B>"` | Пілота **не** створено під OP-B: або створено під OP-A, або запит відхилено (docs: ендпоінт «doesn't accept it as a request field»; що саме відбувається, не уточнено). `GET /pilots` під OP-B не показує нового пілота | API-AUTH «Enforcement» | Ready (інваріант) / Needs clarification щодо коду (**Q-02**) |
| TC-ACC-21 | P1 | `operator_id` у тілі `POST /aircraft` | Як TC-ACC-20 для дрона | Як TC-ACC-20 | API-AUTH | Ready (інваріант) / **Q-02** |
| TC-ACC-22 | P1 | `operator_id` у тілі `POST /flight-requests` | Створити заявку зі своїми пілотом/дроном і `"operator_id": "<OP-B>"` | Заявка не належить OP-B; у списку OP-B (`GET /flight-requests` під OP-B) її немає | API-AUTH; API-FR #9 | Ready (інваріант) / **Q-02** |
| TC-ACC-23 | P1 | `?operator_id=` у `GET /pilots` і `GET /aircraft` | `GET /pilots?operator_id=<OP-B>`, `GET /aircraft?operator_id=<OP-B>` | Повертаються тільки пілоти й дрони OP-A, query-параметр не впливає на скоуп | API-REG «Extension beyond the original 16» («no query param at all») | Ready |
| TC-ACC-24 | P1 | `?operator_id=` у `GET /flight-requests` без ревʼюера | `GET /flight-requests?operator_id=<OP-B>` | Тільки заявки OP-A (скоуп примусово з сесії) | API-AUTH «Reviewer-or-owner reads»; API-FR #12 | Ready |
| TC-ACC-25 | P2 | Списки містять лише свої записи | OP-A і OP-B мають по кілька пілотів, дронів і заявок | `GET /pilots`, `GET /aircraft`, `GET /flight-requests` під OP-A | Жодного запису OP-B; рядки пілотів і дронів містять похідні поля (currency, weight band) | API-REG «Extension»; API-FR #12 | Ready |

#### 5.3.4 UI: оператор бачить тільки своє

| ID | Пріор. | Назва | Кроки | Очікуваний результат | Джерело | Статус |
|---|---|---|---|---|---|---|
| TC-ACC-30 | P1 | Пікери пілота й дрона у формі заявки | Під OP-A відкрити форму створення заявки, розгорнути пікери пілота й дрона | Є тільки PIL-A/AC-A та інші сутності OP-A; PIL-B/AC-B відсутні | FE «Request creation form» («scoped to the current operator») | Ready |
| TC-ACC-31 | P2 | Пікери не фільтруються за бізнес-правилами | OP-A має пілота `R_BASIC` і дрон `C3` | Відкрити форму, вибрати `LVV-RST-2` (RESTRICTED) | Пілот `R_BASIC` і дрон `C3` доступні у пікерах (фільтрація тільки за оператором, не за правилами) | FE «The form does not pre-filter» | Ready |
| TC-ACC-32 | P1 | «My requests» показує лише свої | Під OP-A відкрити «My requests» | Заявок OP-B немає; фільтр за станом працює в межах заявок OP-A | FE «Operator portal»; API-FR #12 | Ready |
| TC-ACC-33 | P1 | Реєстри пілотів і дронів показують лише свої | Під OP-A відкрити сторінки реєстрів | Тільки сутності OP-A | FE; API-REG | Ready |
| TC-ACC-34 | P2 | Пряме відкриття деталей чужої заявки за URL | Під OP-A відкрити detail view заявки FR-B-DRAFT, підставивши її id в URL | Дані заявки не показуються (API повертає `403`). Як саме UI показує помилку, docs не описують | FE; API-FR #11 | Needs clarification (**Q-08**) |
| TC-ACC-35 | P2 | Новостворені сутності привʼязані до сесії | Під OP-A створити пілота й дрон через UI-форми, потім залогінитись як OP-B | Під OP-B нових сутностей у реєстрах і пікерах немає | API-AUTH «Enforcement» | Ready |

#### 5.3.5 Ревʼюер і межі його доступу

| ID | Пріор. | Назва | Кроки | Очікуваний результат | Джерело | Статус |
|---|---|---|---|---|---|---|
| TC-ACC-40 | P1 | Ревʼюер читає дані будь-якого оператора | Без сесії, з `X-Reviewer-Id: REV`: `GET /pilots/{PIL-B}`, `/aircraft/{AC-B}`, `/operators/{OP-B}`, `/flight-requests/{FR-B-UR}`, `POST /flight-requests/{FR-B-UR}/validate` | Успішні відповіді з даними; `validate` не змінює стан (FR-B-UR лишається `UNDER_REVIEW`) | API-AUTH «Reviewer-or-owner reads»; API-FR #13 | Ready |
| TC-ACC-41 | P1 | Ревʼюер бачить заявки всіх операторів і може фільтрувати за `operator_id` | `GET /flight-requests` з `X-Reviewer-Id`; потім `?operator_id=<OP-B>` | Без фільтра: заявки і OP-A, і OP-B. З фільтром: тільки OP-B | API-AUTH; API-FR #12 | Ready |
| TC-ACC-42 | P1 | Оператор не може approve/reject без `X-Reviewer-Id` | Під сесією OP-A (і навіть для своєї заявки в `UNDER_REVIEW`) `POST /flight-requests/{id}/approve` і `/reject` з `note` без заголовка | `400`; стан не змінився, `ReviewAction` не створено | API-FR «Reviewer — requires X-Reviewer-Id» («Return `400` if missing») | Ready |
| TC-ACC-43 | P2 | approve/reject зовсім без автентифікації | Без cookies і без заголовка: `POST .../approve` | Docs суперечать: розділ Reviewer каже `400` за відсутності заголовка, а API-ERR визначає `401` як «no operator session and no `X-Reviewer-Id`» | API-FR; API-ERR | Needs clarification (**Q-11**) |
| TC-ACC-44 | P2 | Ревʼюер на write-ендпоінтах оператора | Тільки `X-Reviewer-Id`, без сесії: `POST /pilots`, `POST /flight-requests`, `POST /flight-requests/{FR-B-DRAFT}/submit`, `/cancel` | Операція не виконується (немає сесії, з якої взяти `operator_id`; submit/cancel тільки для власника). Код (`401` чи `403`) docs не визначають | API-AUTH; API-FR #10, #14; API-ERR | Needs clarification (**Q-12**) |

### 5.4 Оператор у статусі `SUSPENDED`

> **Важливо.** За `05-open-questions.md` (рядок 4, §1) питання **відкрите**: статус-код і тіло відповіді для заблокованого запиту призупиненого оператора не визначені. `403` у `03-api-contract.md` — лише запропонований placeholder, і канонічне джерело (05) не вважає його узгодженим. Тому в кейсах нижче код і тіло **не** вказуються. Перевіряються тільки однозначно визначені інваріанти:
> - SUSPENDED — це передумова для write-операцій **create / submit / approve**, а не гейт (`05` §1, `07` п. 3);
> - відповідь **не** має бути `422` з `failing_gate` (`05` §1, `07` п. 3).

| ID | Пріор. | Назва | Передумови | Кроки | Очікуваний результат | Джерело | Статус |
|---|---|---|---|---|---|---|---|
| TC-SUS-01 | P1 | Створення заявки призупиненим оператором | OP-S `SUSPENDED`, залогінений; валідні PIL-S/AC-S | `POST /flight-requests` з валідним тілом | Заявку не створено (у `GET /flight-requests` її немає). Відповідь **не** `422` і не містить `failing_gate`. Код і тіло визначити неможливо | OQ §1; CL п. 3 | **Blocked** (OQ рядок 4) |
| TC-SUS-02 | P1 | Submit заявки призупиненим оператором | FR-S-DRAFT у `DRAFT` | `POST /flight-requests/{FR-S-DRAFT}/submit` | Стан лишається `DRAFT`; гейти не запускались (немає `failing_gate`/`rejection_reason`, заявка не в `REJECTED`). Відповідь не `422`. Код і тіло не визначені | OQ §1; CL п. 3; SM | **Blocked** (OQ рядок 4) |
| TC-SUS-03 | P1 | Submit заявки, яка точно не пройде гейти | FR-S-DRAFT з пілотом `R_BASIC` у зоні `LVV-RST-2` | `POST .../submit` | Відповідь **не** `422` з `failing_gate = PILOT`: SUSPENDED — окрема передумова, а не частина пайплайна гейтів, і відповідь не можна підганяти під форму `422`. Стан не `REJECTED` | OQ §1; API «Gate pipeline» (останній абзац); CL п. 3 | **Blocked** (код невідомий; порядок «передумова vs гейти» явно не описано, див. **Q-13**) |
| TC-SUS-04 | P1 | Ревʼюер схвалює заявку призупиненого оператора | FR-S-UR у `UNDER_REVIEW`; OP-S переведено в `SUSPENDED` після submit | `POST /flight-requests/{FR-S-UR}/approve` з `X-Reviewer-Id: REV` | Стан лишається `UNDER_REVIEW`, `ReviewAction` не створено. Відповідь не `422` з `failing_gate`. Код і тіло не визначені | OQ §1 (approve у переліку write-операцій); CL п. 3 | **Blocked** (OQ рядок 4) |
| TC-SUS-05 | P2 | Відмова (reject) заявки призупиненого оператора | FR-S-UR у `UNDER_REVIEW` | `POST .../reject` з `note` і `X-Reviewer-Id` | Docs називають тільки create / submit / approve. Чи блокується reject, невідомо | OQ §1; CL п. 3 | Needs clarification (**Q-14**) |
| TC-SUS-06 | P2 | Cancel призупиненим оператором | FR-S-DRAFT / FR-S-UR | `POST .../cancel` | Cancel не входить у перелік заблокованих операцій; поведінку не визначено | OQ §1 | Needs clarification (**Q-14**) |
| TC-SUS-07 | P2 | Реєстрація пілота або дрона призупиненим оператором | OP-S залогінений | `POST /pilots`, `POST /aircraft` | «Create» у docs сформульовано в контексті `POST /flight-requests` (07 п. 3). Чи стосується це `/pilots` і `/aircraft`, не визначено | CL п. 3; OQ §1 | Needs clarification (**Q-14**) |
| TC-SUS-08 | P2 | Validate (dry-run) призупиненим оператором | FR-S-DRAFT | `POST .../validate` | Validate не є write-операцією і не змінює стан. Чи блокується для SUSPENDED, не визначено. Якщо відповідає, то SUSPENDED не має зʼявлятися як результат якогось гейту | OQ §1; API-FR #13 | Needs clarification (**Q-14**) |
| TC-SUS-09 | P2 | Логін, `GET /auth/me`, читання власних даних призупиненим оператором | OP-S `SUSPENDED` | 1. `POST /auth/login`. 2. `GET /auth/me`. 3. `GET /pilots`, `GET /flight-requests` | Docs обмежують тільки write-операції; чи може SUSPENDED-оператор логінитися й читати, явно не сказано. Якщо `GET /auth/me` повертає профіль, `status = SUSPENDED` | DM «Operator.status»; OQ §1 | Needs clarification (**Q-14**) |
| TC-SUS-10 | P2 | UI: як призупинений оператор бачить блокування | OP-S залогінений в UI | Спробувати створити й засабмітити заявку через форму | Docs не описують повідомлення чи стан UI для SUSPENDED. Перевіряється тільки те, що заявку не створено і не засабмічено | FE (не описано); OQ §1 | **Blocked** (OQ рядок 4) |
| TC-SUS-11 | P1 | Регресія: блокування SUSPENDED не зачіпає інших операторів | OP-S `SUSPENDED`, OP-A `ACTIVE` | Під OP-A створити й засабмітити валідну заявку | OP-A працює як звичайно (заявка створюється, submit переводить її в `UNDER_REVIEW`) | SM; API-FR | Ready |
| TC-SUS-12 | P2 | Ізоляція діє і для SUSPENDED | OP-S залогінений | `GET /pilots/{PIL-B}`, `GET /flight-requests/{FR-B-DRAFT}` | `403`: правила ownership не залежать від статусу | API-AUTH «Enforcement» | Ready |

---

## 6. Відкриті питання та знахідки в документації

| # | Питання / знахідка | Де | Впливає на |
|---|---|---|---|
| Q-01 | Не вказано код успішної відповіді для `POST /operators` (`200`/`201`?) і `POST /auth/login`, а також форму тіла відповіді | `03` «Authentication & authorisation» | TC-REG-02, TC-AUTH-02 |
| Q-02 | Що робити з полями, які клієнт «не має права» передавати (`authorisation_level` у реєстрації, `operator_id` у тілі `POST /pilots`, `/aircraft`, `/flight-requests`): мовчки ігнорувати чи повертати `400`? | `03` «Enforcement», «authorisation_level is no longer self-service» | TC-REG-04, TC-ACC-20…22 |
| Q-03 | Статус-код і тіло для дубліката `email` / `national_id` при реєстрації (`400`? `409`? `422`?). `409` за контрактом означає стан або конфлікт розкладу, `422` — порушення гейту з `failing_gate`, тож жоден не підходить очевидно | `01` (unique), `03` «Error semantics» | TC-REG-07, TC-REG-08 |
| Q-04 | Атомарність реєстрації: якщо профіль `Operator` не створився (напр. дублікат `national_id`), чи відкочується створений Supabase Auth-користувач? | `03`, `04` («creating both together») | TC-REG-09 |
| Q-05 | Обовʼязкові поля профілю та правила валідації (формат email, вимоги до пароля, формат `phone`/`postal_code`/`national_id`) не описані | `01` «Operator», `04` | TC-REG-10 |
| Q-06 | Початкові значення `status` (імовірно `ACTIVE`) і `registered_on` при реєстрації явно не задані | `01` «Operator» | TC-REG-11 |
| Q-07 | Код відповіді `POST /auth/login` при хибних облікових даних (`401`?) не визначено | `03` | TC-AUTH-03, TC-AUTH-04 |
| Q-08 | Поведінка UI для незалогіненого користувача на захищених сторінках і для помилок `403`/`404` (редірект, повідомлення) не описана; `data-testid` для полів форм логіну й реєстрації не перелічені | `04` | TC-AUTH-09, TC-ACC-34 |
| Q-09 | Чи повертає `POST /flight-requests` саме `403`, якщо в тілі `pilot_id`/`aircraft_id` належать іншому операторові (а не `400`/`404`/`422`)? Формулювання «reaching for another operator's resource» це, імовірно, покриває, але явно не сказано | `03` «Enforcement» | TC-ACC-18 |
| Q-10 | Як у тестовому середовищі перевести оператора в `SUSPENDED`: API немає, а QA-кіту заборонено звертатися до Supabase. Потрібна процедура від власника стенду | `01`, `03` | Підготовка даних для 5.4 |
| Q-11 | **Суперечність:** розділ «Reviewer» каже `400`, якщо немає `X-Reviewer-Id`, а «Error semantics» визначає `401` як «no operator session and no `X-Reviewer-Id`». Що повертає approve/reject без жодної автентифікації? | `03` «Reviewer», «Error semantics» | TC-ACC-43 |
| Q-12 | Код для запиту тільки з `X-Reviewer-Id` (без сесії) на operator-only write-ендпоінти (`POST /pilots`, `/flight-requests`, `submit`, `cancel`): `401` чи `403`? | `03` | TC-ACC-44 |
| Q-13 | **Відкрите (OQ рядок 4):** статус-код і тіло для заблокованої операції SUSPENDED-оператора. Додатково не описано порядок: чи перевіряється передумова SUSPENDED до гейтів | `05` §1, `07` п. 3 | TC-SUS-01…04, 10 |
| Q-14 | Межі блокування SUSPENDED: docs називають тільки create / submit / approve. Не визначено: reject, cancel, `POST /pilots`, `POST /aircraft`, validate, логін і читання | `05` §1, `07` п. 3 | TC-SUS-05…09 |
| F-01 | **Знахідка:** `03-api-contract.md` («Error semantics») описує `403` для SUSPENDED як «pre-existing precondition», ніби це вирішено, а `05` §1 тримає питання відкритим і називає `403` лише placeholder. За правилами пріоритету канонічний статус береться з `05` | `03` vs `05` | 5.4 |

---

## 7. Ризики

| Ризик | Наслідок | Мітигація |
|---|---|---|
| Обхід авторизації через пряме API (UI ховає чуже, а сервер пропускає) | Витік або зміна даних інших операторів | Пріоритет P1 на API-кейси 5.3; обовʼязково перевіряти через HTTP-клієнт, не тільки через UI |
| SUSPENDED реалізовано «за аналогією» як `422` з вигаданим `failing_gate` | Порушення контракту, хибна діагностика для оператора | TC-SUS-01…04 явно перевіряють відсутність `422`/`failing_gate` |
| Немає способу підготувати SUSPENDED-оператора (Q-10) | Розділ 5.4 не можна виконати | Отримати процедуру від власника стенду до початку циклу |
| Неатомарна реєстрація (Q-04) | «Осиротілі» auth-акаунти, які блокують повторну реєстрацію з тим самим email | TC-REG-09 після відповіді на Q-04 |

---

## 8. Критерії входу та виходу

**Вхід:**
- Стенд доступний, каталог зон наповнений.
- Підготовлені OP-A, OP-B і їхні сутності (розділ 3.2).
- Для розділу 5.4 є процедура підготовки OP-S (Q-10). Без неї 5.4 виконується тільки в частині TC-SUS-11.

**Вихід:**
- Усі кейси зі статусом **Ready** виконані; P1 пройдені без відкритих дефектів критичної або високої важливості.
- Кожен кейс зі статусом **Blocked** або **Needs clarification** привʼязаний до питання в розділі 6, а питання передані власнику API-контракту чи бізнес-правил.
- Після відповідей на Q-xx кейси оновлено: додано конкретні коди й тіла та змінено статус на Ready.

---

## 9. Зведення

| Розділ | Всього | Ready | Needs clarification | Blocked |
|---|---|---|---|---|
| 5.1 Реєстрація | 12 | 7 | 5 | 0 |
| 5.2 Логін / логаут | 11 | 8 | 3 | 0 |
| 5.3 Ізоляція даних | 31 | 27 | 4 | 0 |
| 5.4 SUSPENDED | 12 | 2 | 5 | 5 |
| **Разом** | **66** | **44** | **17** | **5** |

> TC-REG-02, TC-AUTH-02, TC-REG-10 і TC-ACC-20…22 пораховано як Ready: їхній основний інваріант визначений, а уточнити треба тільки код відповіді (Q-01, Q-02, Q-05).
