# renterd Smart Autopilot

**Мови:** [English](README.md) · Українська · [Русский](README_RU.md)

Неофіційне експериментальне розширення Sia `renterd`, призначене для
прозорої оцінки хостів, керування портфелем контрактів і розумнішого
розміщення нових даних.

> **Поточний реліз:** `v0.3.3-rc.1` --- Release Candidate з обліком
> витрат за контрактами (Σ storage, Σ ingress, Σ egress, Σ spent),
> переробленою оцінкою Economics, паузою невдалих кандидатів,
> виправленням помилки Merkle-proof після продовження контракту та
> оновленою таблицею Active contracts.

## Що змінилося

Оригінальні механізми renterd залишаються низькорівневим виконавчим
шаром: формування контрактів, renew/refresh, upload/download, міграція
та архівація.

Smart Autopilot додає над ними policy layer:

``` text
вимірювання та історія хостів
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

Мета --- не замінити renterd одним новим «магічним score», а зберегти
різні властивості хоста окремими й використовувати відповідні критерії
для кожного рішення.

## Що нового у v0.3.3-rc.1

-   **Облік витрат за контрактами.** У БД з'явилися окремі витрати на
    зберігання (storage), приймання даних (ingress) і читання (egress).
    У таблиці Active contracts додано колонки **Σ storage, Σ ingress,
    Σ egress, Σ spent**, а також My Data, Capacity, Initial/Remaining
    Allowance, Σ collateral і Used Collateral. Суми рахуються за
    ланцюгом продовжень. Σ egress --- нижня межа: скасовані та невдалі
    читання renterd не бачить.
-   **Economics перероблено.** Застава коригує ефективну ціну
    зберігання, Economics Score --- це Cost Score скоригованої вартості.
    Якорі вартості й застави беруться лише з придатних хостів; хости з
    нульовою вартістю не спотворюють якір.
-   **Пауза кандидата Smart.** Кандидат, у якого формування контракту
    провалилося з вини хоста, виключається з плану на 24 години (при
    повторних невдачах до 7 діб).
-   **Виправлення Merkle-proof.** Виправлено помилку (дефект upstream),
    через яку після продовження контракту розсилання його ревізій
    відхилялося (`file contract revision parent has invalid Merkle
    proof`). Уже зіпсовані елементи лагодяться один раз під час запуску.
-   **Швидша історія цін.** Новий індекс `price_history` прискорює запит
    історії цін приблизно в 35 разів.
-   **UI.** Вибір колонок запам'ятовується в Hosts і Active contracts,
    згруповані колонки контрактів, кольори, сортування, підсумки під
    заголовками, прибрано підказки, що повторюють значення.

Докладніше --- у release notes релізу. Три міграції БД (`00067`,
`00068`, `00069`) виконуються автоматично й лише додають колонки та
індекс.

## Реалізований профіль хоста

У RC1 повністю завершено чотири із семи запланованих груп:

-   **G2 Performance** --- фактична швидкість upload/download із
    накопиченої історії.
-   **G3 Resources** --- експозиція renter до хоста: наші дані, вільне
    місце, чужі дані, фінансова експозиція, вартість recovery/migration,
    залишок строку контракту та частка втрачених секторів.
-   **G4 Economics** --- вартість storage/ingress/egress із поправкою на
    якість collateral: застава коригує ефективну ціну зберігання, а
    підсумковий Economics Score --- це Cost Score скоригованої вартості
    (якір p10 за придатними хостами).
-   **G5 Risk Coverage** --- історична стабільність цін
    storage/ingress/egress та collateral.

Ще заплановано як повноцінні окремі scoring-групи:

-   **G1 Reliability**
-   **G6 Technical Compatibility**
-   **G7 Network Diversity**

Частину перевірок compatibility/diversity уже використано як
eligibility/policy constraints, але поки це не завершені групи 1--10.

## Smart Overall

Чотири завершені групи об'єднуються з користувацькими вагами:

``` text
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

Використовує звичайну policy формування та обслуговування контрактів
renterd. Smart Portfolio decisions і Smart upload ordering не
застосовуються.

Уже запущені persistent Smart safety lifecycles безпечно завершуються.

### Smart Shadow

Розраховує ті самі Smart-рішення, що й Active, і пише їх у
`smart-autopilot.log`, але не виконує нових Smart portfolio mutations.

Для нових upload Shadow залишає оригінальний порядок
`Uploader.Estimate()` і логує його поруч зі Smart order.

### Smart Active

Виконує Smart Portfolio decisions і використовує Smart Placement для
нових користувацьких upload.

Кандидати сортуються за `SmartOverall DESC` усередині наявного
Good/Usable upload pool.

## Smart Portfolio

Smart Portfolio вміє:

-   заповнювати відсутні contract slots через **Smart ADD**;
-   знаходити найслабшого поточного Good incumbent за Smart Overall;
-   застосовувати налаштовуваний **G4 Economics improvement threshold**;
-   порівнювати **KeepCost** і **MigrationCost** до перенесення даних;
-   ставити контракт у відновлюваний **Economic Pause**, якщо заміна
    економічно не виправдана;
-   виконувати безпечний **REPLACE**, спершу створюючи новий контракт;
-   виводити старий контракт через persistent **Smart Drain**;
-   перевіряти безпеку даних до архівації;
-   тимчасово виключати кандидата, у якого формування провалилося з
    вини хоста (24 години, при повторних невдачах до 7 діб; зберігається
    в пам'яті та скидається при перезапуску).

Default Smart Replacement Threshold: `15%`.

Звичайний optimization REPLACE має 4-годинний cadence. ADD ним не
блокується.

## Gouging lifecycle

Smart Active додає окремий безпечний lifecycle для наявного контракту,
якщо єдина проблема хоста --- Gouging:

``` text
Good
→ Gouging Paused
→ recover / replace / timeout drain
```

Gouging-Paused контракт:

-   не отримує нових upload;
-   може залишатися readable, якщо download price припустима;
-   зберігає persistent Smart state;
-   може бути замінений одразу після появи eligible replacement;
-   може запускати safety repair, коли доступних shards стає `K` або
    менше.

Налаштування **Keep Gouging Contract Until Replacement** визначає, чи
може такий контракт продовжувати очікувати/renew replacement, чи має
бути drained після safety boundary.

### Allow Existing Data on Overpriced Contracts

Налаштування **Allow Existing Data on Overpriced Contracts** дозволяє
Smart Active зберігати вже розміщені дані на Gouging-контракті замість
перенесення даних лише через завищену ціну.

-   Налаштування знаходиться на сторінці **Scoring** і **за
    замовчуванням вимкнене**, тож попередня поведінка зберігається.
-   При ввімкненому налаштуванні Smart, як і раніше, може сформувати
    replacement, але старий контракт не переводиться в drain лише тому,
    що replacement уже створено.
-   Після успішного Gouging replacement старий контракт стає
    **legacy**-контрактом лише для читання, поки його хост залишається
    Gouging: нові upload туди не йдуть, renew/refresh для збереження
    Gouging-контракту не виконуються, слот Smart Portfolio він не
    займає.
-   Наявні дані можуть залишатися доступними через чинні
    read/download-перевірки. Якщо хост перестає бути Gouging,
    legacy-контракт може відновитися через звичайний Gouging recovery
    lifecycle.
-   Якщо налаштування знову вимкнути, **Smart Active** може запустити
    drain збережених legacy-даних.
-   Захист від спливання контракту не залежить від цінової політики: для
    Gouging/legacy-контракту, який більше не продовжуватиметься, при
    вході в другу половину renew window запускається наявний
    expiration-safety drain.
-   Gouging safety repair не змушує переносити дані, поки політика
    дозволяє їх зберігати; у відповідному випадку рішення фіксується як
    `allowed_existing_data`.

## Paused

У проєкті також додано повноцінний стан `Paused` між Good і Bad:

-   нові upload не приймаються;
-   наявні дані можуть залишатися readable;
-   Paused placements вважаються healthy redundancy;
-   recovery може повернути контракт у Good;
-   hard threshold/timeout переводить його в Bad.

## Історія та діагностика

Smart Autopilot використовує історію:

-   upload/download speed;
-   prices і collateral;
-   scan results.

Окремий:

``` text
smart-autopilot.log
```

фіксує ADD, REPLACE, economic pause/recovery, Gouging lifecycle, safety
repair і порівняння Original/Smart upload order.

## UI

Додано:

-   окрему сторінку **Scoring**;
-   налаштовувані legacy host-score параметри;
-   ваги Smart groups;
-   Smart Replacement Threshold;
-   Keep Gouging Contract Until Replacement;
-   Allow Existing Data on Overpriced Contracts;
-   згруповані host-profile колонки;
-   raw upload/download Mbps;
-   Smart Overall;
-   покращене сортування хостів;
-   налаштування відображення таблиць і чисел;
-   таблицю **Active contracts** із групами колонок (General, Time,
    Contract data, Financial, Financial original), колонками Σ spent,
    Σ storage, Σ ingress, Σ egress, My Data, Capacity, Allowance та
    collateral, сортуванням, підсумками під заголовками та збереженням
    вибору колонок.

## Перевірка

Перед публікацією v0.3.3-rc.1 пройшов build/vet/test (зокрема e2e та
`-race` за зачепленими пакетами), тест виправлення Merkle-proof на
реальному ланцюгу, а також практичне тестування на робочій
інсталяції: міграції `00067`--`00069`, одноразове виправлення 8
контрактів (`invalid: 8` → `repaired: 8`, на наступному старті
`invalid: 0`), успішне розсилання ревізії, перевірка таблиці контрактів
у браузері. Релізну збірку зібрано з тих самих комітів, що й
працюючу на робочій інсталяції.

Раніше, перед публікацією RC1, пройшли перевірки Smart-шляхів:

``` text
4-hour optimization REPLACE cadence
ADD до спливання cadence при випадінні контракту
Good → Paused → Good
Paused → Bad → ADD
```

## Build information

``` text
Release:        v0.3.3-rc.1
renterd:        v0.3.3-rc.1
Backend commit: 3738c027
Web commit:     83a2dde8
Network:        mainnet
```

``` text
renterd.exe
Size: 62,078,153 bytes
SHA256: C9FB10B32D08BBD12150C2C34ED1C38092B472B55BCC2BD7A797E26423B94F17
```

``` text
renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip
Size: 22011895 bytes
SHA256: E0AB8B28DD4A2701535D02E4F177D1E29BE4B83EE7DD6397998C58CC049D8B68
```

## Встановлення

1.  Зробіть резервну копію даних і конфігурації renterd (для SQLite ---
    `.backup` БД і `metrics.sqlite`).
2.  Завантажте `renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip`
    з GitHub Release.
3.  Розпакуйте `renterd.exe` і `LICENSE`.
4.  Зупиніть renterd і замініть executable.
5.  Для першого запуску використовуйте **Original** або **Smart
    Shadow**, а потім за потреби переходьте в **Smart Active**.

Міграції бази виконуються стандартним механізмом renterd. Під час
першого запуску виконаються міграції `00067`--`00069` (створення
індексу займає секунди), а в логу з'являться рядки `contract elements
checked` і, за потреби, `contract elements repaired`. Відкат: поверніть
попередній `renterd.exe`; старий бінарник стартує на БД із новими
міграціями.

## Докладне порівняння

Див. [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md).

## Реліз

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.3-rc.1

## Ліцензія та upstream

Це неофіційна експериментальна похідна збірка на основі Sia Foundation
renterd.

Див. `LICENSE`.

Це не офіційний реліз Sia Foundation. RC-версії призначені для
тестування; зберігайте резервні копії та контролюйте логи/alerts.
