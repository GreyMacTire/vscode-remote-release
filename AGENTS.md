# Контекст проекта

Цель: **корпоративная сборка Remote-SSH с полностью отключённым сетевым функционалом**, кроме прямого подключения к другому хосту по SSH. Механизм загрузки vscode-server — отдельный функционал: нужен для работы, но эндпоинт может быть заменён на внутреннее зеркало (крючок `ossDownloadUrl`).

Объект работы: `orig_vsix/ms-vscode-remote.remote-ssh-0.128.0/` (распакованный VSIX; исходники закрыты — правим собранные бандлы `out/extension.js`, `out/resolver.js` и `package.json`).

**Полная инвентаризация сетевого функционала (память проекта): [NETWORK_INVENTORY.md](NETWORK_INVENTORY.md)** — обязательно сверяться с ней перед планированием правок. Кратко, отключению подлежат: телеметрия (`aiKey` + `@vscode/extension-telemetry`), IMDS-детект облака (`guessRemoteCloud`), Copilot-чат (`remote.SSH.experimental.chat`, Diagnose with Copilot), Web Access (`enableWebAccess`, `-web`-артефакт), установка расширений на удалённый хост (`defaultExtensions`), extensionPack-зависимости; отдельно решается зеркало для `update.code.visualstudio.com`.
