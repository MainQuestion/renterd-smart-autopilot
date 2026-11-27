# renterd Smart Autopilot

**Мови:** [English](README.md) · Українська · [Русский](README_RU.md)

Неофіційне експериментальне розширення Sia `renterd`, призначене для прозорого оцінювання хостів, керування портфелем контрактів і розумнішого розміщення нових даних.

Якщо в хоста виникає проблема (завищена ціна, недоступність, нестача застави), стандартна логіка виводить контракт і переносить його дані на інші хости, а перенесення даних коштує грошей. Smart Autopilot ставить такий контракт **на паузу** і переносить дані лише тоді, коли заміна економічно виправдана (докладніше в розділі «Головна ідея» нижче).

> **Поточний випуск:** `v0.3.3-rc.1` — Release Candidate з обліком витрат за контрактами (Σ storage, Σ ingress, Σ egress, Σ spent), переробленою оцінкою Economics, паузою невдалих кандидатів, виправленням помилки Merkle-proof після продовження контракту та оновленою таблицею Active contracts.

## Головна ідея: пауза замість міграції

**Контракт, у якого з'явилася проблема, не потрібно одразу виводити й переносити. Smart Autopilot ставить його на паузу й переносить дані, лише коли це справді виправдано.**

### Проблема

У стандартній логіці хост, що вийшов за допустимі ціни, став недоступним або не проходить перевірки, веде до виведення контракту та перенесення його даних на інші хости. Перенесення коштує грошей: дані потрібно прочитати з решти хостів, відновити, записати заново й оплатити передавання та зберігання за цінами нових хостів. Стрибок цін або короткий збій можуть запустити такі перенесення відразу на кількох хостах, і витрата стає непропорційною проблемі, з якої все почалося.

### Що робить пауза

- Контракт на паузі **не отримує нових даних**, а дані, що вже лежать на хості, **залишаються на місці** і можуть залишатися доступними для читання: перенесення не запускається лише через завищену ціну чи тимчасовий збій.
- Паузи **вважаються здоровою надмірністю** (Paused placements), доки даних достатньо для відновлення; якщо доступних шардів лишається `K` або менше, запускається safety repair.
- Заміна виконується **лише коли вона виправдана**: Smart Portfolio порівнює **KeepCost** і **MigrationCost** до перенесення даних. Якщо заміна не виправдана, контракт отримує відновлювану **Economic Pause**.
- Заміна безпечна: **спочатку створюється новий контракт, і лише потім старий виводиться** через persistent Smart Drain; безпеку даних перевіряють до архівації.
- Якщо проблема зникла, контракт повертається в Good, і перенесення не знадобилося.

### Як це виглядає

```text
Good
  → Paused (ціна / економіка)
      → проблема зникла                        → знову Good, перенесення не знадобилося
      → з'явився eligible replacement          → новий контракт → потім drain старого
      → заміна не виправдана                   → лишається на Economic Pause
      → hard threshold / timeout               → Bad → ADD (формується новий контракт)
```

Переходи `Good → Paused → Good` і `Paused → Bad → ADD` перевірено практично (див. «Перевірка»).

### Що налаштовується

- Режим Smart (Original / Smart Shadow / Smart Active): паузи та заміни виконує **Smart Active**; Smart Shadow лише пише в лог, що зробив би автопілот.
- **Keep Gouging Contract Until Replacement**: чи може контракт на паузі чекати заміну, чи має бути виведений після safety boundary.
- **Allow Existing Data on Overpriced Contracts** (за замовчуванням вимкнено): зберігати вже розміщені дані на контракті із завищеною ціною замість перенесення.
- **Smart Replacement Threshold** (за замовчуванням `15%`) на сторінці **Scoring**.

Де дивитися результат: `smart-autopilot.log` (economic pause/recovery, Gouging lifecycle, safety repair), сторінка **Scoring**. Докладний опис: «Паузи контрактів» нижче.

## Що змінилося

Оригінальні механізми renterd залишаються низькорівневим виконавчим шаром: формування контрактів, renew/refresh, upload/download, міграція та архівація.

Smart Autopilot додає над ними policy layer:

```text
виміри та історія хостів
        ↓
багатовимірний профіль хоста
        ↓
Smart Overall + policy gates
        ↓
PortfolioPlan
        ↓
ADD / PAUSE / RECOVER / REPLACE
        ↓
наявні механізми renterd
```

Мета — не замінити renterd одним новим «магічним score», а зберегти різні властивості хоста окремими та використовувати відповідні критерії для кожного рішення.

## Що нового у v0.3.3-rc.1

- **Облік витрат за контрактами.** У БД з'явилися окремі витрати на зберігання (storage), приймання даних (ingress) і читання (egress). У таблиці Active contracts додано колонки **Σ storage, Σ ingress, Σ egress, Σ spent**, а також My Data, Capacity, Initial/Remaining Allowance, Σ collateral і Used Collateral. Суми рахуються за ланцюгом продовжень. Σ egress — нижня межа: скасовані та невдалі читання renterd не бачить.
- **Economics перероблено.** Collateral коригує ефективну ціну зберігання, Economics Score — це Cost Score скоригованої вартості. Якорі вартості та collateral беруться лише за придатними хостами; хости з нульовою вартістю не спотворюють якір.
- **Пауза кандидата Smart.** Кандидат, у якого формування контракту провалилося з вини хоста, виключається з плану на 24 години (за повторних невдач до 7 діб).
- **Виправлення Merkle-proof.** Виправлено помилку (дефект upstream), через яку після продовження контракту його розсилання ревізій відхилялося (`file contract revision parent has invalid Merkle proof`). Уже зіпсовані елементи лагодяться один раз під час запуску.
- **Швидша історія цін.** Новий індекс `price_history` пришвидшує запит історії цін приблизно в 35 разів.
- **UI.** Вибір колонок запам'ятовується в Hosts і Active contracts, згруповані колонки контрактів, кольори, сортування, підсумки під заголовками, прибрано підказки, що повторюють значення.

Докладніше — в release notes випуску. Три міграції БД (`00067`, `00068`, `00069`) виконуються автоматично й лише додають колонки та індекс.

## Реалізований профіль хоста

У RC1 повністю завершено чотири із семи запланованих груп:

- **G2 Performance** — фактична швидкість upload/download з накопиченої історії.
- **G3 Resources** — експозиція renter до хоста: наші дані, вільне місце, чужі дані, фінансова експозиція, вартість recovery/migration, залишок строку контракту та частка втрачених секторів.
- **G4 Economics** — вартість storage/ingress/egress з поправкою на якість collateral: застава коригує ефективну ціну зберігання, а підсумковий Economics Score — це Cost Score скоригованої вартості (якір p10 за придатними хостами).
- **G5 Risk Coverage** — історична стабільність цін storage/ingress/egress і collateral.

Ще заплановано як повноцінні окремі scoring-групи:

- **G1 Reliability**
- **G6 Technical Compatibility**
- **G7 Network Diversity**

Частина перевірок compatibility/diversity уже використовується як eligibility/policy constraints, але поки це не завершені групи 1–10.

## Smart Overall

Чотири завершені групи об'єднуються з користувацькими вагами:

```text
SmartOverall =
    G2 × W2 / 100 +
    G3 × W3 / 100 +
    G4 × W4 / 100 +
    G5 × W5 / 100
```

Persisted defaults: `25 / 25 / 25 / 25`.

Smart Overall навмисно не є єдиним критерієм для всіх рішень.

## Три режими

### Original

Використовує звичайну policy формування й обслуговування контрактів renterd. Smart Portfolio decisions і Smart upload ordering не застосовуються.

Уже запущені persistent Smart safety lifecycles (зокрема паузи) безпечно завершуються.

### Smart Shadow

Розраховує ті самі Smart-рішення, що й Active, і пише їх у `smart-autopilot.log`, але не виконує нових Smart portfolio mutations (зокрема не починає нових пауз).

Для нових upload Shadow залишає оригінальний порядок `Uploader.Estimate()` і логує його поруч зі Smart order.

### Smart Active

Виконує Smart Portfolio decisions (включно з паузами та замінами) і використовує Smart Placement для нових користувацьких upload.

Кандидати сортуються за `SmartOverall DESC` усередині наявного Good/Usable upload pool.

## Паузи контрактів

Паузи — основний механізм, що запобігає зайвим перенесенням даних (див. «Головна ідея»). Нові паузи починає Smart Active.

### Стан Paused

Додано повноцінний стан `Paused` між Good і Bad:

- нові upload не приймаються;
- наявні дані можуть залишатися readable;
- Paused placements вважаються healthy redundancy;
- recovery може повернути контракт у Good;
- hard threshold/timeout переводить його в Bad.

### Цінова пауза (Gouging lifecycle)

Smart Active додає окремий безпечний lifecycle для наявного контракту, якщо єдина проблема хоста — Gouging:

```text
Good
→ Gouging Paused
→ recover / replace / timeout drain
```

Gouging-Paused контракт:

- не отримує нових upload;
- може залишатися readable, якщо download price допустима;
- зберігає persistent Smart state;
- може бути замінений одразу після появи eligible replacement;
- може запускати safety repair, коли доступних shards стає `K` або менше.

Налаштування **Keep Gouging Contract Until Replacement** визначає, чи може такий контракт і далі очікувати/renew replacement, чи має бути drained після safety boundary.

#### Allow Existing Data on Overpriced Contracts

Налаштування **Allow Existing Data on Overpriced Contracts** дозволяє Smart Active зберігати вже розміщені дані на Gouging-контракті замість перенесення даних лише через завищену ціну.

- Налаштування знаходиться на сторінці **Scoring** і **за замовчуванням вимкнене**, тому попередня поведінка зберігається.
- За ввімкненого налаштування Smart як і раніше може сформувати replacement, але старий контракт не переводиться в drain лише тому, що replacement уже створено.
- Після успішного Gouging replacement старий контракт стає **legacy**-контрактом лише для читання, доки його хост залишається Gouging: нові upload туди не йдуть, renew/refresh для збереження Gouging-контракту не виконуються, слот Smart Portfolio він не займає.
- Наявні дані можуть залишатися доступними через чинні read/download-перевірки. Якщо хост перестає бути Gouging, legacy-контракт може відновитися через звичайний Gouging recovery lifecycle.
- Якщо налаштування знову вимкнути, **Smart Active** може запустити drain збережених legacy-даних.
- Захист від спливання контракту не залежить від цінової політики: для Gouging/legacy-контракту, який більше не продовжуватиметься, під час входу в другу половину renew window запускається наявний expiration-safety drain.
- Gouging safety repair не змушує переносити дані, доки політика дозволяє їх зберігати; у відповідному випадку рішення фіксується як `allowed_existing_data`.

### Економічна пауза (Economic Pause)

Smart Portfolio порівнює **KeepCost** і **MigrationCost** до перенесення даних. Якщо заміна економічно не виправдана, контракт ставиться на відновлювану **Economic Pause** замість перенесення. Докладніше про правила заміни: «Smart Portfolio».

## Smart Portfolio

Smart Portfolio вміє:

- заповнювати відсутні contract slots через **Smart ADD**;
- знаходити найслабший поточний Good incumbent за Smart Overall;
- застосовувати налаштовуваний **G4 Economics improvement threshold**;
- порівнювати **KeepCost** і **MigrationCost** до перенесення даних;
- ставити контракт на відновлювану **Economic Pause**, якщо заміна економічно не виправдана (див. «Паузи контрактів»);
- виконувати безпечний **REPLACE**, спочатку створюючи новий контракт;
- виводити старий контракт через persistent **Smart Drain**;
- перевіряти безпеку даних до архівації;
- тимчасово виключати кандидата, у якого формування провалилося з вини хоста (24 години, за повторних невдач до 7 діб; зберігається в пам'яті та скидається під час перезапуску). Це не пауза контракту, а виключення хоста з кандидатів на нові контракти.

Default Smart Replacement Threshold: `15%`.

Звичайний optimization REPLACE має 4-годинний cadence. ADD ним не блокується.

## Історія та діагностика

Smart Autopilot використовує історію:

- upload/download speed;
- prices і collateral;
- scan results.

Окремий:

```text
smart-autopilot.log
```

фіксує ADD, REPLACE, economic pause/recovery, Gouging lifecycle, safety repair і порівняння Original/Smart upload order.

## UI

Додано:

- окрему сторінку **Scoring**;
- налаштовувані legacy host-score параметри;
- ваги Smart groups;
- Smart Replacement Threshold;
- Keep Gouging Contract Until Replacement;
- Allow Existing Data on Overpriced Contracts;
- згруповані host-profile колонки;
- raw upload/download Mbps;
- Smart Overall;
- покращене сортування хостів;
- налаштування відображення таблиць і чисел;
- таблицю **Active contracts** із групами колонок (General, Time, Contract data, Financial, Financial original), колонками Σ spent, Σ storage, Σ ingress, Σ egress, My Data, Capacity, Allowance і collateral, сортуванням, підсумками під заголовками та збереженням вибору колонок.

## Перевірка

Перед публікацією v0.3.3-rc.1 пройдено build/vet/test (включно з e2e та `-race` за зачепленими пакетами), тест виправлення Merkle-proof на реальному ланцюгу, а також практичне тестування на робочій установці: міграції `00067`–`00069`, одноразове виправлення 8 контрактів (`invalid: 8` → `repaired: 8`, на наступному старті `invalid: 0`), успішне розсилання ревізії, перевірка таблиці контрактів у браузері. Релізну збірку зроблено з тих самих комітів, що й робочу на робочій установці.

Раніше, перед публікацією RC1, пройшли перевірки Smart-шляхів:

```text
4-hour optimization REPLACE cadence
ADD до спливання cadence за випадіння контракту
Good → Paused → Good
Paused → Bad → ADD
```

## Build information

```text
Release:        v0.3.3-rc.1
renterd:        v0.3.3-rc.1
Backend commit: 3738c027
Web commit:     83a2dde8
Network:        mainnet
```

```text
renterd.exe
Size: 62,078,153 bytes
SHA256: C9FB10B32D08BBD12150C2C34ED1C38092B472B55BCC2BD7A797E26423B94F17
```

```text
renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip
Size: 22011895 bytes
SHA256: E0AB8B28DD4A2701535D02E4F177D1E29BE4B83EE7DD6397998C58CC049D8B68
```

## Встановлення

1. Зробіть резервну копію даних і конфігурації renterd (для SQLite — `.backup` БД і `metrics.sqlite`).
2. Завантажте `renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip` з GitHub Release.
3. Розпакуйте `renterd.exe` і `LICENSE`.
4. Зупиніть renterd і замініть executable.
5. Для першого запуску використовуйте **Original** або **Smart Shadow**, а потім за потреби переходьте в **Smart Active**.

Міграції бази виконуються стандартним механізмом renterd. Під час першого запуску виконаються міграції `00067`–`00069` (створення індексу займає секунди), а в лозі з'являться рядки `contract elements checked` і, за потреби, `contract elements repaired`. Відкат: поверніть попередній `renterd.exe`; старий бінарник стартує на БД із новими міграціями.

## Докладне порівняння

Див. [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md).

## Випуск

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.3-rc.1

## Ліцензія та upstream

Це неофіційна експериментальна похідна збірка на основі Sia Foundation renterd.

Див. `LICENSE`.

Це не офіційний випуск Sia Foundation. RC-версії призначені для тестування; зберігайте резервні копії та контролюйте логи/alerts.
