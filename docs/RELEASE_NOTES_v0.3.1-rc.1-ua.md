# Smart Autopilot — v0.3.1-rc.1

## Нова функція: Allow Existing Data on Overpriced Contracts

Smart Autopilot може зберігати наявні дані на контрактах, ціна яких перевищує налаштовані обмеження, замість перенесення даних лише через перевищення ціни.

Налаштування:
- Backend: `AllowExistingDataOnOverpricedContracts`
- API JSON: `allowExistingDataOnOverpricedContracts`
- Значення за замовчуванням: `false`

Після увімкнення наявні дані можуть залишатися на overpriced/Gouging-контракті; вони залишаються доступними для читання за звичайних обмежень завантаження. Нові завантаження залишаються забороненими. Gouging-хости не вибираються для нових замін, а сам факт Gouging не запускає примусову міграцію наявних даних. Незалежні механізми repair/migration не змінюються, `MaxDownloadPrice` не послаблюється.

Під час Smart Gouging replacement придатний новий контракт усе одно може бути створений, але старий incumbent не переводиться в drain лише через ціну, доки налаштування увімкнене.

Після успішної заміни старий incumbent переходить у persistent legacy/replaced state: paused/read-only, виключається з Good upload pool і не замінюється повторно, доки зберігається Gouging.

Якщо налаштування вимкнути, legacy overpriced-контракти під час Smart Active maintenance починають drain, доки їхній хост залишається Gouging. У Shadow лише фіксується запланована дія; Original не виконує цю Smart-мутацію.

---

## Посилення надійності

### 1. Атомарна Smart-заміна контракту

Коміт `602fae82` — `Harden Smart contract replacement atomically`

Після успішного RHP formation новий контракт B, фіналізація incumbent A та зв’язок A→B зберігаються в одній транзакції БД.

Це виключає часткові стани, коли B уже існує, але A не фіналізований і replacement-зв’язок не збережений.

У разі помилки БД транзакція відкочується, Smart повідомляє `partial_failure`, а fallback-запис B поза транзакцією не виконується.

Помилка persistence блокує додаткові Smart replacements у поточному maintenance pass; наступний pass може повторити спробу.

Економічна та Gouging-заміни використовують спільний атомарний механізм.

### 2. Збереження точної причини Smart drain під час архівації

Коміт `18ee5027` — `Preserve Smart drain reasons through archival`

Причини Smart policy тепер зберігаються в persistent drain state і явно перетворюються на причину архівації:

| Smart policy | Persistent drain | Архівація |
|---|---|---|
| `gouging` після `MaxDowntimeHours` | `gougingTimeout` | `smartgouging` |
| `expiration_safety` | `expirationSafety` | `smartexpiration` |
| `overpriced_data_not_allowed` | `overpricedDataNotAllowed` | `smartoverpriced` |
| Economic/Gouging replacement | `replacement` | `smartreplacement` |

Міграція БД не потрібна: використовується наявне `smart_contract_state.reason`.

`FinalizeSmartDrain` тепер працює fail-closed для невідомої причини замість мовчазного призначення `smartreplacement`.

Вже запущені persistent drains продовжуються під час зміни Smart mode. Історичні `gougingTimeout` не backfill-яться, оскільки початкову причину неможливо надійно відновити.

---

## Пост-RC1 виправлення: Gouging recovery та occupancy замінених контрактів

### Коміт `25318c8c` — `fix smart gouging recovery and replaced occupancy`

Виправлено lifecycle контракту, який був замінений через Gouging, але його початковий хост згодом відновився.

Цільовий lifecycle:

```text
Good
  ↓
Gouging
  ↓
Paused / replacement
  ↓
Legacy / replaced
  ↓
хост відновився
  ↓
Good + Replaced=true
```

Ключова семантика:

- коли хост перестає бути Gouging, **будь-яка Gouging pause, включно з legacy/replaced state, повертається в `Good`**;
- `Replaced=true` зберігається як історична ознака;
- дані **не мігрують назад** із replacement-контракту;
- дані, що залишилися на відновленому контракті, залишаються на ньому;
- відновлений контракт знову є Good і може займати слот Smart portfolio;
- хост, що відновився, пізніше може отримати **новий** контракт, якщо Smart незалежно вибере його;
- зворотна міграція лише через відновлення початкового хоста не виконується;
- доки хост залишається Gouging, існуючий legacy/replacement lifecycle не змінюється.

### Occupancy

`Replaced=true` сам по собі не робить контракт неіснуючим для portfolio accounting.

| Стан | `Replaced` | Займає слот |
|---|---:|---:|
| Good | false | так |
| Good | true | так |
| Paused | false | так |
| Paused | true | ні |
| Bad | false | ні |
| Bad | true | ні |

Таким чином, відновлений `Replaced+Good` знову враховується в портфелі, а `Replaced+Paused`, який drain-иться, слот не займає.

---

## Жива перевірка recovery fix

Виправлення протестовано на працюючому mainnet renterd.

Два контракти, які раніше пройшли:

```text
Gouging → Pause → Replacement → Legacy
```

після запуску нового бінарника отримали:

```text
smart pause
contractID: e9421132765cd59bfecb3f3f1b28ca34f987f66e13bcc46eccc5ea3c5e3425ec
decision: recover
why: host no longer gouging
outcome: success
legacy: true
pausedAt: 2026-10-05T14:55:42+03:00
```

і:

```text
smart pause
contractID: 70887117eebd7bfdc9b5a2f2307a51f1a448a43cf6a62d74b31b57dc1bb91d11
decision: recover
why: host no longer gouging
outcome: success
legacy: true
pausedAt: 2026-10-05T14:55:42+03:00
```

Звичайний contract-maintenance pass після цього повідомив, що обидва контракти знову usable, і оновив два контракти.

Smart portfolio повідомив:

```text
wantedContracts: 10
occupancy: 12
plannedOccupancy: 12
```

що підтверджує участь відновлених `Replaced+Good` контрактів в occupancy.

Зворотна міграція для них не запускалася.

У тому самому maintenance pass Smart незалежно виконав звичайну економічну заміну іншого incumbent, що підтверджує відсутність конфлікту recovery зі звичайною оптимізацією.

---

## Перевірка та тести

Повний Autopilot test suite:

```powershell
go test ./autopilot/... -count=1 -v
```

Пройшли тести для:

- Gouging legacy lifecycle;
- Gouging legacy setting ON/OFF;
- Gouging replacement;
- Gouging pause recovery;
- Smart economic pause;
- persistent replacement state;
- Gouging legacy portfolio;
- replaced-contract occupancy;
- Smart planning/execution;
- contractor;
- migrator;
- pruner;
- scanner;
- Smart packages.

Фінальну збірку виконано з `25318c8c`.

```text
renterd v0.3.1-rc.1-1-g25318c8c
Network mainnet
Commit: 25318c8c
Build Date: 2026-10-05 19:31:34 +0300 EEST
```

Windows amd64:

```text
renterd.exe
Size: 61,758,577 bytes
SHA256: 9CC3D68BDECB5B8AC885464F21F841A4832887A1BE58D9404CC4B26188CC30EB
```

Встановлений бінарник перевірено за SHA256 і він збігається зі збіркою.

Попередній встановлений бінарник було збережено перед заміною:

```text
Size: 61,760,113 bytes
SHA256: A2601E64D20A5CB4518B8298B7FF3D44FBA1CF197768487932919B0941708442
```

---

## Поточна Git-база

- `602fae82` — `Harden Smart contract replacement atomically`
- `18ee5027` — `Preserve Smart drain reasons through archival`
- `25318c8c` — `fix smart gouging recovery and replaced occupancy`

Після останнього fix-коміту робоче дерево було чистим.

Тег `v0.3.1-rc.1` створено до `25318c8c`, тому фінальний бінарник ідентифікується як:

```text
v0.3.1-rc.1-1-g25318c8c
```

Це зберігає точне Git-походження протестованого бінарника.

---

## Функціональне резюме

Smart Autopilot — окремий policy layer поверх існуючих механізмів renterd для контрактів, migration, repair і передачі даних. Рішення приймаються за окремими Smart scoring dimensions, а не за одним непрозорим host score.

Поточна функціональність:

- Smart Active / Shadow / Original;
- Smart Overall scoring;
- вибір хостів і балансування портфеля;
- ADD / KEEP / PAUSE / REPLACE / EXIT;
- економічна заміна за порівнянням вартості міграції із залишковою вартістю контракту;
- persistent Smart draining;
- Gouging replacement;
- Gouging pause/recovery lifecycle;
- legacy/replaced lifecycle;
- участь відновлених `Replaced+Good` у portfolio occupancy;
- налаштування збереження наявних даних на overpriced-контрактах;
- явні причини Smart drain та archival;
- атомарне збереження replacement і зв’язок A→B;
- відсутність автоматичної зворотної міграції після відновлення заміненого хоста.

Smart layer залишається налаштовуваним і використовує наявні низькорівневі механізми renterd для formation, renewal, repair, migration, upload і download.
