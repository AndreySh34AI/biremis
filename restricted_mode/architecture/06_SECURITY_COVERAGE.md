# Проект инвариантов и матрица проверки

**DRAFT / NOT APPROVED**. Дата: 2026-09-27. Commit: `42a84aec8abf72e54958ddf8ca9480514cc0d390`.
Это проект отсутствующего SECURITY_INVARIANTS.md. ID I01–I10 сохранены из brief. CODE означает исследование исходников; ни один инвариант Restricted не получил runtime PASS.

## Свойства и покрытие

| ID | Проект обязательного свойства | Требования / код | Проверки | Статус |
|---|---|---|---|---|
| I01 | Full secrets недоступны Restricted/Locked в принятой модели угроз | R01/R08/R09; E02–E07/E09/E18/E19 | V01/V02/V04/V11 | BLOCKED D01/D07; текущие общие secrets не удовлетворяют проекту |
| I02 | Restricted persistence не содержит запрещённой декодируемой истории | R03/R09; E08/E14–E20/E23/E24 | V02/V05/V06/V09 | Не реализовано; найдены независимые stores и ingress |
| I03 | При отсутствии валидной policy чувствительный доступ закрыт | R03/R05; E11 | V03/V09/V10 | Не реализовано; default LockState нельзя использовать как policy fallback |
| I04 | После отзыва нет операций старой сессии | R06/R08; E03/E13/E22 | V04/V09 | Не доказано; process-local exclusivity не покрывает extensions |
| I05 | Все записи и выдача применяют согласованную policy generation | R03–R05/R08; E14–E20/E24 | V05/V06/V09 | Полный граф ещё не построен |
| I06 | Counters/search/preview/navigation не раскрывают hidden history | R03/R10; E15–E21/E23 | V05/V06/V07/V12 | Не реализовано; семантика identity/references D02 |
| I07 | Full возвращается в согласованное состояние после Restricted | R02/R08; E14/E24 | V08 | BLOCKED D04/D05; полного сохранения недоступного серверного контента не обещаем |
| I08 | Full сохраняет функции вне принятых исключений | R02/R06/R07; E01/E11/E18–E22 | V07/V08/V12/V13 | Baseline build отдельно от функциональной регрессии; исключения D06/D08 не приняты |
| I09 | Ошибки/миграции/restore не открывают Full и не разрешают всё | R01/R04/R09; E06/E08/E11 | V03/V09/V10/V11 | Не реализовано; delete-on-error нельзя перенести без анализа |
| I10 | Заявления о защите соответствуют доказанному уровню | R09/R10; вся карта | V14 и review evidence | Отчёты различают CODE / runtime / неизвестно; безопасность не заявлена |

## Протокол экспериментов

Все будущие V01–V14 выполняются на синтетических данных и выделенном тестовом аккаунте. Набор S: один visible и один hidden cloud dialog, общая группа, Saved Messages, archived/pinned/muted, topic; отдельные canary в тексте, имени, username, phone, filename, avatar, thumbnail, audio/video. Secret-chat fixture отдельный. Evidence не содержит реальных ключей/PIN/чужих сообщений: хранить assert/result, SHA, среду и обезличенное воспроизведение.

| ID | Setup / действия | Ожидаемое наблюдение | Среда, инструменты и evidence | Ограничение |
|---|---|---|---|---|
| V01 Auth authority | S; в кандидатном Restricted runtime запросить hidden history напрямую, минуя UI, затем после lock | Для принятого T3/T4 запрос не должен раскрыть контент; успех запроса опровергает кандидата | Тестовый network harness, response assertion, список доступных auth paths | Один отказ из-за неизвестного peer/access hash не доказывает ограничение полномочий |
| V02 Key/store closure | S; записать media/history, lock; исследовать копию контейнера, metadata, keychain representations, WAL/SHM/temp и backups | Ни запрещённый контент, ни материал для его раскрытия не доступны в выбранном T-уровне | Parser соответствующих форматов, DB tools, media decode; manifest и assertions | Canary grep — дополнительная проверка; отсутствия строк недостаточно |
| V03 PIN/policy faults | Разные/одинаковые/неверные PIN; missing/corrupt/stale policy, wrong key, cold start | Нет открытия Full; одинаковые PIN отвергаются; ошибка не удаляет Full данные | Unit/fault injection + Simulator; state transitions и file hashes | Не доказывает стойкость к offline brute force |
| V04 Revocation | Задержать RPC, write, download, extension callback; lock и сменить сессию | Старое поколение не пишет и не выдаёт контент; новый runtime ждёт завершения | Deterministic barriers, lifecycle trace без контента, physical iPhone для suspension | dispose/deinit без наблюдений недостаточно |
| V05 UI/API | S; search local/global, folders/archive, counters, forward/reply/mention, Saved Messages, shared media, public profile, direct deep link | Ровно согласованная таблица видимости D02, без bypass к hidden history | XCUITest + engine tests; sanitized screenshots/assertions | Пока D02 открыт, часть expected results не определена |
| V06 Ingress | S; updates/getHistory/getDialogs/search, contacts/recent peers, story/media prefetch, retries, edits/deletes | Запрещённые content/references не попали в store/cache и выдачу; cursor корректен | Hooks перед persistence и декодирование после; path coverage | Hook одного ingress не покрывает остальные |
| V07 Push/calls | S; normal/timeout/crash/no-run, badge/sound/attachments, delivered Full notifications, VoIP/CallKit | Согласованное D03 поведение при каждом fallback; отсутствие недопустимого следа | Подписанный iPhone, APNs test fixture, фактические entitlements; paired device отдельно | Simulator injection не доказывает APNs и entitlement |
| V08 Full recovery | Full → lock → часы Restricted → новые/edit/delete/expire → Full; отдельно secret chat/outgoing/read state | Нет ложного продвижения Full cursor, gaps восстановлены в принятых пределах D04/D05 | Test account harness, сравнение content+cursor по этапам, duplicate/gap assertions | Уже исчезнувшее на сервере не считается восстановленным по обещанию |
| V09 Policy transaction | Hide/unhide при активной extension; kill после каждого этапа staging/publish/cleanup; rollback generation | Нет смешанной policy; старый writer не может изменить новый store; доступ закрыт при неопределённости | Fault injection, file/key manifests; повторный старт | Системные уведомления не часть DB-транзакции |
| V10 Lifecycle | Full/Restricted; inactive/background, system lock, unlock, suspension, kill/relaunch, reboot, protected-data unavailable, scenes | PIN после системного lock, отсутствие визуального/файлового доступа до него | Physical iPhone + Simulator subset; event/state traces и cover snapshots | Callback protected-data не является единственным доказательством |
| V11 Key lifecycle | PIN change, key rotation, forgotten PIN, app upgrade, migration failure, logout/login, backup restore на этом/другом устройстве | Нет обхода Full PIN; data-loss только по принятому recovery contract; старые keys не открывают новые данные | Синтетические envelopes, KDF benchmark, device Keychain tests, manifests | Алгоритмы/параметры и policy recovery пока не выбраны |
| V12 External surfaces | Share/Siri/intents, Widget/Spotlight snapshots, Watch mirroring, CarPlay, exports, temp и logs | Никаких запрещённых данных в принятом scope; Full функции сохранены | Физические targets по наличию, OS index checks, sanitized log scan | Полная инвентаризация targets ещё нужна; внешние экспортированные копии отдельно |
| V13 Full regression | Baseline feature suite и те же сценарии после изменения: chats/search/media/calls/share/background/accounts | Нет отличий вне утверждённых исключений | Build/unit/integration/XCUITest/device; исходный и новый SHA | `testLaunch` проверяет только запуск, не полную регрессию |
| V14 Evidence review | Сверить все claims с SHA, кодом, командой, средой, тестом и ограничениями | Нет PASS без проверки, нет Accepted ADR без решения владельца | Независимое документальное review | Не заменяет практические V01–V13 |

## Обязательная матрица перед реализацией и приёмкой — предложение

- Каждая задача: затронутый target build, значимые unit/integration, diff review, обновлённая evidence matrix.
- Security/architecture: review плана, применимые V01–V12, отдельное согласование решения и интеграционного кандидата согласно принятому workflow.
- Simulator: build arm64 и smoke, UI/API сценарии и fault injection. При невозможности — явный BLOCKED соответствующего gate.
- Подписанный iPhone: PIN/Keychain/protected data, APNs/CallKit, suspension/reboot, backup. Требуется до заявлений о защите на устройстве.
- Минимальная iOS 13.0 и фактическая целевая версия: согласовать поддерживаемую матрицу; один Simulator iOS 26.5 не покрывает диапазон.

Текущие выполненные проверки перечислены только в 00_STATUS.md. Таблица V — план, а не протокол пройденных тестов.
