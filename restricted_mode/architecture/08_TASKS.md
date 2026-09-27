# Задания следующего этапа

Дата: 2026-09-27. Base commit исследования: `42a84aec8abf72e54958ddf8ca9480514cc0d390`.
**Задания подготовлены; Workers не запускались. По отдельному поручению выполнен live RPC опыт AUTH-01 — см. [результаты](AUTH-01_RESULT.md). Остальные задания не выполнялись.** READY FOR SPIKES означает готовность постановки, а не разрешение менять код. Перед выполнением зафиксировать новый base SHA, проверить AGENTS и получить отдельный scope эксперимента. Все задачи security; review плана обязательно перед кодом. В первом проходе новые агенты не запускались.

## Общие условия всех заданий

- Protected paths: `restricted_mode/SPEC.md`, `restricted_mode/THREAT_MODEL.md`, `restricted_mode/SECURITY_INVARIANTS.md`, `local-config/`, signing/entitlements/dependency versions. Не изменять без отдельного решения владельца.
- Только выделенный тестовый аккаунт и синтетические данные; не читать личный sandbox/Keychain, не использовать известные тестовые аккаунты без контроля владения. Не сохранять auth keys, PIN, api_hash, provisioning или личный контент в Git/отчётах.
- Новый harness допустим в предложенном `Tests/RestrictedModeExperiments/` только после отдельного разрешения. Сначала проверить возможность переиспользования существующих Tests/TelegramCoreBuildTest и UI test infrastructure. Добавление BUILD внутри harness допустимо в scope эксперимента; production BUILD не менять молча.
- Результат: task/base/head, изменённые пути, команды, exit codes, обезличенные наблюдения, ограничения и вывод о гипотезе. Успешное опровержение небезопасного варианта считается результатом исследования, не FAIL исполнителя.
- Остановить затронутую работу при конфликте инварианта, выходе за разрешённые пути, необходимости реальных credentials/устройства или обнаружении неучтённого writer. Продолжить независимые документальные части.

## AUTH-01 — первое задание: граница авторизации

**Роль:** исследователь/программист эксперимента. **Риск:** security. **Зависимости:** G0, отдельное разрешение и контролируемая test fixture. **Инварианты:** I01/I02/I06/I10; R03/R09, D01.

**Цель:** проверить гипотезу «Restricted runtime с обычной авторизацией аккаунта не может получить hidden cloud history». Это самый ранний эксперимент, потому что его результат может исключить целое семейство решений.

**Scope:** чтение `Account/Account.swift`, `Network/Network.swift`, `TelegramEngine/Messages/SearchMessages.swift`, `SyncCore_AccountBackupDataAttribute.swift`, AccountManager; экспериментальный harness в `Tests/RestrictedModeExperiments/`; отчёт в `restricted_mode/architecture/`. Никаких production hooks, UI-фильтров, изменения серверного протокола или копирования личных ключей.

**Процедура:**

1. Подготовить аккаунт A и контролируемого собеседника B; visible/hidden dialogs, уникальное синтетическое сообщение в hidden. Подтвердить его обычное получение в test Full fixture.
2. Составить доступный материал кандидатного Restricted runtime: auth/notification/backup/login tokens. Для каждой копии указать путь и доступность; значения не выводить.
3. Под теми же полномочиями запросить hidden history через `Network.request` без UI policy. Использовать валидный known peer/access hash; отдельно проверить получение peer через dialogs/search. Записать только тип ответа и результат canary assertion.
4. Раздельно оценить T2 extraction, T3 use-in-place, T4 arbitrary RPC. Не называть обычный вызов harness доказательством взлома Keychain.
5. Негативный контроль: контекст без авторизации не получает историю. Ошибка сети/peer не считается доказательством ограничения прав.

**Приёмка:** воспроизводимый вывод «гипотеза опровергнута / подтверждена только для перечисленных путей / данных недостаточно»; точный authority path; обновление C01/ADR-001 и предложение владельцу. **Проверки:** V01 и build harness. **Остановка:** нет test fixture, нужен production account, невозможна изоляция данных, попытка ослабить T3/T4 без решения.

## STORAGE-01 — ключи, metadata, media и envelope

**Роль:** исследователь криптографической/файловой границы. **Зависимости:** AUTH-01 inventory; выбор D01 для окончательного вывода. **Инварианты:** I01/I02/I03/I09/I10.

**Scope:** чтение BuildConfig, Postbox, AccountManager, SyncCore backup; синтетический harness в `Tests/RestrictedModeExperiments/`; architecture reports. Не менять действующие форматы/алгоритмы приложения.

**Цель/работа:** замкнуть key inventory E02–E08/E23/E24; декодировать синтетические metadata/atomic-state/WAL/media; исследовать доступные криптобиблиотеки; сравнить envelope/KDF/device binding, rotation/PIN change/restore. Измерить стоимость KDF и поведение Keychain на физическом устройстве; не обещать аппаратный лимит app PIN без измерений/документации.

**Результат/приёмка:** таблица всех ключевых копий и stores, доказанные доступы T2/T3, кандидат протокола и версии, failure/recovery cases, список неустранённых путей. **Проверки:** V02/V03/V11; benchmark на синтетических данных, build harness. **Остановка:** непроверенный самодельный crypto protocol, необходимость менять реальные ключи/backup, отсутствие device evidence для соответствующего claim.

## EXT-01 — полная карта ingress и внешних поверхностей

**Роль:** исследователь. **Зависимости:** G0; независима от D01 для инвентаризации. **Инварианты:** I02/I05/I06/I08/I10.

**Scope:** чтение `Telegram/`, `submodules/TelegramCore/`, `TelegramUI/`, `Postbox/`, `TelegramCallsUI/`, соответствующих BUILD; запись только architecture reports. Никаких отключений targets.

**Работа:** дополнить E14–E24 всеми history/search/contact/story/media ingress, direct DB reads, folders/counters/forward/references/shared media, logging, Photos/Files, App Intents, Live Activities, Watch/CarPlay и старым NotificationContent target. Указать каждый process/store/key/output, обнаруженные targets и реально активирующие условия. Проверить прямые print в Spotlight и intent donation CallKit.

**Приёмка:** path→policy boundary→тест V05/V06/V12; неизвестные ветви явно BLOCKED; наличие символа не считать coverage. **Проверки:** source trace и повторяемый поиск, без запуска личного аккаунта. **Остановка:** конфликт semantics D02/D08 блокирует соответствующий expected result, но не карту.

## NOTIFY-01 — APNs, fallback и вызовы

**Роль:** исследователь/программист эксперимента. **Зависимости:** EXT-01, тестовое устройство/подпись, согласованное ожидаемое поведение D03. **Инварианты:** I01/I02/I06/I08/I09/I10.

**Scope:** чтение NotificationService, NotificationContent, Telegram BUILD, CallKitIntegration; отдельный harness/test instructions; architecture reports. Не менять entitlements, provisioning, bundle ID или server notification settings личного аккаунта.

**Работа:** получить только список разрешений фактической подписи без вывода credentials; воспроизвести V07 (success, timeout, crash, no-run, sound, badge, attachment, VoIP, delivered Full notification, mirroring). Различать подавление content и любого следа события.

**Приёмка:** матрица observed vs required для каждого fallback; entitlement доказан profile/signed binary или UNKNOWN; решение о достижимости D03. **Проверки:** physical device V07; Simulator — дополнительный subset. **Остановка:** нет approved signing/device/payload fixture; недопустимый fallback нельзя скрыть пустым текстом или молчаливым отключением уведомлений.

## SYNC-01 — два состояния и возврат Full

**Роль:** исследователь state/sync. **Зависимости:** AUTH-01, D01; cloud/secret semantics D04/D05. **Инварианты:** I02/I05/I07/I08/I09.

**Scope:** чтение State, Account, SecretChats/SyncCore; test harness и architecture reports. Не подключать два production Account к общей базе.

**Работа:** тест отдельного state ownership на synthetic stores; Full→Restricted→Full с update/edit/delete/expire, gaps/replay/duplicates, reconnect, queued outgoing/downloads и read states. Secret-chat keys/qts/ack/outgoing исследовать отдельно; определить, кто владеет MTProto session и межпроцессными writers.

**Приёмка:** соответствие каждого cursor реально записанному состоянию, не копировать Restricted cursor в Full; перечислить невосстановимый контент и конфликт требований. **Проверки:** V08/V06; build harness. **Остановка:** восстановление требует хранения запрещённого plaintext или доступных Full keys в Restricted; это ACR, а не повод отключить проверку.

## LIFE-01 — отзыв сессии и восстановление

**Роль:** исследователь lifecycle. **Зависимости:** STORAGE-01, D06; интерфейсы B/C условные до G2. **Инварианты:** I01/I03/I04/I09.

**Scope:** чтение AppLock/AppLockState, AppDelegate, Account/SharedAccountContext, extensions; synthetic harness, architecture reports. Никаких изменений UX Full в production.

**Работа:** V04/V09/V10: generation checks, delayed callbacks, writer cancellation, close handles, extension concurrency, system lock/background/suspension/kill/reboot/scenes. Проверить доступность protected data на устройстве и остаточные копии памяти в границах модели.

**Приёмка:** протокол Closing с измеренным завершением или обоснованный blocker; отсутствие auto-open при crash/policy corruption; crash table для каждого шага. **Проверки:** build harness + Simulator/device subset, реальное устройство обязательно для system-lock claim. **Остановка:** невозможно отличить lock при желаемом UX, writer не отзывается, требуется новое исключение Full.

## REVIEW-01 — независимая проверка результатов

**Роль:** проверяющий. **Зависимости:** конкретный snapshot отчётов/экспериментов. **Scope:** чтение diff/кода/evidence, запись отдельного отчёта в architecture; код не редактировать. **Инварианты:** все применимые I01–I10.

Проверить требования, альтернативные пути утечки, соответствие T-уровням, отсутствие секретов, реальность тестовых fixture и связь SHA. Замечание содержит severity, путь/символ, воспроизведение и требование. **Приёмка:** явный PASS/CHANGES_REQUESTED/BLOCKED с остаточными рисками. **Проверки:** V14 и выборочное воспроизведение разрешённых экспериментов. **Остановка:** missing evidence не заменять доверием к отчёту Worker; новый SHA требует повторного review.

## INTEGRATE-01 — подготовка интеграции после G2

**Роль:** интегратор. **Зависимости:** принятые требования/ADR, готовый implementation TASK, review конкретного head, разрешённая Git policy. Сейчас **BLOCKED**.

**Scope:** будущая интеграционная ветка и отчёт; функциональные исправления только отдельным Worker TASK. **Инварианты:** I08/I10 и все затронутые.

**Приёмка:** reviewed head не изменился; candidate на актуальном main собран, применимые проверки пройдены, согласование связано с candidate SHA; после разрешённого merge — smoke. **Проверки:** 06 и baseline commands 00. **Остановка:** конфликт, смена SHA, провал теста или отсутствие полномочий commit/merge/push. Security finding не отменяется ради зелёной сборки.
