# renterd Smart Autopilot v0.3.0-rc.1

Це перший протестований Release Candidate, у якому Smart Autopilot працює як реальний decision/execution layer, а не лише як розширення оцінки хостів.

## Основне

- Режими Smart Portfolio: **Original / Smart Shadow / Smart Active**
- Чотири завершені групи профілю:
  - G2 Performance
  - G3 Resources
  - G4 Economics
  - G5 Risk Coverage
- Налаштовувані ваги **Smart Overall**
- **Smart ADD** для відсутніх contract slots
- **Optimization REPLACE** з налаштовуваним G4 threshold
- Економічний gate: `KeepCost > MigrationCost`
- Відновлюваний **Economic Pause**
- Persistent **Smart Drain** з verification перед архівацією
- **Gouging Pause**, Gouging REPLACE, timeout drain і safety repair
- **Smart Placement** для нових upload через `SmartOverall DESC`
- Окремий `smart-autopilot.log`
- Розширений Scoring/Hosts UI
- Покращене сортування хостів і налаштування таблиць

## Smart Portfolio

Smart Autopilot залишає оригінальні механізми renterd низькорівневими executors і додає над ними policy layer.

Звичайна optimization replacement проходить так:

```text
знайти найслабший incumbent
→ знайти кращого кандидата
→ перевірити G4 threshold
→ порівняти KeepCost і MigrationCost
→ створити replacement contract
→ Smart Drain старого contract
→ migrate
→ verify
→ archive
```

Новий контракт завжди створюється до початку Smart Drain старого.

Звичайний optimization REPLACE має 4-годинний cadence. Smart ADD ним не блокується.

## Economic Pause

Якщо кандидат проходить G4 threshold, але міграція економічно не виправдана:

```text
KeepCost <= MigrationCost
```

incumbent переходить у відновлюваний Economic Pause.

На наступних optimization opportunities він повторно оцінюється і може або повернутися в Good, або перейти до REPLACE, коли економіка зміниться.

## Gouging lifecycle

Якщо Gouging — єдина проблема хоста в Smart Active, контракт може перейти у спеціальний Gouging Pause замість негайного Bad.

Lifecycle підтримує:

```text
pause
→ recover
→ replace
→ safety repair
→ timeout drain
```

Параметр **Keep Gouging Contract Until Replacement** визначає, чи може контракт продовжувати очікувати/renew replacement, або має бути drained після safety boundary.

## Smart Placement

Для нових користувацьких upload:

```text
Original     → Uploader.Estimate() ASC
Smart Shadow → Original order + logged Smart order
Smart Active → SmartOverall DESC
```

Migration uploads навмисно залишаються на існуючій policy renterd.

## Перевірка RC1

RC1 пройшов build/vet/test, targeted та e2e перевірки Smart-шляхів.

Також було виконано практичне тестування на робочій установці:

```text
4-hour optimization REPLACE cadence
ADD до завершення cadence при втраті контракту
Good → Paused → Good
Paused → Bad → ADD
```

Опублікований executable побайтно збігається з установленим бінарником, на якому проводилося фінальне тестування.

## Build

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

Це неофіційна експериментальна RC-збірка на основі Sia renterd. Перед тестуванням зробіть резервну копію даних/конфігурації та за можливості починайте зі Smart Shadow.
