# renterd Smart Autopilot

**Языки:** [English](README.md) · [Українська](README_ua.md) · Русский

Неофициальное экспериментальное расширение Sia `renterd`, предназначенное для прозрачной оценки хостов, управления портфелем контрактов и более разумного размещения новых данных.

> **Текущий релиз:** `v0.3.0-rc.1` — первый протестированный Release Candidate с работающим Smart Portfolio decision/execution layer.

## Что изменилось

Оригинальные механизмы renterd остаются низкоуровневым исполнительным слоем: формирование контрактов, renew/refresh, upload/download, миграция и архивация.

Smart Autopilot добавляет над ними policy layer:

```text
измерения и история хостов
        ↓
многомерный профиль хоста
        ↓
Smart Overall + policy gates
        ↓
PortfolioPlan
        ↓
ADD / PAUSE / RECOVER / REPLACE
        ↓
существующие механизмы renterd
```

Цель — не заменить renterd одним новым «магическим score», а сохранить разные свойства хоста раздельными и использовать подходящие критерии для каждого решения.

## Реализованный профиль хоста

В RC1 полностью завершены четыре из семи запланированных групп:

- **G2 Performance** — фактическая скорость upload/download из накопленной истории.
- **G3 Resources** — экспозиция renter к хосту: наши данные, свободное место, чужие данные, финансовая экспозиция, стоимость recovery/migration, остаток срока контракта и доля потерянных секторов.
- **G4 Economics** — стоимость storage/ingress/egress плюс качество collateral. Итоговый Economics Score использует худшее значение между Cost Score и Collateral Score.
- **G5 Risk Coverage** — историческая стабильность storage/ingress/egress цен и collateral.

Ещё запланированы как полноценные отдельные scoring-группы:

- **G1 Reliability**
- **G6 Technical Compatibility**
- **G7 Network Diversity**

Часть compatibility/diversity проверок уже используется как eligibility/policy constraints, но пока это не завершённые группы 1–10.

## Smart Overall

Четыре законченные группы объединяются с пользовательскими весами:

```text
SmartOverall =
    G2 × W2 / 100 +
    G3 × W3 / 100 +
    G4 × W4 / 100 +
    G5 × W5 / 100
```

Persisted defaults: `25 / 25 / 25 / 25`.

Smart Overall намеренно не является единственным критерием для всех решений.

## Три режима

### Original

Использует обычную policy формирования и обслуживания контрактов renterd. Smart Portfolio decisions и Smart upload ordering не применяются.

Уже запущенные persistent Smart safety lifecycles безопасно завершаются.

### Smart Shadow

Рассчитывает те же Smart-решения, что и Active, и пишет их в `smart-autopilot.log`, но не выполняет новые Smart portfolio mutations.

Для новых upload Shadow оставляет оригинальный порядок `Uploader.Estimate()` и логирует его рядом со Smart order.

### Smart Active

Исполняет Smart Portfolio decisions и использует Smart Placement для новых пользовательских upload.

Кандидаты сортируются по `SmartOverall DESC` внутри существующего Good/Usable upload pool.

## Smart Portfolio

RC1 умеет:

- заполнять недостающие contract slots через **Smart ADD**;
- находить самый слабый текущий Good incumbent по Smart Overall;
- применять настраиваемый **G4 Economics improvement threshold**;
- сравнивать **KeepCost** и **MigrationCost** до переноса данных;
- ставить контракт в восстанавливаемый **Economic Pause**, если замена экономически не оправдана;
- выполнять безопасный **REPLACE**, сначала создавая новый контракт;
- выводить старый контракт через persistent **Smart Drain**;
- проверять безопасность данных до архивации.

Default Smart Replacement Threshold: `15%`.

Обычный optimization REPLACE имеет 4-часовой cadence. ADD им не блокируется.

## Gouging lifecycle

Smart Active добавляет отдельный безопасный lifecycle для существующего контракта, если единственная проблема хоста — Gouging:

```text
Good
→ Gouging Paused
→ recover / replace / timeout drain
```

Gouging-Paused контракт:

- не получает новые upload;
- может оставаться readable, если download price допустима;
- сохраняет persistent Smart state;
- может быть заменён сразу после появления eligible replacement;
- может запускать safety repair, когда доступных shards становится `K` или меньше.

Настройка **Keep Gouging Contract Until Replacement** определяет, может ли такой контракт продолжать ожидать/renew replacement, либо должен быть drained после safety boundary.

## Paused

В проекте также добавлено полноценное состояние `Paused` между Good и Bad:

- новые upload не принимаются;
- существующие данные могут оставаться readable;
- Paused placements считаются healthy redundancy;
- recovery может вернуть контракт в Good;
- hard threshold/timeout переводит его в Bad.

## История и диагностика

Smart Autopilot использует историю:

- upload/download speed;
- prices и collateral;
- scan results.

Отдельный:

```text
smart-autopilot.log
```

фиксирует ADD, REPLACE, economic pause/recovery, Gouging lifecycle, safety repair и сравнение Original/Smart upload order.

## UI

Добавлены:

- отдельная страница **Scoring**;
- настраиваемые legacy host-score параметры;
- веса Smart groups;
- Smart Replacement Threshold;
- Keep Gouging Contract Until Replacement;
- сгруппированные host-profile колонки;
- raw upload/download Mbps;
- Smart Overall;
- улучшенная сортировка хостов;
- настройки отображения таблиц и чисел.

## Проверка RC1

Перед публикацией RC1 прошёл build/vet/test, targeted и e2e проверки Smart-путей, а также практическое тестирование на рабочей установке:

```text
4-hour optimization REPLACE cadence
ADD до истечения cadence при выпадении контракта
Good → Paused → Good
Paused → Bad → ADD
```

Бинарник релиза побайтно совпадает с установленным бинарником, на котором проводилось финальное тестирование.

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

## Установка

1. Сделайте резервную копию данных и конфигурации renterd.
2. Скачайте `renterd-smart-autopilot-v0.3.0-rc.1-windows-amd64.zip` из GitHub Release.
3. Распакуйте `renterd.exe` и `LICENSE`.
4. Остановите renterd и замените executable.
5. Для первого запуска используйте **Original** или **Smart Shadow**, а затем при необходимости переходите в **Smart Active**.

Миграции базы выполняются стандартным механизмом renterd.

## Подробное сравнение

См. [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md).

## Релиз

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.0-rc.1

## Лицензия и upstream

Это неофициальная экспериментальная производная сборка на основе Sia Foundation renterd.

См. `LICENSE`.

Это не официальный релиз Sia Foundation. RC-версии предназначены для тестирования; сохраняйте резервные копии и контролируйте логи/alerts.
