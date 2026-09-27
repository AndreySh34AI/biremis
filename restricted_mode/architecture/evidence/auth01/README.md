# AUTH-01: процедура live RPC

Этот опыт выполнен через Swift JIT LLDB в существующем процессе Simulator. Он не является CI-тестом. Не повторять создание fixture без отдельного намерения создать новые каналы. Для нового запуска нужны собственный run ID и разрешение владельца аккаунта.

## Подготовка

1. Зафиксировать UUID загруженного TelegramUIFramework. Не считать его соответствующим checkout без проверки.
2. Подключить LLDB к PID текущего форка. Применить module search paths и Clang flags из generated Xcode Swift debug settings **после attach**, до импорта Swift модулей. Привести Bazel output directories к действующей сборке. Не импортировать AccountContext/UIKit в Swift: в этом опыте это вызвало конфликты PCM cache.
3. Получить указатель UIApplication.delegate через ObjC, далее Mirror: `contextValue` → unwrap optional → `context` → `account`. Проверить cast в TelegramCore.Account. Ничего не печатать из Account/context.
4. Импортировать TelegramCore, TelegramApi, SwiftSignalKit, Foundation, MtProtoKit. Сохранить logger flags и выключить logToFile/logToConsole.
5. Хранить callback результаты в одном NSMutableDictionary, на main queue. Persistent value-type переменные LLDB для callback результатов в этом запуске оказались ненадёжны. Cast persistent переменной делать отдельным присваиванием после инициализации.

## Fixture и положительные проверки

- `channels.createChannel(flags: 1, title: uniqueTitle, about: syntheticDescription, geoPoint: nil, address: nil, ttlPeriod: nil)`, `automaticFloodWait: false`.
- Принимать только channel с точным title, отсутствующим username и непустым accessHash. Из ответа собрать inputPeerChannel; адрес держать только в памяти. Передать Updates в account.stateManager.addUpdates.
- `messages.sendMessage(flags: 0, peer: fixturePeer, message: canary, randomId: randomInt64, остальные optional: nil)`; применить Updates.
- Full baseline: `messages.getHistory` для каждого fixture peer, offset/min/max/hash = 0, limit = 10. Сравнивать текст с синтетическим canary в памяти; сохранять только Bool.
- Policy model: `allowed = Set(["visible"])`; ветка `!allowed.contains("hidden")` фиксирует отказ без RPC.
- Повторить getHistory(hidden) напрямую через тот же Account.network. Проверить точное совпадение canary.
- `messages.getPeerDialogs(peers: [.inputDialogPeer(peer: hidden)])`: проверить canary среди messages.
- `messages.search(flags: 0, peer: hidden, q: hiddenCanary, filter: .inputMessagesFilterEmpty, limit: 10, offsets/dates/hash: 0, остальные optional: nil)`: проверить canary.
- После start продолжать process, затем interrupt и читать только allowlisted Bool/Int/status. Не оставлять Simulator остановленным между действиями пользователя.

## Отрицательный контроль

Новый MTContext с сериализацией, encryptionProvider и apiEnvironment из действующего Network; `isTestingEnvironment: false`, `useTempAuthKeys: false`. Подключить адрес того же DC через setSeedAddressSetForDatacenterWithId. **Не присваивать личный keychain, не копировать authInfo/authToken.**

Создать MTProto с новым context, текущим datacenterId, usageCalculationInfo nil, requiredAuthToken nil, authTokenMasterDatacenterId 0. Оставить стандартный `useUnauthorizedMode=false`: нужен обычный криптографический handshake без пользовательского login. Создать MTRequestMessageService и добавить его в новый MTProto.

MTRequest.payload — сериализованный тот же getHistory(hidden). completed сохраняет только errorCode и Bool `errorDescription == "AUTH_KEY_UNREGISTERED"` на main queue. Любой успешный ответ пометить как unexpected success; не сериализовать ответ. Add request, resume MTProto, continue process. В этом запуске получен 401 AUTH_KEY_UNREGISTERED.

## Завершение

Dispose подписки стенда, удалить только его pending request, stop отрицательного MTProto, восстановить оба logger flags, detach LLDB с продолжением процесса. Аккаунт, каналы и обычные сессии не удалять. Экспортировать только allowlisted проверочные значения; не экспортировать общий словарь, содержащий peer handles.
