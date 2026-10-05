# renterd Smart Autopilot

**Языки:** [English](README.md) · [Українська](README_UA.md) · Русский

Неофициальное экспериментальное расширение Sia `renterd`,
предназначенное для прозрачной оценки хостов, управления портфелем
контрактов и более разумного размещения новых данных.

> **Текущий релиз:** `v0.3.1-rc.1` --- обновлённый Release Candidate с
> опциональной политикой сохранения существующих данных на overpriced
> (Gouging) контрактах.

## Что изменилось

Оригинальные механизмы renterd остаются низкоуровневым исполнительным
слоем: формирование контрактов, renew/refresh, upload/download, миграция
и архивация.

Smart Autopilot добавляет над ними policy layer:

``` text
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

Цель --- не заменить renterd одним новым «магическим score», а сохранить
разные свойства хоста раздельными и использовать подходящие критерии для
каждого решения.

## Реализованный профиль хоста

В RC1 полностью завершены четыре из семи запланированных групп:

-   **G2 Performance** --- фактическая скорость upload/download из
    накопленной истории.
-   **G3 Resources** --- экспозиция renter к хосту: наши данные,
    свободное место, чужие данные, финансовая экспозиция, стоимость
    recovery/migration, остаток срока контракта и доля потерянных
    секторов.
-   **G4 Economics** --- стоимость storage/ingress/egress плюс качество
    collateral. Итоговый Economics Score использует худшее значение
    между Cost Score и Collateral Score.
-   **G5 Risk Coverage** --- историческая стабильность
    storage/ingress/egress цен и collateral.

Ещё запланированы как полноценные отдельные scoring-группы:

-   **G1 Reliability**
-   **G6 Technical Compatibility**
-   **G7 Network Diversity**

Часть compatibility/diversity проверок уже используется как
eligibility/policy constraints, но пока это не завершённые группы 1--10.

## Smart Overall

Четыре законченные группы объединяются с пользовательскими весами:

``` text
SmartOverall =
    G2 × W2 / 100 +
    G3 × W3 / 100 +
    G4 × W4 / 100 +
    G5 × W5 / 100
```

Persisted defaults: `25 / 25 / 25 / 25`.

Smart Overall намеренно не является единственным критерием для всех
решений.

## Три режима

### Original

Использует обычную policy формирования и обслуживания контрактов
renterd. Smart Portfolio decisions и Smart upload ordering не
применяются.

Уже запущенные persistent Smart safety lifecycles безопасно завершаются.

### Smart Shadow

Рассчитывает те же Smart-решения, что и Active, и пишет их в
`smart-autopilot.log`, но не выполняет новые Smart portfolio mutations.

Для новых upload Shadow оставляет оригинальный порядок
`Uploader.Estimate()` и логирует его рядом со Smart order.

### Smart Active

Исполняет Smart Portfolio decisions и использует Smart Placement для
новых пользовательских upload.

Кандидаты сортируются по `SmartOverall DESC` внутри существующего
Good/Usable upload pool.

## Smart Portfolio

RC1 умеет:

-   заполнять недостающие contract slots через **Smart ADD**;
-   находить самый слабый текущий Good incumbent по Smart Overall;
-   применять настраиваемый **G4 Economics improvement threshold**;
-   сравнивать **KeepCost** и **MigrationCost** до переноса данных;
-   ставить контракт в восстанавливаемый **Economic Pause**, если замена
    экономически не оправдана;
-   выполнять безопасный **REPLACE**, сначала создавая новый контракт;
-   выводить старый контракт через persistent **Smart Drain**;
-   проверять безопасность данных до архивации.

Default Smart Replacement Threshold: `15%`.

Обычный optimization REPLACE имеет 4-часовой cadence. ADD им не
блокируется.

## Gouging lifecycle

Smart Active добавляет отдельный безопасный lifecycle для существующего
контракта, если единственная проблема хоста --- Gouging:

``` text
Good
→ Gouging Paused
→ recover / replace / timeout drain
```

Gouging-Paused контракт:

-   не получает новые upload;
-   может оставаться readable, если download price допустима;
-   сохраняет persistent Smart state;
-   может быть заменён сразу после появления eligible replacement;
-   может запускать safety repair, когда доступных shards становится `K`
    или меньше.

Настройка **Keep Gouging Contract Until Replacement** определяет, может
ли такой контракт продолжать ожидать/renew replacement, либо должен быть
drained после safety boundary.

### Allow Existing Data on Overpriced Contracts

Новая настройка **Allow Existing Data on Overpriced Contracts**
позволяет Smart Active сохранять уже размещённые данные на
Gouging-контракте вместо переноса данных только из-за завышенной цены.

-   Настройка находится на странице **Scoring** и **по умолчанию
    выключена**, поэтому прежнее поведение сохраняется.
-   При включённой настройке Smart по-прежнему может сформировать
    replacement, но старый контракт не переводится в drain только
    потому, что replacement уже создан.
-   После успешного Gouging replacement старый контракт становится
    **legacy**-контрактом только для чтения, пока его хост остаётся
    Gouging: новые upload туда не идут, renew/refresh для сохранения
    Gouging-контракта не выполняются, слот Smart Portfolio он не
    занимает.
-   Существующие данные могут оставаться доступными через действующие
    read/download-проверки. Если хост перестаёт быть Gouging,
    legacy-контракт может восстановиться через обычный Gouging recovery
    lifecycle.
-   Если настройку снова выключить, **Smart Active** может запустить
    drain сохранённых legacy-данных.
-   Защита от истечения контракта независима от ценовой политики: для
    Gouging/legacy-контракта, который больше не будет продлеваться, при
    входе во вторую половину renew window запускается существующий
    expiration-safety drain.
-   Gouging safety repair не заставляет переносить данные, пока политика
    разрешает их сохранять; в соответствующем случае решение фиксируется
    как `allowed_existing_data`.

## Paused

В проекте также добавлено полноценное состояние `Paused` между Good и
Bad:

-   новые upload не принимаются;
-   существующие данные могут оставаться readable;
-   Paused placements считаются healthy redundancy;
-   recovery может вернуть контракт в Good;
-   hard threshold/timeout переводит его в Bad.

## История и диагностика

Smart Autopilot использует историю:

-   upload/download speed;
-   prices и collateral;
-   scan results.

Отдельный:

``` text
smart-autopilot.log
```

фиксирует ADD, REPLACE, economic pause/recovery, Gouging lifecycle,
safety repair и сравнение Original/Smart upload order.

## UI

Добавлены:

-   отдельная страница **Scoring**;
-   настраиваемые legacy host-score параметры;
-   веса Smart groups;
-   Smart Replacement Threshold;
-   Keep Gouging Contract Until Replacement;
-   Allow Existing Data on Overpriced Contracts;
-   сгруппированные host-profile колонки;
-   raw upload/download Mbps;
-   Smart Overall;
-   улучшенная сортировка хостов;
-   настройки отображения таблиц и чисел.

## Проверка RC1

Перед публикацией RC1 прошёл build/vet/test, targeted и e2e проверки
Smart-путей, а также практическое тестирование на рабочей установке:

``` text
4-hour optimization REPLACE cadence
ADD до истечения cadence при выпадении контракта
Good → Paused → Good
Paused → Bad → ADD
```

Бинарник релиза побайтно совпадает с установленным бинарником, на
котором проводилось финальное тестирование.

## Build information

``` text
Release:        v0.3.1-rc.1
renterd:        v0.3.1-rc.1-1-g25318c8c
Backend commit: 25318c8c
Web commit:     0d084776
Network:        mainnet
```

``` text
renterd.exe
Size: 61,758,577 bytes
SHA256: 9CC3D68BDECB5B8AC885464F21F841A4832887A1BE58D9404CC4B26188CC30EB2
```

``` text
renterd-smart-autopilot-v0.3.1-rc.1-windows-amd64.zip
Size: 22322705 bytes
SHA256: ACE44262B67D7000FF1F090599AC262135E39011EE469ACB49A1B18D099D61CA
```

## Установка

1.  Сделайте резервную копию данных и конфигурации renterd.
2.  Скачайте `renterd-smart-autopilot-v0.3.1-rc.1-windows-amd64.zip` из
    GitHub Release.
3.  Распакуйте `renterd.exe` и `LICENSE`.
4.  Остановите renterd и замените executable.
5.  Для первого запуска используйте **Original** или **Smart Shadow**, а
    затем при необходимости переходите в **Smart Active**.

Миграции базы выполняются стандартным механизмом renterd.

## Подробное сравнение

См. [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md).

## Релиз

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.1-rc.1

## Лицензия и upstream

Это неофициальная экспериментальная производная сборка на основе Sia
Foundation renterd.

См. `LICENSE`.

Это не официальный релиз Sia Foundation. RC-версии предназначены для
тестирования; сохраняйте резервные копии и контролируйте логи/alerts.
