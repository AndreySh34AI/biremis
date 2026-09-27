# ADR-002: открывать данные после авторизации профиля

Статус: **Proposed**. Дата: 2026-09-27. Commit: `42a84aec8abf72e54958ddf8ca9480514cc0d390`.

## Контекст

E01–E08/E11/E12 показывают раннее создание runtime, общий `.tempkey`, строковый challenge и auth backup в metadata. SharedAccountContext и extensions имеют дополнительные пути открытия. AppLock управляет overlay и lock state; этого недостаточно для I01–I05/I09.

## Предлагаемое решение

Проектировать разрешение на открытие store **до** accountWithId/standaloneStateManager и secret-bearing metadata. Ключи профиля открываются соответствующим PIN по проверенному envelope. Политика и session generation действуют на ingress/output/actions; Closing немедленно отзывает доступ, затем завершает writers/handles. Межпроцессный протокол требует самостоятельной проверки. Реальные структуры и размещение модулей выбираются после STORAGE-01/LIFE-01.

## Альтернативы и последствия

Флаг в AccountContext оставляет Network/Postbox и background доступными. Два каталога с общим `.tempkey` и AccountBackupData не изолируют secrets. Полный новый runtime дорог; сначала проверить существующие factory/disposable/master механизмы и пригодность B/C.

Нужно изменить несколько границ, включая metadata и extensions, а не только UI. Full UX/background изменения требуют D06/D08. Ни полное стирание Swift памяти, ни защита от T4 в том же процессе этим решением не обещаются.

## Принятие

Утверждённые требования, D01/D06/D07, закрытый key inventory, результаты V02/V03/V04/V09/V10/V11 и review. При ошибке ключа нельзя применять destructive fallback как способ пройти тест.
