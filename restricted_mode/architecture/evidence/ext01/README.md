# Evidence EXT-01

Base SHA: `87a048874df13b1f865f7ddfa1855b04e7367692`. Дата: 2026-09-28.

- `source_manifest.json`: пути, SHA-256 и строки-якоря 45 записей исходников. Это CODE evidence, не runtime assertions.
- `bundle_inventory.json`: только имена extensions, extension points и minimum OS из прежнего чистого Simulator baseline bundle. Приватные bundle IDs и credentials не экспортированы; подпись не проверена.
- `surface_search.json`: ограниченный поиск современных ActivityKit/AppIntent/CarPlay маркеров. Пустой результат не доказывает отсутствия платформенной поверхности.
- `validation.json`: проверка документов/manifest; не тест защищённого режима.

## Воспроизведение source inventory

На указанном checkout прочитать пути/строки из manifest и вызывающий код. Для каждого source сравнить SHA-256 содержимого и строку с needle. При расхождении обновить анализ, а не только номера строк.

Для каждого pattern из surface_search выполнить без shell-подстановки:

```text
rg -l <pattern> Telegram submodules --glob '*.swift' --glob '*.m' --glob '*.h' --glob '*.plist' --glob BUILD
```

Exit 0 означает наличие matching files; 1 — отсутствие совпадений в этом scope. Другой код — ошибка, её нельзя засчитывать как отсутствие функции. Проверять обработчики legacy INIntent и notification categories отдельно: они не обязаны содержать современные маркеры.

## Воспроизведение bundle inventory

В отдельной baseline сборке/распакованном IPA прочитать `Payload/Telegram.app/PlugIns/*.appex/Info.plist`: имя appex, `NSExtension.NSExtensionPointIdentifier`, `MinimumOSVersion`. Проверить наличие `Watch/`. Не использовать конфигурацию или контейнер личного аккаунта. Сравнение с BUILD обязательно: default target, отключённая конфигурация, упакованный target и реально разрешённый подписью target — разные свидетельства.

Предыдущий артефакт в `/tmp` может быть удалён системой. В этом случае inventory JSON остаётся записью наблюдения; новая сборка и её происхождение должны фиксироваться отдельно. Наличие appex не подтверждает его запуск/работу на устройстве.

Runtime-план XT01–XT31 находится в EXT-01_RESULT.md; эти сценарии не выполнялись.
