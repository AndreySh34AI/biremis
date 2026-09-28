# Архитектурное исследование Biremis — первый проход

Дата: 2026-09-27. **READY FOR SPIKES: задания экспериментов подготовлены. Production implementation BLOCKED.**
Этот статус не разрешает автоматически выполнять эксперименты. Требования DRAFT, ADR Proposed; владелец принял threat scope T1 в D01 (2026-09-28); остальные продуктовые решения открыты. Исследование не является подтверждением безопасности.

**Дополнение после первого прохода:** по отдельному поручению владельца выполнен live RPC опыт AUTH-01 на его сессии в Simulator. Hidden canary доступен через прямые getHistory/getPeerDialogs/peer search; новый контекст без login получает 401 AUTH_KEY_UNREGISTERED. Оба тестовых canary прочитаны. [Результаты, границы и отклонения](AUTH-01_RESULT.md). Это не полное закрытие V01 и не снятие production BLOCKED.

**Решение 2026-09-28:** [D01 принят](09_OPEN_DECISIONS.md): обязательна защита от штатных действий T1; извлечение контейнера, использование auth в обход штатных путей, изменение кода/debugger/memory dump исключены. ACR-001 разрешён выбором scope. AUTH-01 больше не блокирует online T1; D02–D09 и проверка штатных путей остаются открыты.

**Необязательное направление:** по следующему решению владельца добавлен [T2-01](08_TASKS.md) — исследование извлечённых копий, ключей и возможной схемы защиты. Задача подготовлена, эксперименты не запускались. T2 не становится условием выпуска T1; T3/T4 остаются исключены.

**EXT-01 выполнен (2026-09-28, CODE):** [отчёт](EXT-01_RESULT.md) содержит 45 source entries, шесть подтверждённых в baseline bundle extensions, 31 будущий тест и места проверки policy. Новые критичные пути: Notification Content direct MediaBox, Siri actions, Widget direct Postbox, Storage Usage/Downloads, sparse media calendar, system publishers и опциональный отдельный Watch. Runtime-проверки не выполнялись; D02/D03/D08 и release-покрытие остаются открыты.

## Исходное состояние

- Checkout: `/Users/andrey/Work/Telegram/Telegram-iOS`.
- Ветка: `restricted-mode`; commit: `42a84aec8abf72e54958ddf8ca9480514cc0d390` (`Перед архитектурным исследованием`).
- Origin: `https://github.com/AndreySh34AI/biremis.git`; отдельный upstream remote не настроен. Точка расхождения с upstream в этом проходе не устанавливалась, fetch/pull не выполнялись.
- Рабочее дерево перед исследованием чистое. Brief и swarm-документ уже входят в исследуемый commit. Прежний статус из диалога не переносился на новый checkout.
- В `restricted_mode/` не было файлов требований или предыдущих отчётов. `SPEC.md`, `THREAT_MODEL.md`, `SECURITY_INVARIANTS.md` отсутствуют.
- Применим корневой AGENTS.md; вложенных AGENTS в исследованных Telegram/submodules не найдено. Путь проекта в AGENTS отличается от фактического, использован текущий checkout.
- macOS 26.6.2 (25G83), arm64; Xcode 26.6 (17F113), developer dir `/Applications/Xcode.app/Contents/Developer`; Python 3.14.0; Bazel binary `build-input/bazel-8.4.2-darwin-arm64`.
- BUILD minimum iOS 13.0; API layer 228 (E25). Доступен Simulator runtime iOS 26.5, включая iPhone 17. Наличие физического тестового iPhone/provisioning не подтверждено.
- `local-config/development.json` не редактировался и его содержимое в отчёты не копировалось.

## Основные выводы

1. В текущем bootstrap аккаунты открываются без Full PIN как границы доступа (E01/E03/E12). AppLock — UI/lifecycle механизм, не готовый профильный key gate.
2. `.tempkey` содержит параметры DB; app PIN сериализуется строкой в metadata; auth keys имеют резервный путь через AccountBackupData → atomic-state (E02/E05–E07).
3. Шифрование Postbox не следует из наличия encryptionParameters: передаётся `forceEncryptionIfNoSet:false`. MediaBox отдельно пишет Data; forensic изоляция не доказана (E08).
4. Ограничение UI не ограничивает серверные полномочия Network. AUTH-01 подтвердил это для прямого RPC; D01 исключил такую возможность атакующего из обязательной T1 (E04/E16).
5. Updates, history holes, dialogs, search, recent peers, contacts, media и extensions имеют отдельные пути чтения/записи. Одного фильтра updates недостаточно (E09/E14–E19/E24).
6. Notification Service имеет fallback к initialContent; entitlement filtering условен в BUILD. Строгое отсутствие следа требует физического эксперимента, а не пустого текста (E09/E10).
7. Раздельные stores требуют отдельных согласованных cursors и протокола возврата Full. Сохранность исчезнувших серверных данных и secret chats пока не решена (E14/E24).
8. Spotlight/Widget/CallKit/Share/Siri создают отдельные поверхности; обнаружен прямой `print` имён в Spotlight и donation call intent. Logger redaction не закрывает их автоматически (E18–E23).

CODE-свидетельства не являются проверкой состояния реального аккаунта или взломом устройства. К личной истории, контейнерам и Keychain пользователя не обращались.

## Baseline: сборка

**Telegram для iOS Simulator arm64 успешно собран** на исследуемом commit.

Первый запуск штатного Make.py использовал обязательный `--overrideXcodeVersion`, подтвердил замену ожидаемого Xcode 26.2 на установленный 26.6. В песочнице Bazel не мог писать output base `/var/tmp`; повтор вне песочницы остановился на стадии анализа: отсутствует `@build_configuration//provisioning:Share.mobileprovision`, который требуется ShareExtension. Это ошибка provisioning/configuration, не Swift-компиляции.

В фактическом Make.py `invoke_build` поддерживает внутренний флаг disable_provisioning_profiles, но использованный build CLI его не устанавливает. Выполнен эквивалентный вызов уже подготовленного Bazel с существующим target flag `--//Telegram:disableProvisioningProfiles` для Simulator, без изменения signing configuration или исходников.

Команды из корня checkout:

```sh
python3 build-system/Make/Make.py \
  --overrideXcodeVersion \
  --cacheDir="$HOME/telegram-bazel-cache" \
  build --configurationPath=local-config/development.json \
  --xcodeManagedCodesigning --configuration=debug_sim_arm64 --buildNumber=1

build-input/bazel-8.4.2-darwin-arm64 build Telegram/Telegram \
  --//Telegram:disableProvisioningProfiles \
  --features=swift.use_global_module_cache --verbose_failures --remote_cache_async \
  --define=buildNumber=1 --disk_cache="$HOME/telegram-bazel-cache" \
  -c dbg --ios_multi_cpus=sim_arm64 --watchos_cpus=arm64_32 \
  '--@build_bazel_rules_swift//swift:copt="-j"' \
  '--@build_bazel_rules_swift//swift:copt="10"'
```

Результат второго пути: exit 0; `Build completed successfully, 4835 total actions`; 525.302 s. Артефакт: `bazel-bin/Telegram/Telegram.ipa`. Локальные диагностические логи: `/tmp/biremis-architecture-baseline.log` и `/tmp/telegram-build-simulator.log`; в Git не включены. Это не подтверждение device signing или APNs.

## Проверки запуска

Существующий `UITests.testLaunch` выбран отдельно: он запускает приложение с `--ui-test` и проверяет runningForeground. Регистрация/удаление аккаунтов и остальные сетевые UI-тесты не запускались. `--ui-test` очищает отдельный тестовый каталог при каждом запуске; для будущих kill/relaunch recovery tests этот механизм потребует адаптации, иначе он уничтожит исследуемое состояние (E01/E25).

```sh
xcodebuild test -project Telegram/Telegram.xcodeproj \
  -scheme iOSAppUITestSuite \
  -destination 'platform=iOS Simulator,name=iPhone 17,OS=26.5' \
  -only-testing:iOSAppUITestSuite/UITests/testLaunch \
  -resultBundlePath /tmp/biremis-architecture-launch-unrestricted.xcresult
```

Первый запуск в песочнице не смог подключиться к CoreSimulator и не нашёл destination. Повтор вне песочницы завершился exit 65 до выполнения теста. Первая значимая ошибка: `Building for device, but no provisioning_profile attribute was set` для `//Telegram:Telegram` при анализе generator target `Telegram_xcodeproj`. Xcode build phase `Generate Bazel Dependencies` вызывает generated Bazel target. Причина включения device обнаружена в `Telegram/BUILD:1829`: top_level_target Telegram явно задаёт `target_environments = ["device", "simulator"]`. `ProjectGeneration.py:39` переносит disableProvisioningProfiles в project Bazel flags. Поэтому один Simulator destination не исключает device analysis, а profile для device отсутствует. Это проблема конфигурации/генерации проекта и provisioning, не провал assertion testLaunch. Настройки не исправлялись в рамках запрета brief.

Лог: `/tmp/biremis-architecture-launch-unrestricted.log`; result bundle содержит сбой сборки, не пройденный тест. В Bazel UI runner также зафиксирован iOS 26.2, а доступен 26.5; Xcode destination выбран явно. Несоответствие runtime не выдаётся за причину обнаруженной ошибки device provisioning.

### Прямой smoke успешно выполнен

Создан отдельный чистый Simulator `Biremis-Architecture-Baseline`, iPhone 17 / iOS 26.5, UDID `D3F5B8EA-DCE8-4CD4-A658-792B7D1C43A9`. В него установлено приложение из успешно собранного IPA; существующие Simulator-контейнеры пользователя не использовались. `simctl launch` с `--ui-test` завершился exit 0, PID 60215. Через 87 секунд процесс оставался жив; снимок экрана просмотрен: экран приветствия Telegram с кнопкой Start Messaging. Авторизация и действия с аккаунтом не выполнялись.

Evidence: `/tmp/biremis-architecture-launch-smoke.json`, `/tmp/biremis-architecture-launch.png`. Это PASS только запуска/экрана приветствия, не XCUITest, не Full regression и не Restricted security. Тестовый Simulator после проверки остановлен, оставлен для воспроизведения; временный распакованный bundle находится в `/tmp/biremis-architecture-simulator-app/`.

Воспроизведение: создать отдельный iPhone 17 / iOS 26.5 через `simctl create`, boot/bootstatus, распаковать `bazel-bin/Telegram/Telegram.ipa`, `simctl install` его `Payload/Telegram.app`, затем `simctl launch <test-udid> <CFBundleIdentifier из собранного Info.plist> --ui-test`. Bundle ID не копируется из чувствительного development.json. Снять экран и проверить процесс; завершить Simulator. Запуск без `--ui-test` не входит в эту процедуру.

## Документы и следующий шаг

- [01_REQUIREMENTS_REVIEW.md](01_REQUIREMENTS_REVIEW.md): проект SPEC, semantic table и конфликты.
- [02_REPOSITORY_MAP.md](02_REPOSITORY_MAP.md): 25 свидетельств, process/store/key inventory, пробелы покрытия.
- [03_THREAT_MODEL_REVIEW.md](03_THREAT_MODEL_REVIEW.md): проект T1–T4 и первичные источники.
- [04_ARCHITECTURE_OPTIONS.md](04_ARCHITECTURE_OPTIONS.md): варианты A–D и критерии отказа.
- [05_TARGET_ARCHITECTURE.md](05_TARGET_ARCHITECTURE.md): условный проект и пять диаграмм.
- [06_SECURITY_COVERAGE.md](06_SECURITY_COVERAGE.md): draft I01–I10 и V01–V14.
- [07_IMPLEMENTATION_PLAN.md](07_IMPLEMENTATION_PLAN.md): G0–G5 и gates.
- [08_TASKS.md](08_TASKS.md): задания исследователю/программисту/reviewer/интегратору.
- [09_OPEN_DECISIONS.md](09_OPEN_DECISIONS.md): D01–D09 и ACR-001–003.
- [ADR-001](adr/ADR-001-authority-before-storage.md), [ADR-002](adr/ADR-002-session-and-key-boundary.md), [ADR-003](adr/ADR-003-external-surfaces-gate.md): Proposed, не Accepted.

Следующий конкретный шаг — принять D02/D03/D08 по вопросам EXT-01_RESULT, затем проверить NOTIFY-01 и уточнить storage/state/lifecycle проект. Карта штатных путей и тестов подготовлена; независимое REVIEW-01 и runtime-покрытие ещё требуются. Автоматический запуск нескольких агентов не разрешён brief.

## Ограничения

Не выполнен полный набор V01–V14; live RPC часть V01 отражена в AUTH-01_RESULT; нет device/forensic/Keychain/APNs/backup/sync evidence; список UI consumers и ingress ещё не исчерпывающий. Новая криптосхема и политика secret chats не выбраны. Без оставшихся решений D02–D09 и закрытия G1 нельзя объявлять READY FOR IMPLEMENTATION. Изменения этого прохода ограничены отчётами в architecture; production code, требования, зависимости, entitlements и пользовательские конфигурации не менялись; commit/push не выполнялись.

Итоговая проверка артефактов: 13 Markdown-файлов, существующие локальные ссылки, согласованная ширина таблиц, закрытые code fences и отсутствие whitespace errors. Tracked diff пуст; новые файлы только в `restricted_mode/architecture/`. Проверка документов выполнена автором исследования, независимое REVIEW-01 ещё не проводилось.

## Зафиксированные submodules

`git submodule status` на начало исследования: все перечисленные checkout соответствуют gitlink (без `+`, `-` или `U`). Рекурсивный аудит вложенных зависимостей не выполнялся.

```text
 a99414ad848c3aeb84640934352ecc85d8a937f5 build-system/bazel-rules/apple_support (2.5.2)
 1791d916de4083388f22e20248d8b010d23f0d6b build-system/bazel-rules/rules_apple (4.5.2-1-g1791d916)
 9dce728ed1e9168ec8c912fcd3443dad48a286fe build-system/bazel-rules/rules_swift (3.4.1-19-g9dce728e)
 997f2db058596f91663e54782b79490de87208da build-system/bazel-rules/rules_xcodeproj (4.0.0)
 feea27cfc88eccc58af0cfe5674444e945cfb75f build-system/bazel-rules/sourcekit-bazel-bsp (0.7.1-aspect-9320898-10-gfeea27c)
 4a3144b5d527429f7bbd0f07003cb372bf8939ce submodules/LottieCpp/lottiecpp (heads/main)
 e3069322a3d1e16ecb11a5e302242e59ddd7f09e submodules/TgVoipWebrtc/tgcalls (ios-release-11.13-36-ge306932)
 67f103bc8b625f2a4a9e94f1d8c7bd84c5a08d1d submodules/rlottie/rlottie (heads/master)
 53cb43cb66908a28812d7629d03fed94c9827a24 third-party/XcodeGen (2.43.0-2-g53cb43cb)
 330e20672e85f9de1678dccd6957845898ef57a1 third-party/dav1d/dav1d (1.5.0-23-g330e206)
 e7bfd8b6c230a6824e7fd1efa2378a7322986128 third-party/libvpx/libvpx (v1.13.0-435-ge7bfd8b6c)
 e894536b2f46caad93f997448d2daff9431b19dd third-party/td/td (v1.8.0-7675-ge894536b2)
 3817e906cb6c22ec9cc62023b073e1a668d9cb33 third-party/webrtc/webrtc (heads/M123-9-g3817e906cb)
```
