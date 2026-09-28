# EXT-01 — штатные пути доступа, фоновые операции и внешние поверхности

Дата: 2026-09-28. Base SHA: `87a048874df13b1f865f7ddfa1855b04e7367692`.
**Статус: документальная задача выполнена; карта ниже — CODE, безопасность и runtime-покрытие не подтверждены.** Архитектура остаётся Proposed. Нерешённые D02/D03/D05/D06/D08 явно привязаны к сценариям; они не мешали исследовать код.

## Вывод

Ограничения нужно применять на нескольких путях: штатный запрос/действие, обработка результата до сохранения, выдача из локального состояния, публикация в системные поверхности и смена режима. Одного фильтра списка чатов или одного запрета открытия ChatController недостаточно.

Самые важные обнаруженные пути:

1. **Notification Content читает изображения из MediaBox напрямую**, по данным уведомления, и может завершить показ без запроса истории. Это отдельное расширение от Notification Service (X34).
2. **Siri выполняет действия:** отправка, получение сообщений по ID/непрочитанных, изменение read state. Проверка старого `isAppLocked` не выражает разрешения Restricted. getMessages читает Postbox по ID; unreadMessages использует root chat list и history views для unmuted CloudUser; missedCalls использует callListView и формирует INCallRecord (X29/X43).
3. **Widget читает peers, unread count и последнее сообщение через accountTransaction**, а список кандидатов для настройки виджета строится отдельно в IntentsExtension (X29–X31).
4. **Storage Usage и Downloads — дополнительные средства просмотра сообщений.** У Storage Usage собственные выборка, preview и навигация; фильтр основного chat list не является контролем этих путей (X23–X25).
5. **Shared media/calendar имеет отдельные RPC и записи:** `getSearchResultsCalendar` обновляет peers и добавляет сообщения; это не только основной history hole (X16/X17).
6. **Уведомление может инициировать отправку без открытия чата.** Reply callback выбирает account, меняет read state и вызывает enqueueMessages (X03/X41).
7. **Spotlight, CallKit и Now Playing сохраняют/публикуют данные вне chat list.** Также найдены прямые `print` в Spotlight и CallKit, обходящие Logger redaction (X32/X35/X40).
8. **Опциональный Watch-клиент имеет собственную TDLib-сессию и хранилище.** Изменение iPhone policy само по себе не является политикой этого клиента (X01/X37).

Это свойства исследованного baseline, в котором Restricted ещё не реализован. Их нельзя выдавать за успешно воспроизведённые обходы готового режима.

## Scope и метод

- Обязательная модель — T1 по D01. Разрешённые пользовательские действия включают поиск, ссылки, системные intents, настройки, share/export, повторный запуск и переключение режимов.
- T2 — отдельное необязательное исследование; T3/T4 не являются критериями этой задачи. Извлечение памяти/контейнера, debugger и прямые live RPC не выполнялись.
- Изучены текущие исходники, BUILD/plist и метаданные ранее собранного чистого baseline bundle. Личный Simulator/account/container не открывался. Сообщения, ключи и local-config не читались.
- Использованы `rg`, чтение участков source, проверка связей вызовов и manifest с SHA-256 файлов. Production source/BUILD не менялись, targets и функции не отключались.
- X01–X45 ниже — новые якоря. E01–E25 из [карты первого прохода](02_REPOSITORY_MAP.md) дополняют bootstrap/storage/state/key inventory; они не подменяют результаты EXT-01.
- Ни отсутствие символа в поиске, ни наличие API/target в исходнике не считается доказательством поведения подписанного приложения.

## Состав приложения и условия активации

В `Telegram/BUILD` default `disableExtensions=false`; `Telegram.extensions` содержит шесть targets. `embedWatchApp=false` по умолчанию, Watch добавляется отдельно. Условия подписи не обходились (X01).

| Процесс / target | Подтверждение сборочной конфигурации и trigger | Данные / результат | Что ещё требуется |
|---|---|---|---|
| Main Telegram | Основной ios_application; запуск, URL/userActivity/notification/shortcut | Account/Postbox/MediaBox, UI, badges и system publishers | Все режимы и account switching, V03–V06/V09–V13 |
| ShareExtension | `com.apple.share-services`; отправка из другого приложения | Общий root, metadata, выбор account/recipient, sentShareItems, import/story staging | Policy при bootstrap, выборе и отправке; XT20 |
| NotificationServiceExtensionv1 | `com.apple.usernotifications.service`; подходящий push | Standalone state manager, payload, poll/update, текст/media/badge | Реальные push/подпись/timeout/no-run; NOTIFY-01 |
| NotificationContentExtension | `com.apple.usernotifications.content-extension`; категории withReplyMedia/withMuteMedia | Payload → media file → preview; при необходимости fetch | Фактическая доставка указанных категорий не проверена; XT22 |
| IntentsExtension | `com.apple.intents-service`; Send/Search/SetAttribute/Call/SearchCallHistory и SelectFriends intents в plist | Supplementary account, сообщения/контакты, действия и widget picker | Handler dispatch, старые identifiers, системная доступность; XT23/XT24 |
| WidgetExtension | `com.apple.widgetkit-extension`; configured friends + timeline | accountTransaction → title/avatar/badge/top message | Старые timeline/snapshot и очистка после смены mode; XT25 |
| BroadcastUploadExtension | `com.apple.broadcast-services-upload`; ReplayKit broadcastStarted | App Group coordination, video/audio buffers; IPC или embedded branch по coordination file | Прекращение старой трансляции/доступа при смене режима; XT29 |
| TelegramWatchApp | Prebuilt watch target включается embedWatchApp; source Telegram/WatchApp | Собственные TDLib login, DB/files в Application Support/tdlib/account UUID | D08: границы продукта и тест на Watch; XT30 |

**Артефакт сборки:** в `/tmp/biremis-architecture-simulator-app/Payload/Telegram.app` найдены все шесть `.appex`; Watch отсутствует. У Widget MinimumOSVersion=14.0, у остальных пяти=13.0. Это метаданные предыдущей baseline-сборки, не новая сборка EXT-01 и не проверка installed приложения/подписи. [bundle_inventory.json](evidence/ext01/bundle_inventory.json).

**Capabilities и смежные поверхности:**

- Siri entitlement зависит от `telegram_enable_siri`. Массивы IntentsRestrictedWhileLocked/WhileProtectedDataUnavailable в generated plist пустые; делать вывод о запрете доступа только по plist нельзя. В handlers есть собственные lock checks (X01/X29).
- CarPlay messaging entitlement в BUILD добавляется для official bundle IDs; реальная подпись форка не проверена. В main коде явно есть `.allowInCarPlay` и `.allowAnnouncement` у notification categories (X01/X03). Отсутствие отдельного CarPlay UI target не убирает уведомления/объявления из аудита.
- В ограниченном поиске Swift/ObjC/header/plist/BUILD под Telegram и submodules не найдены `import ActivityKit`, `ActivityConfiguration`, `AppShortcutsProvider`, соответствие `: AppIntent`, `CPTemplateApplicationScene`, `import CarPlay`, `NSSupportsLiveActivities`. Это **не** доказательство отсутствия этих функций в generated/remote code или на устройстве. [Запросы и результаты](evidence/ext01/surface_search.json). Legacy Siri INIntent реализован и входит в scope независимо от этих отрицательных результатов.
- `Telegram/Watch/` с legacy исходниками не назван watch_application текущего target: BUILD ссылается на prebuilt tgwatch/WatchApp. В AppDelegate wakeup wiring указано `watchTasks: .single(nil)`. Это не доказательство поведения системного зеркалирования уведомлений; проверка mirroring остаётся D08/NOTIFY-01.

## Предлагаемые места проверки policy

Это обязанности, а не утверждённые новые классы. Переиспользовать существующие engine/transaction/lifecycle механизмы после проверки их применимости.

| Метка | Обязанность |
|---|---|
| B0 — вход в сессию | Установить account, mode, валидную policy и её поколение до открытия штатного пути; определить Locked/error поведение. Повторить проверку при смене account |
| B1 — выдача и переход | Проверять peer/message/thread/story/resource и связанные entities до результата, preview и навигации. Применять к локальным и сетевым ответам, включая totals, pagination и cached results |
| B2 — запись и ingress | Проверять response/update до Postbox/MediaBox/item-cache/temp commit, учитывая I02, связанные peers/replies/stories и sync state. Cursor нельзя продвигать произвольно ради фильтрации |
| B3 — действие | Проверять источник и адресата непосредственно перед send/forward/read/export/call, в том числе после асинхронного выбора и после смены policy |
| B4 — внешний вывод | Применять ограничения перед OS notification/widget/Spotlight/CallKit/Now Playing, очистить или заменить старые результаты в согласованном scope |
| B5 — смена режима | Отозвать старые callbacks/subscriptions/downloads/player/broadcast; закрыть preview/cover; новая сессия не наследует старые разрешения |

Ресурсный ID без связи с account/сообщением недостаточен для решения о выдаче. Shared media, forwards и сохранённые копии могут связывать один ресурс с разными контекстами: правила определяются D02, а не только peerId текущего экрана.

## Матрица путь → policy → тест

Все XT ниже — **спецификации будущих тестов, NOT RUN**. Для каждого сценария нужны Full baseline (разрешённый canary доступен), Restricted (запрещённый canary не выдан, разрешённый доступен) и повтор после смены policy/Locked/cold start. Тестовый код может использовать hooks для проверки записи; это инструмент разработчика, а не добавление T3/T4 к модели атакующего.

| Путь / источники | Поток и возможное раскрытие / действие | Места проверки | Проверяемый сценарий и зависимость |
|---|---|---|---|
| XT01 Chat list, folder, archive, pinned, badge — X11–X13, E15 | getDialogs/getPinnedDialogs → Postbox → list/group reference, unread и app badge; скрытый peer может влиять на aggregate | B1/B2; вычислять счётчики по согласованной policy | Hidden archived/pinned/muted/unread; folder preview/названия/total не раскрывают запрещённое. V05/V06; D02 |
| XT02 History, holes, threads, scheduled — X08/X10, E14 | ViewTracker → history/hole RPC → сообщения и связанные записи; переход в thread/по ID обходит главный список | B1 до загрузки/выдачи, B2 до commit | Открыть hidden по известной ссылке/ID, тему и старый экран; delayed response после переключения. V04/V05/V06; D02/D05 |
| XT03 Search local/global/hashtag, pagination — X05–X07 | UI объединяет local, remote, global posts, hashtag и totals; поиск может вернуть данные без вставки в обычную историю | B1 на query/result/next-page/count; B2 для записываемых peers/messages | Найти canary по тексту/имени/username, следующая страница и cached result после hide. V05/V06; D02 |
| XT04 Saved Messages, авторы/пересылки — X05/X07/X14 | Отдельный поиск Saved peers, channel message author lookup, forward source metadata | B1 для source/author/reference, B3 для действий | Сохранённая копия hidden и переход к источнику; заранее определить судьбу копии и имени. V05; D02 |
| XT05 Recents/top peers/inline bots — X19/X20 | contacts.getTopPeers → peers/item cache; recently searched → ordered list → предложения | B2 до cache, B1 перед предложениями, B5 для старых recents | Hidden присутствовал в recent/top до переключения; пустой поиск и выбор получателя не раскрывают его вопреки D02. V05/V06 |
| XT06 Контакты, общие группы, профиль — X07/X18/X21 | Import/search/contacts view; getCommonChats обновляет peers; профиль и общий чат открывают альтернативную навигацию | B1 на identities, common count и переход; B2 на import/cache | Контакт скрытого чата виден в общей группе/профиле: проверить принятую семантику, запрещённая история не открывается. V05/V06; D02 |
| XT07 URL/userActivity/shortcut/account switch — X02/X04 | openUrl/openChatWhenReady могут переключить account; resolved channel/reply/story ведут к отдельным экранам | B0 после выбора account; B1 до resolve/download/preview и открытия | tg/universal/private message link, Spotlight contact action, shortcut при холодном старте/смене account. V03/V05/V10/V12; D05/D08 |
| XT08 Reply/forward/mention/navigation — X14/X15 | Message group → forward picker → enqueue; navigateToMessage может открыть другой peer/thread | B1 для связанного контента, B3 для источника/цели, B5 для открытого picker | Reply quote/forward автора hidden; выбрать получателя, сменить policy и отправить. V04/V05/V06; D02 |
| XT09 Shared media/calendar — X16/X17 | sparse list positions/calendar → RPC → peers/messages; собственный grid/cache и openMessage | B1 на sparse выдаче/count/calendar/gallery, B2 до addMessages | Фото hidden открывалось в Full; поиск по дате, медиа-панель и gallery после Restricted. V05/V06; D02 |
| XT10 Downloads и thumbnails — X23, E24 | fetchedMediaResource с background continuation → MediaBox + RecentDownloadItem(messageId/resourceId) → Downloads UI | B2 до файла/recents, B1 на ресурсе/списке, B5 | Начать download в Full, переключить режим, открыть Downloads/thumbnail и retry. V04/V06/V12 |
| XT11 Storage Usage — X24/X25 | collect stats → render messages/peers → preview → navigateToChat; cached existingMessages — отдельная ветка | B1 на статистике/именах/preview и обеих ветках render; B3 на действиях | Hidden занимает cache: открыть список peer/file/media и старый preview. Общие размеры — D02; запрещённая история не выдаётся. V05/V12 |
| XT12 Stories — X04/X09/X22 | URL/subscriptions/updates/getStoriesByID → story table + peers + media → viewer | B1/B2 для story/reference/viewer; B3 для seen/action | Story hidden peer/репост, prefetch и открытая story после смены режима. V04/V05/V06; D02 |
| XT13 Основной updates replay — X09, E14 | Разные message/story/entity updates → транзакции и состояние | B2 до всех применимых writes; B5 generation; сохранить согласованный state | Новые/edit/delete/expire + reconnect во время Restricted; canary не попадает в запрещённый store/output. V06/V08/V09; D04/D05 |
| XT14 Send/read/background queues — X03/X29/X41, E13 | UI/intent/notification → enqueue/read state → managed outgoing/sync | B3 перед постановкой и выполнением; B5 при delayed callbacks | Старая pending отправка/mark-read после hide/lock не выполняется с устаревшим разрешением. V04/V06/V09; D06 |
| XT15 Export/Photos/pasteboard — X26/X27 | Message group/MediaBox → fetch/temp → PHPhotoLibrary, pasteboard или ShareController | B1 для исходного объекта, B3 перед экспортом; B5 для незавершённой загрузки | Export hidden из старого gallery/share sheet; delayed save после смены режима. V04/V12; судьба уже экспортированных копий — D08 |
| XT16 App switcher/cover/relaunch — X02/X39, E22 | UIKit lifecycle → covering snapshot/lock; background account может продолжать работу | B0/B5, отдельно B4 для системного snapshot | Full preview → background/system lock → Restricted; kill/relaunch, ошибочная policy, возврат к старой навигации. V03/V04/V09/V10; D06 |
| XT17 Now Playing / audio controls — X40, SharedMediaPlayer | Для music state MediaManager публикует metadata/artwork в MPNowPlayingInfoCenter и регистрирует remote commands | B1/B4 для metadata/playlist, B5 для playback/next/prefetch | Full трек → Restricted, Control Center/remote next; старые title/artwork/audio не нарушают scope. V04/V12; D06/D08. Условия voice/video требуют отдельного runtime теста |
| XT18 Settings и сохранённые исключения — X01/X12/X24, E20 | Folder/include/exclude lists, notification exceptions, cache и account selectors могут показывать identities | B1 для списков/preview; B3 для изменений | Settings/export/debug/account picker после Restricted не раскрывают запрещённые identities; точные исключения D02/D05. V05/V12 |
| XT19 Debug/log export — X32/X35/X38/X44/X45 | TabBar при 10 быстрых taps вызывает debug action Settings → debugController; прямой print имени Spotlight/UUID CallKit; debug UI собирает логи main/share/notification и может экспортировать | Контроль записи; B1/B3 при штатном доступе к диагностике | Canary в тестовом log, открытый debug/export путь: проверять доступность в конкретной сборке и policy. V12/V14; console извне сама по себе T4, экспорт через UI — T1 |
| XT20 Share Extension / history import — X28, E18 | Собственный bootstrap/passcode; selected peers → sentShareItems; import/story staging отдельно | B0 на запуск/смену account, B1 на picker, B3 на send/import, B2 temp | Открыть Share из другого приложения, выбрать hidden/старый recipient, import/history/story, переключить mode до отправки. V03/V04/V06/V12; D02/D05 |
| XT21 Push / Notification Service — X33, E09/E10 | Payload → standalone polling/state/attachments → notification; fallback initialContent | B0/B2 перед poll/write; B4 перед content/badge/sound/fallback | Full-delivered + новый hidden push, success/timeout/crash/no-run, reaction/story/VoIP. V07; D03 и фактическая подпись обязательны |
| XT22 Notification Content — X34 | Long press/category → userInfo account/peer/message/media → прямой Data(MediaBox), затем fetch при отсутствии файла | B0/B1 **до** прямого чтения и preview; B2 для fetch; B5 для старого content | Уведомление создано в Full, затем Restricted; открыть large/thumbnail/video, cached и uncached. V07/V12; D03/D06 |
| XT23 Siri messaging/calls/history — X29/X43 | Handler dispatch → lock-file check → account; send, getMessages/unread, mark-read, call continuation/history | B0 плюс B1 на resolve/search, B3 на send/read/call; не полагаться на один lock Bool | Старые IN identifiers, missing/corrupt policy, Siri search/send/read/call во всех режимах. V03/V05/V12; D02/D03/D08 |
| XT24 Widget configuration picker — X29 | SelectFriends → allAccounts → transaction.searchPeers/getTopChatListEntries → Friend identity/avatar | B0/B1 на каждый account и result, B5 для старых выбранных IDs | Добавить widget в Full; менять настройки в Restricted; поиск hidden/переключение account. V05/V12; D02/D05 |
| XT25 Widget timeline — X30/X31 | Configured account:peer → read-only accountTransaction → peer/unread/topMessage/avatar path → timeline | B0/B1 на provider; B4/B5 на публикацию и устаревший timeline | Существующий widget после hide/lock; latest message, badge/avatar и tap-to-open согласованы. V05/V12; D02/D06/D08 |
| XT26 Spotlight / suggested identities — X32/X42, X02 | Contacts+recent apps → data.json/avatar + CSSearchableIndex; result launches chat; удаления async | B1 на feed; B4 при index/remove; B0/B1 на userActivity | Full indexed hidden → Restricted → системный поиск/старый result; delayed index callback. V04/V12; D02/D08 |
| XT27 CallKit / VoIP / intent donation — X35, E21 | Peer identity → localizedCallerName/handle → system call UI; startCall → INInteraction.donate | B3 до call/action; B4 перед report/donation; B5 lifetime | Hidden входящий/исходящий/недавний вызов, старый donated intent и переход режима во время звонка. V07/V12; D03/D06/D08 |
| XT28 CarPlay/announcements — X01/X03/X29 | Notification categories allowInCarPlay/allowAnnouncement + legacy messaging/call intents | B4 на публикации; B1/B3 на intent; подпись отдельно | Проверить фактическое разрешение, hidden notification/read/reply/announcement на поддержанном устройстве. V07/V12; D03/D08 |
| XT29 Broadcast — X36 | ReplayKit buffers → shared coordination → IPC/embedded call transport; приложение может быть background | B0/B3 для сессии трансляции, B5 для buffers/transport; согласовать Full continuity | Начать broadcast в Full, Restricted/lock/crash host; проверить оба implementation type. V04/V12; D06/D08 |
| XT30 Watch — X01/X37 | Отдельный login/client → DB/files → chats/media; опционально embedded target | Нужен собственный согласованный контракт либо явная граница D08; iPhone policy не считается достаточной | Отдельно Watch-app navigation и OS notification mirroring; iPhone Full→Restricted. V07/V12; D08 |
| XT31 Live Activities / современные App Intents — bounded search | В просмотренном scope реализации не обнаружены; отсутствие в runtime не доказано | При появлении target/API — B0/B1/B3/B4/B5 по функции | Проверить generated sources и release bundle/entitlements; при наличии добавить конкретный тест. V12/V14; D08 |

## Почему существующие механизмы нельзя принять за готовую защиту

- `isAppLocked` отвечает на другой вопрос, чем «разрешён ли этот peer/message в текущем режиме». В Siri проверка отказа срабатывает при успешно прочитанном/декодированном LockState с locked=true; будущая missing/corrupt Restricted policy должна иметь явно безопасный fallback, независимо от этого baseline поведения.
- `isReadOnly` у widget/accountTransaction не запрещает чтение скрытого сообщения. Folder filters, excludeDisabled и `messageIsHidden` для self-expiring/cache logic не являются Restricted policy.
- UI-флаги captureProtected/hiddenMedia, правила запрета пересылки и настройки скрытия push previews выполняют свои upstream задачи. Их нельзя объявлять общей границей профилей.
- Dispose/cancel/reload/deleteSearchableItems — полезные механизмы повторного использования, но асинхронность требует проверки результата и поколения. Request на reload widget сам по себе не доказывает немедленное исчезновение старого snapshot.
- Notification Content имеет прямой файловый fast path; запрет только сетевого запроса или основного Postbox read его не охватывает.
- Полное удаление уже экспортированных Photos/Files, скриншотов, системных копий и данных другого авторизованного устройства не следует из переключения режима. D08 должен определить поддержанные поверхности и границы обещаний.

## Решения, необходимые для expected results

Следующие вопросы уточняют D02/D03, но **не считаются ответами владельца**:

| Решение | Конкретный выбор, который требуется зафиксировать | Затронутые тесты |
|---|---|---|
| D02-a | Скрывается только история разговора или также identity/profile/контакт человека? Допустимо ли увидеть его в общей группе? | XT03–XT08, XT18, XT23–XT26 |
| D02-b | Что делать с копией/цитатой/пересылкой hidden сообщения в разрешённом чате или Saved Messages: текст, автор, ссылка, превью? | XT04/XT08/XT09/XT15 |
| D02-c | Для скрытого публичного канала запрещено повторное открытие через global search/username/link? Можно ли начать новый разговор с тем же peer? | XT03/XT06/XT07/XT14/XT20 |
| D02-d | Как учитывать shared group/topic/story, mentions/replies и общие media ресурсы, связанные с hidden peer? | XT02/XT06/XT08/XT09/XT12 |
| D02-e | Запрещены ли следы в folder names/counts, unread/mentions, общих размерах cache и списке account? | XT01/XT05/XT11/XT18/XT24/XT25 |
| D03-a | Требуется отсутствие самого события hidden push (badge/sound/attachment) или допустим обезличенный сигнал? | XT21/XT22/XT25/XT28/XT30 |
| D03-b | Что требуется от уже доставленных Full notifications и предпросмотров после входа в Restricted? | XT21/XT22/XT26 |
| D03-c | Как обрабатывать hidden VoIP/CallKit и fallback timeout/no-run с текущей подписью? | XT21/XT27/XT28 |
| D06/D08 | Как поступать с активным звонком, плеером, broadcast, system snapshot и Watch при переключении? Какие поверхности входят в обещания продукта? | XT16/XT17/XT25–XT31 |

Ограничение личности/копий не выбирать автоматически по наличию peer ID в blacklist. Пример обязательного свойства независимо от этих решений: путь, которому запрещён доступ к hidden history, не должен возвращать её при повторном открытии по deep link или из старого widget.

## Тестовая fixture и порядок проверки

Выделенный тестовый аккаунт; visible и hidden cloud peers, общий group/topic, Saved Messages, тестовый контакт/публичный peer при необходимости. Разные canary в тексте, имени, filename, thumbnail, story; синтетические forwarded/reply копии. Сохранить Full baseline и настроить test widgets/Spotlight/notifications **до** переключения, чтобы проверять старое состояние.

1. После решения D02 — engine/UI тесты XT01–XT15/XT18; запрос, локальный результат, commit и действие проверяются отдельно. Учитывать totals и pagination, а не только наличие строки на экране.
2. LIFE-01 — задержанные ответы, старый выбранный получатель, скачивание/экспорт/плеер/трансляция, concurrent extension, смена policy и account. Проверить V03/V04/V09/V10.
3. На подписанном iPhone — NOTIFY-01 и XT21–XT28; paired Watch/CarPlay только в принятом D08. Simulator не заменяет проверку APNs, подписи и системных кэшей.
4. Full regression до/после: видимые данные и разрешённые действия работают; Full сохраняет upstream функции вне согласованных исключений. Запрет функции ради прохождения теста не является исправлением.
5. Evidence хранит case/state/generation, Bool совпадения canary, тип результата, SHA и среду. Не сохранять личные данные, auth keys, реальные PIN или raw API objects.

## Что выполнено и что осталось

**Выполнено:** inventory bundled targets и условий; просмотр цепочек входа, чтения/записи и внешнего вывода для перечисленных семейств; карта B0–B5; 31 спецификация проверки; привязка решений; source manifest и inspection предыдущего baseline bundle. Проверены ссылки документации, существование/строки/хеши источников и JSON evidence. Сборка приложения не запускалась: source/BUILD не менялись.

**Не выполнено:** runtime тесты XT01–XT31; подпись/entitlements/APNs/paired devices; исчерпывающий анализ каждой ветви всех consumers, всех Telegram API и сторонних/generated зависимостей; независимое review. Представленная карта не является доказательством полного покрытия безопасности. Автоматический поиск не доказывает отсутствие новых/динамических entrypoints.

**Оставшиеся технические проверки:** runtime Siri resolution и legacy OS dispatch; runtime debug gesture в release-подписи (цепочка исходников X44/X45 установлена); secret-chat/outgoing/ack semantics (D05/SYNC-01); resource-to-message mapping и многократные ссылки; media prefetch/thumbnail variants, bots/web apps/payments/inline content, полный набор reference/update constructors. Эти ветви должны войти в декомпозицию реализации и coverage review до protected release; их не объявлять PASS на основании этой карты.

**Следующий этап:** принять D02/D03/D08, затем проверить NOTIFY-01 и уточнить storage/state/lifecycle проект по карте. Независимое REVIEW-01 должно проверить source snapshot и полноту consumer inventory. T2-01 остаётся отдельным необязательным исследованием и не задерживает T1 сам по себе. EXT-01 не разрешает production implementation и не переводит ADR в Accepted.

## Реестр исходников X01–X45

Хеши файлов и точные поисковые якоря: [source_manifest.json](evidence/ext01/source_manifest.json). Пути относительны корню; номера строк привязаны к base SHA. Каждая запись означает чтение соответствующего участка, а не доказанное покрытие всех ветвей файла.

| ID | Исходник и строки |
|---|---|
| X01 | `Telegram/BUILD` — 73, 105, 491, 1268, 1749, 1798 |
| X02 | `submodules/TelegramUI/Sources/AppDelegate.swift` — 2546, 2662, 2787, 2811 |
| X03 | `submodules/TelegramUI/Sources/AppDelegate.swift` — 2840, 2859, 2889, 2989 |
| X04 | `submodules/TelegramUI/Sources/OpenResolvedUrl.swift` — 64, 341, 343, 1491 |
| X05 | `submodules/ChatListUI/Sources/ChatListSearchListPaneNode.swift` — 1995, 2246, 2765, 2853 |
| X06 | `submodules/TelegramCore/Sources/TelegramEngine/Messages/SearchMessages.swift` — 80, 308, 684, 774 |
| X07 | `submodules/TelegramCore/Sources/TelegramEngine/Peers/SearchPeers.swift` — 30, 70, 185, 191 |
| X08 | `submodules/TelegramUI/Sources/ChatHistoryViewForLocation.swift` — 39, 249 |
| X09 | `submodules/TelegramCore/Sources/State/AccountStateManagementUtils.swift` — 3910, 4260, 5440 |
| X10 | `submodules/TelegramCore/Sources/State/Holes.swift` — 423, 1179 |
| X11 | `submodules/TelegramCore/Sources/State/FetchChatList.swift` — 273, 289 |
| X12 | `submodules/ChatListUI/Sources/Node/ChatListNode.swift` — 732, 743 |
| X13 | `submodules/TelegramUI/Sources/ApplicationContext.swift` — 139, 140 |
| X14 | `submodules/TelegramUI/Sources/ChatControllerForwardMessages.swift` — 18, 55, 217 |
| X15 | `submodules/TelegramUI/Sources/Chat/ChatControllerNavigateToMessage.swift` — 27, 162 |
| X16 | `submodules/TelegramUI/Components/PeerInfo/PeerInfoVisualMediaPaneNode/Sources/PeerInfoVisualMediaPaneNode.swift` — 1266, 1270 |
| X17 | `submodules/TelegramCore/Sources/TelegramEngine/Messages/SparseMessageList.swift` — 122, 876, 901 |
| X18 | `submodules/TelegramCore/Sources/TelegramEngine/Peers/GroupsInCommon.swift` — 59, 101, 127 |
| X19 | `submodules/TelegramCore/Sources/TelegramEngine/Peers/RecentPeers.swift` — 51, 64, 77 |
| X20 | `submodules/TelegramCore/Sources/TelegramEngine/Peers/RecentlySearchedPeerIds.swift` — 6, 50 |
| X21 | `submodules/TelegramCore/Sources/State/ContactSyncManager.swift` — 322, 328, 346 |
| X22 | `submodules/TelegramCore/Sources/TelegramEngine/Messages/Stories.swift` — 2457, 2466, 1922 |
| X23 | `submodules/FetchManagerImpl/Sources/FetchManagerImpl.swift` — 263, 266, 775 |
| X24 | `submodules/TelegramUI/Components/StorageUsageScreen/Sources/StorageUsageScreen.swift` — 2338, 2392, 2532, 2762 |
| X25 | `submodules/TelegramCore/Sources/TelegramEngine/Resources/CollectCacheUsageStats.swift` — 216, 271, 249 |
| X26 | `submodules/SaveToCameraRoll/Sources/SaveToCameraRoll.swift` — 17, 100, 167, 209 |
| X27 | `submodules/TelegramUI/Sources/ChatControllerOpenMessageShareMenu.swift` — 94, 188 |
| X28 | `submodules/TelegramUI/Components/ShareExtensionContext/Sources/ShareExtensionContext.swift` — 553, 921, 738 |
| X29 | `Telegram/SiriIntents/IntentHandler.swift` — 58, 462, 522, 593, 736, 952 |
| X30 | `Telegram/WidgetKitWidget/TodayViewController.swift` — 68, 132, 174, 219 |
| X31 | `submodules/TelegramUI/Sources/WidgetDataContext.swift` — 92, 202, 258, 289 |
| X32 | `submodules/TelegramUI/Sources/SpotlightContacts.swift` — 124, 162, 184, 102 |
| X33 | `Telegram/NotificationService/Sources/NotificationService.swift` — 902, 2553, 2596, 2585 |
| X34 | `submodules/TelegramUI/Sources/NotificationContentContext.swift` — 148, 192, 194, 245 |
| X35 | `submodules/TelegramCallsUI/Sources/CallKitIntegration.swift` — 68, 91, 235, 273 |
| X36 | `Telegram/BroadcastUpload/BroadcastUploadExtension.swift` — 37, 129, 317, 366 |
| X37 | `Telegram/WatchApp/tgwatch Watch App/TDClient.swift` — 373, 384, 431, 121 |
| X38 | `submodules/DebugSettingsUI/Sources/DebugController.swift` — 314, 480 |
| X39 | `submodules/AppLock/Sources/AppLock.swift` — 238, 306 |
| X40 | `submodules/TelegramUI/Sources/MediaManager.swift` — 278, 293, 298 |
| X41 | `submodules/TelegramCore/Sources/PendingMessages/EnqueueMessage.swift` — 572, 680 |
| X42 | `submodules/TelegramUI/Sources/SharedAccountContext.swift` — 1064, 1075 |
| X43 | `Telegram/SiriIntents/IntentMessages.swift` — 20, 33, 104, 165 |
| X44 | `submodules/TelegramUI/Sources/TelegramRootController.swift` — 233 |
| X45 | `submodules/TabBarUI/Sources/TabBarController.swift` — 175, 178 |
