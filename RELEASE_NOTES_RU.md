# renterd Smart Autopilot v0.3.0-rc.1

Это первый протестированный Release Candidate, в котором Smart Autopilot работает как полноценный decision/execution layer, а не только как расширение системы оценки хостов.

## Основное

- Режимы Smart Portfolio: **Original / Smart Shadow / Smart Active**
- Четыре завершённые группы профиля:
  - G2 Performance
  - G3 Resources
  - G4 Economics
  - G5 Risk Coverage
- Пользовательские веса **Smart Overall**
- **Smart ADD** для недостающих contract slots
- **Optimization REPLACE** с настраиваемым G4 threshold
- Экономический gate: `KeepCost > MigrationCost`
- Восстанавливаемый **Economic Pause**
- Persistent **Smart Drain** с verification перед архивацией
- **Gouging Pause**, Gouging REPLACE, timeout drain и safety repair
- **Smart Placement** новых пользовательских upload по `SmartOverall DESC`
- Отдельный `smart-autopilot.log`
- Расширенный Scoring/Hosts UI
- Улучшенная сортировка хостов и настройки таблиц

## Smart Portfolio

Smart Autopilot сохраняет оригинальные механизмы renterd в качестве низкоуровневых executors и добавляет над ними policy layer.

Обычная optimization replacement выполняется так:

```text
найти самый слабый incumbent
→ найти лучшего кандидата
→ проверить G4 threshold
→ сравнить KeepCost и MigrationCost
→ сформировать replacement contract
→ Smart Drain старого contract
→ migrate
→ verify
→ archive
```

Новый контракт всегда формируется до начала Smart Drain старого.

Обычный optimization REPLACE ограничен 4-часовым cadence. Smart ADD этим интервалом не блокируется.

## Economic Pause

Если кандидат проходит G4 threshold, но миграция экономически не оправдана:

```text
KeepCost <= MigrationCost
```

incumbent переходит в восстанавливаемый Economic Pause.

На следующих optimization opportunities он пересматривается и может либо вернуться в Good, либо перейти к REPLACE, когда экономика изменится.

## Gouging lifecycle

Если Gouging — единственная проблема хоста в Smart Active, контракт может перейти в отдельный Gouging Pause вместо немедленного Bad.

Lifecycle поддерживает:

```text
pause
→ recover
→ replace
→ safety repair
→ timeout drain
```

Настройка **Keep Gouging Contract Until Replacement** определяет, может ли контракт продолжать ждать/renew replacement, либо должен быть drained после safety boundary.

## Smart Placement

Для новых пользовательских upload:

```text
Original     → Uploader.Estimate() ASC
Smart Shadow → Original order + logged Smart order
Smart Active → SmartOverall DESC
```

Migration uploads намеренно продолжают использовать существующую policy renterd.

## Проверка RC1

RC1 прошёл build/vet/test, targeted и e2e проверки Smart-путей.

Также выполнено практическое тестирование на рабочей установке:

```text
4-hour optimization REPLACE cadence
ADD до истечения cadence при выпадении контракта
Good → Paused → Good
Paused → Bad → ADD
```

Публикуемый executable побайтно совпадает с установленным бинарником, на котором выполнялось финальное тестирование.

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

Это неофициальная экспериментальная RC-сборка на основе Sia renterd. Перед тестированием сделайте резервную копию данных/конфигурации и по возможности начните со Smart Shadow.
