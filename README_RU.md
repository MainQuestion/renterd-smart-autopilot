# renterd Smart Autopilot

**Языки:** [English](README.md) · [Українська](README_UA.md) · Русский

Неофициальное экспериментальное расширение Sia `renterd`,
предназначенное для прозрачной оценки хостов, управления портфелем
контрактов и более разумного размещения новых данных.

> **Текущий релиз:** `v0.3.3-rc.1` --- Release Candidate с учётом
> расходов по контрактам (Σ storage, Σ ingress, Σ egress, Σ spent),
> переработанной оценкой Economics, паузой неудачных кандидатов,
> исправлением ошибки Merkle-proof после продления контракта и
> обновлённой таблицей Active contracts.

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

## Что нового в v0.3.3-rc.1

-   **Учёт расходов по контрактам.** В БД появились раздельные расходы на
    хранение (storage), приём данных (ingress) и чтение (egress). В
    таблице Active contracts добавлены колонки **Σ storage, Σ ingress,
    Σ egress, Σ spent**, а также My Data, Capacity, Initial/Remaining
    Allowance, Σ collateral и Used Collateral. Суммы считаются по цепочке
    продлений. Σ egress --- нижняя граница: отменённые и неудачные
    чтения renterd не видит.
-   **Economics переработан.** Collateral корректирует эффективную цену
    хранения, Economics Score --- это Cost Score скорректированной
    стоимости. Якоря стоимости и collateral берутся только по пригодным
    хостам; хосты с нулевой стоимостью не искажают якорь.
-   **Пауза кандидата Smart.** Кандидат, у которого формирование
    контракта провалилось по вине хоста, исключается из плана на 24
    часа (при повторных неудачах до 7 суток).
-   **Исправление Merkle-proof.** Исправлена ошибка (дефект upstream),
    из-за которой после продления контракта его рассылка ревизий
    отвергалась (`file contract revision parent has invalid Merkle
    proof`). Уже испорченные элементы чинятся один раз при запуске.
-   **Быстрее история цен.** Новый индекс `price_history` ускоряет
    запрос истории цен примерно в 35 раз.
-   **UI.** Выбор колонок запоминается в Hosts и Active contracts,
    сгруппированные колонки контрактов, цвета, сортировка, итоги под
    заголовками, убраны подсказки, повторяющие значение.

Подробнее --- в release notes релиза. Три миграции БД (`00067`, `00068`,
`00069`) выполняются автоматически и только добавляют колонки и индекс.

## Реализованный профиль хоста

В RC1 полностью завершены четыре из семи запланированных групп:

-   **G2 Performance** --- фактическая скорость upload/download из
    накопленной истории.
-   **G3 Resources** --- экспозиция renter к хосту: наши данные,
    свободное место, чужие данные, финансовая экспозиция, стоимость
    recovery/migration, остаток срока контракта и доля потерянных
    секторов.
-   **G4 Economics** --- стоимость storage/ingress/egress с поправкой на
    качество collateral: залог корректирует эффективную цену хранения, а
    итоговый Economics Score --- это Cost Score скорректированной
    стоимости (якорь p10 по пригодным хостам).
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

Smart Portfolio умеет:

-   заполнять недостающие contract slots через **Smart ADD**;
-   находить самый слабый текущий Good incumbent по Smart Overall;
-   применять настраиваемый **G4 Economics improvement threshold**;
-   сравнивать **KeepCost** и **MigrationCost** до переноса данных;
-   ставить контракт в восстанавливаемый **Economic Pause**, если замена
    экономически не оправдана;
-   выполнять безопасный **REPLACE**, сначала создавая новый контракт;
-   выводить старый контракт через persistent **Smart Drain**;
-   проверять безопасность данных до архивации;
-   временно исключать кандидата, у которого формирование провалилось по
    вине хоста (24 часа, при повторных неудачах до 7 суток; хранится в
    памяти и сбрасывается при перезапуске).

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

Настройка **Allow Existing Data on Overpriced Contracts** позволяет
Smart Active сохранять уже размещённые данные на Gouging-контракте
вместо переноса данных только из-за завышенной цены.

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
-   настройки отображения таблиц и чисел;
-   таблица **Active contracts** с группами колонок (General, Time,
    Contract data, Financial, Financial original), колонками Σ spent,
    Σ storage, Σ ingress, Σ egress, My Data, Capacity, Allowance и
    collateral, сортировкой, итогами под заголовками и сохранением
    выбора колонок.

## Проверка

Перед публикацией v0.3.3-rc.1 прошёл build/vet/test (включая e2e и
`-race` по затронутым пакетам), тест исправления Merkle-proof на
реальной цепочке, а также практическое тестирование на рабочей
установке: миграции `00067`--`00069`, разовое исправление 8 контрактов
(`invalid: 8` → `repaired: 8`, на следующем старте `invalid: 0`),
успешная рассылка ревизии, проверка таблицы контрактов в браузере.
Релизная сборка собрана из тех же коммитов, что и работающая на
рабочей установке.

Ранее, перед публикацией RC1, прошли проверки Smart-путей:

``` text
4-hour optimization REPLACE cadence
ADD до истечения cadence при выпадении контракта
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

## Установка

1.  Сделайте резервную копию данных и конфигурации renterd (для
    SQLite --- `.backup` БД и `metrics.sqlite`).
2.  Скачайте `renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip` из
    GitHub Release.
3.  Распакуйте `renterd.exe` и `LICENSE`.
4.  Остановите renterd и замените executable.
5.  Для первого запуска используйте **Original** или **Smart Shadow**, а
    затем при необходимости переходите в **Smart Active**.

Миграции базы выполняются стандартным механизмом renterd. При первом
запуске выполнятся миграции `00067`--`00069` (создание индекса занимает
секунды), а в логе появятся строки `contract elements checked` и, если
нужно, `contract elements repaired`. Откат: верните прежний
`renterd.exe`; старый бинарник стартует на БД с новыми миграциями.

## Подробное сравнение

См. [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md).

## Релиз

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.3-rc.1

## Лицензия и upstream

Это неофициальная экспериментальная производная сборка на основе Sia
Foundation renterd.

См. `LICENSE`.

Это не официальный релиз Sia Foundation. RC-версии предназначены для
тестирования; сохраняйте резервные копии и контролируйте логи/alerts.
