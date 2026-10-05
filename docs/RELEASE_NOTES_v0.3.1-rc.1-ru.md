# Smart Autopilot — v0.3.1-rc.1

## Новая функция: Allow Existing Data on Overpriced Contracts

Smart Autopilot может сохранять существующие данные на контрактах, цена которых превышает настроенные ограничения, вместо переноса данных только из-за превышения цены.

Настройка:
- Backend: `AllowExistingDataOnOverpricedContracts`
- API JSON: `allowExistingDataOnOverpricedContracts`
- По умолчанию: `false`

При включении существующие данные могут оставаться на overpriced/Gouging-контракте; они остаются доступными для чтения при обычных ограничениях скачивания. Новые загрузки остаются запрещёнными. Gouging-хосты не выбираются для новых замен, а сам Gouging не вызывает принудительную миграцию существующих данных. Независимые repair/migration-механизмы не меняются, `MaxDownloadPrice` не ослабляется.

При Smart Gouging replacement подходящий новый контракт всё равно может быть создан, но старый incumbent не переводится в drain только из-за цены, пока настройка включена.

После успешной замены старый incumbent переходит в persistent legacy/replaced state: paused/read-only, исключается из Good upload pool и не заменяется повторно, пока сохраняется Gouging.

Если настройка выключена, legacy overpriced-контракты при Smart Active maintenance начинают drain, пока их хост остаётся Gouging. В Shadow только фиксируется планируемое действие; Original не выполняет эту Smart-мутацию.

---

## Усиление надёжности

### 1. Атомарная Smart-замена контракта

Коммит `602fae82` — `Harden Smart contract replacement atomically`

После успешного RHP formation новый контракт B, финализация incumbent A и связь A→B сохраняются в одной транзакции БД.

Это исключает частичные состояния, когда B существует, но A не финализирован и replacement-связь не сохранена.

При ошибке БД транзакция откатывается, Smart сообщает `partial_failure`, а fallback-запись B вне транзакции не выполняется.

Ошибка persistence блокирует дополнительные Smart replacements в текущем maintenance pass; следующий pass может повторить попытку.

Экономическая и Gouging-замены используют общий атомарный механизм.

### 2. Сохранение точной причины Smart drain при архивации

Коммит `18ee5027` — `Preserve Smart drain reasons through archival`

Причины Smart policy теперь сохраняются в persistent drain state и явно преобразуются в причину архивации:

| Smart policy | Persistent drain | Архивация |
|---|---|---|
| `gouging` после `MaxDowntimeHours` | `gougingTimeout` | `smartgouging` |
| `expiration_safety` | `expirationSafety` | `smartexpiration` |
| `overpriced_data_not_allowed` | `overpricedDataNotAllowed` | `smartoverpriced` |
| Economic/Gouging replacement | `replacement` | `smartreplacement` |

Миграция БД не требуется: используется существующее `smart_contract_state.reason`.

`FinalizeSmartDrain` теперь fail-closed для неизвестной причины вместо молчаливого назначения `smartreplacement`.

Уже запущенные persistent drains продолжаются при смене Smart mode. Исторические `gougingTimeout` не backfill-ятся, поскольку исходную причину нельзя надёжно восстановить.

---

## Пост-RC1 исправление: Gouging recovery и occupancy заменённых контрактов

### Коммит `25318c8c` — `fix smart gouging recovery and replaced occupancy`

Исправлен lifecycle контракта, который был заменён из-за Gouging, но его исходный хост позже восстановился.

Целевой lifecycle:

```text
Good
  ↓
Gouging
  ↓
Paused / replacement
  ↓
Legacy / replaced
  ↓
хост восстановился
  ↓
Good + Replaced=true
```

Ключевая семантика:

- когда хост перестаёт быть Gouging, **любая Gouging pause, включая legacy/replaced state, возвращается в `Good`**;
- `Replaced=true` сохраняется как исторический признак;
- данные **не мигрируют обратно** с replacement-контракта;
- данные, оставшиеся на восстановленном контракте, остаются на нём;
- восстановленный контракт снова является Good и может занимать слот Smart portfolio;
- восстановившийся хост позднее может получить **новый** контракт, если Smart независимо выберет его;
- обратная миграция только из-за восстановления исходного хоста не выполняется;
- пока хост остаётся Gouging, существующий legacy/replacement lifecycle не меняется.

### Occupancy

`Replaced=true` само по себе не делает контракт несуществующим для portfolio accounting.

| Состояние | `Replaced` | Занимает слот |
|---|---:|---:|
| Good | false | да |
| Good | true | да |
| Paused | false | да |
| Paused | true | нет |
| Bad | false | нет |
| Bad | true | нет |

Таким образом, восстановленный `Replaced+Good` снова учитывается в портфеле, а `Replaced+Paused`, который drainится, слот не занимает.

---

## Живая проверка recovery fix

Исправление протестировано на работающем mainnet renterd.

Два контракта, ранее прошедшие:

```text
Gouging → Pause → Replacement → Legacy
```

после запуска нового бинарника получили:

```text
smart pause
contractID: e9421132765cd59bfecb3f3f1b28ca34f987f66e13bcc46eccc5ea3c5e3425ec
decision: recover
why: host no longer gouging
outcome: success
legacy: true
pausedAt: 2026-10-05T14:55:42+03:00
```

и:

```text
smart pause
contractID: 70887117eebd7bfdc9b5a2f2307a51f1a448a43cf6a62d74b31b57dc1bb91d11
decision: recover
why: host no longer gouging
outcome: success
legacy: true
pausedAt: 2026-10-05T14:55:42+03:00
```

Обычный contract-maintenance pass затем сообщил, что оба контракта снова usable, и обновил два контракта.

Smart portfolio сообщил:

```text
wantedContracts: 10
occupancy: 12
plannedOccupancy: 12
```

что подтверждает участие восстановленных `Replaced+Good` контрактов в occupancy.

Обратная миграция для них не запускалась.

В том же maintenance pass Smart независимо выполнил обычную экономическую замену другого incumbent, что подтверждает отсутствие конфликта recovery с обычной оптимизацией.

---

## Проверка и тесты

Полный Autopilot test suite:

```powershell
go test ./autopilot/... -count=1 -v
```

Прошли тесты для:

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

Финальная сборка выполнена из `25318c8c`.

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

Установленный бинарник проверен по SHA256 и совпал со сборкой.

Предыдущий установленный бинарник сохранён перед заменой:

```text
Size: 61,760,113 bytes
SHA256: A2601E64D20A5CB4518B8298B7FF3D44FBA1CF197768487932919B0941708442
```

---

## Текущая Git-база

- `602fae82` — `Harden Smart contract replacement atomically`
- `18ee5027` — `Preserve Smart drain reasons through archival`
- `25318c8c` — `fix smart gouging recovery and replaced occupancy`

После последнего fix-коммита рабочее дерево было чистым.

Тег `v0.3.1-rc.1` создан до `25318c8c`, поэтому финальный бинарник идентифицируется как:

```text
v0.3.1-rc.1-1-g25318c8c
```

Это сохраняет точное Git-происхождение протестированного бинарника.

---

## Функциональное резюме

Smart Autopilot — отдельный policy layer поверх существующих механизмов renterd для контрактов, migration, repair и передачи данных. Решения принимаются по отдельным Smart scoring dimensions, а не по одному непрозрачному host score.

Текущая функциональность:

- Smart Active / Shadow / Original;
- Smart Overall scoring;
- выбор хостов и балансировка портфеля;
- ADD / KEEP / PAUSE / REPLACE / EXIT;
- экономическая замена по сравнению стоимости миграции с оставшейся стоимостью контракта;
- persistent Smart draining;
- Gouging replacement;
- Gouging pause/recovery lifecycle;
- legacy/replaced lifecycle;
- участие восстановленных `Replaced+Good` в portfolio occupancy;
- настройка сохранения существующих данных на overpriced-контрактах;
- явные причины Smart drain и archival;
- атомарное сохранение replacement и связь A→B;
- отсутствие автоматической обратной миграции после восстановления заменённого хоста.

Smart layer остаётся настраиваемым и использует существующие низкоуровневые механизмы renterd для formation, renewal, repair, migration, upload и download.
