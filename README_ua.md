# renterd Smart Autopilot

**Мови:** [English](README.md) · Українська · [Русский](README_ru.md)

Неофіційне експериментальне розширення Sia `renterd`, орієнтоване на прозору оцінку хостів, керування портфелем контрактів і розумніше розміщення нових даних.

> **Поточний реліз:** `v0.3.0-rc.1` — перший протестований Release Candidate з активним Smart Portfolio decision/execution layer.

## Що змінилося

Оригінальні механізми renterd залишаються низькорівневим виконавчим шаром: формування контрактів, renew/refresh, upload/download, міграція та архівація.

Smart Autopilot додає над ними policy layer:

```text
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
існуючі механізми renterd
```

Мета — не замінити renterd одним новим «магічним score», а зберегти окремі властивості хоста видимими та застосовувати відповідні критерії для різних рішень.

## Реалізований профіль хоста

У RC1 повністю завершено чотири з семи запланованих груп:

- **G2 Performance** — реальна швидкість upload/download з історії.
- **G3 Resources** — експозиція renter до хоста: наші дані, вільне місце, чужі дані, фінансова експозиція, вартість recovery/migration, залишок строку контракту та частка втрачених секторів.
- **G4 Economics** — вартість storage/ingress/egress плюс якість collateral. Підсумкова Economics оцінка використовує гірше значення між Cost Score і Collateral Score.
- **G5 Risk Coverage** — історична стабільність цін storage/ingress/egress та collateral.

Ще заплановані як повні окремі scoring-групи:

- **G1 Reliability**
- **G6 Technical Compatibility**
- **G7 Network Diversity**

Частина compatibility/diversity перевірок уже працює як eligibility/policy constraints, але це ще не завершені окремі групи 1–10.

## Smart Overall

Чотири завершені групи можуть об'єднуватися з вагами користувача:

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

Використовує звичайну policy формування та обслуговування контрактів renterd. Smart Portfolio decisions і Smart upload ordering не застосовуються.

Вже розпочаті persistent Smart safety lifecycles безпечно завершуються.

### Smart Shadow

Обчислює ті самі Smart-рішення, що й Active, і записує їх у `smart-autopilot.log`, але не виконує нові Smart portfolio mutations.

Для нових upload Shadow залишає оригінальний порядок `Uploader.Estimate()` і лише порівнює його зі Smart order у логах.

### Smart Active

Виконує Smart Portfolio decisions і використовує Smart Placement для нових користувацьких upload.

Кандидати сортуються за `SmartOverall DESC` у межах існуючого Good/Usable upload pool.

## Smart Portfolio

RC1 уміє:

- заповнювати відсутні слоти через **Smart ADD**;
- визначати найслабший поточний Good incumbent через Smart Overall;
- вимагати налаштовуваний **G4 Economics improvement threshold**;
- порівнювати **KeepCost** і **MigrationCost** перед міграцією;
- ставити контракт у відновлюваний **Economic Pause**, якщо заміна економічно невигідна;
- виконувати безпечний **REPLACE**, спочатку створюючи новий контракт;
- виводити старий контракт через persistent **Smart Drain**;
- перевіряти безпеку даних перед архівацією.

Default Smart Replacement Threshold: `15%`.

Звичайний optimization REPLACE має 4-годинний cadence. ADD ним не блокується.

## Gouging lifecycle

Smart Active додає безпечніший lifecycle для вже існуючого контракту, якщо єдина проблема хоста — Gouging:

```text
Good
→ Gouging Paused
→ recover / replace / timeout drain
```

Gouging-Paused контракт:

- не отримує нових upload;
- може залишатися readable, якщо download price допустима;
- має persistent Smart state;
- може бути замінений одразу після появи eligible replacement;
- може запускати safety repair, якщо доступних shards стає `K` або менше.

Параметр **Keep Gouging Contract Until Replacement** визначає, чи може такий контракт продовжувати чекати/renew replacement, або має бути drained після safety boundary.

## Paused

Проєкт також додає повноцінний стан `Paused` між Good і Bad:

- нові upload не приймаються;
- старі дані можуть залишатися readable;
- Paused placements враховуються як healthy redundancy;
- recovery повертає контракт у Good;
- hard threshold/timeout переводить його у Bad.

## Історія та діагностика

Використовуються історичні дані:

- швидкість upload/download;
- ціни та collateral;
- результати scan.

Окремий:

```text
smart-autopilot.log
```

фіксує ADD, REPLACE, economic pause/recovery, Gouging lifecycle, safety repair і порівняння Original/Smart upload order.

## UI

Додано:

- окрему сторінку **Scoring**;
- налаштування legacy host-score;
- ваги Smart groups;
- Smart Replacement Threshold;
- Keep Gouging Contract Until Replacement;
- групи профілю у Hosts;
- raw upload/download Mbps;
- Smart Overall;
- покращене сортування хостів;
- налаштування відображення таблиць/чисел.

## Перевірка RC1

Перед публікацією RC1 пройшов build/vet/test, targeted та e2e перевірки Smart-шляхів, а також практичне тестування на робочій установці:

```text
4-hour optimization REPLACE cadence
ADD до завершення cadence при втраті контракту
Good → Paused → Good
Paused → Bad → ADD
```

Бінарний файл релізу побайтно збігається з установленим бінарником, на якому виконувалося фінальне тестування.

## Build information

```text
Release:        v0.3.0-rc.1
renterd:        v2.9.4-28-g8b23ee01
Backend commit: 8b23ee01
Web commit:     8b42f0b5
Network:        mainnet
```

```text
renterd.exe
Size: 61,659,108 bytes
SHA256: CE7A9343A1F58C52B44383EFF16C51B4A2D9D3D7DF35E5789ED32577EB988EB2
```

```text
renterd-smart-autopilot-v0.3.0-rc.1-windows-amd64.zip
Size: 22,298,517 bytes
SHA256: E9A016C202E53EEAD39935D9A77B6CFF0A9CB0449A2F1FF2E7349E5A744D5FDD
```

## Встановлення

1. Зробіть резервну копію даних/конфігурації renterd.
2. Завантажте `renterd-smart-autopilot-v0.3.0-rc.1-windows-amd64.zip` з GitHub Release.
3. Розпакуйте `renterd.exe` і `LICENSE`.
4. Зупиніть renterd і замініть executable.
5. Для першої перевірки використовуйте **Original** або **Smart Shadow**, а вже потім **Smart Active**.

Міграції бази виконуються стандартним механізмом renterd.

## Детальне порівняння

Див. [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md).

## Реліз

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.0-rc.1

## Ліцензія та upstream

Це неофіційна експериментальна похідна збірка на основі Sia Foundation renterd.

Див. `LICENSE`.

Це не офіційний реліз Sia Foundation. RC-версії призначені для тестування; зберігайте резервні копії та контролюйте логи/alerts.
