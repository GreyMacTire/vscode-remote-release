# Инвентаризация сетевого функционала — Remote-SSH 0.128.0

**Цель проекта:** корпоративная сборка с полностью отключённым сетевым функционалом, кроме прямого подключения к другому хосту (SSH). Механизм загрузки vscode-server выделен как **отдельный функционал** (см. §1) — он нужен для работы подключения, но его эндпоинт/зеркало решается отдельно.

**Объект:** `orig_vsix/ms-vscode-remote.remote-ssh-0.128.0/` (распакованный VSIX; исходники расширения закрыты — модификация = патчинг бандлов `out/extension.js` (~1.1 МБ), `out/resolver.js` (~823 КБ) и `package.json`).

---

## 1. Механизм загрузки VS Code Server (отдельный функционал)

Ядро установки сервера на удалённый хост. Оставить, но при необходимости направить на внутреннее зеркало.

- **URL скачивания:** `https://update.code.visualstudio.com/commit:<commit>/<артефакт>/<quality>`
  - артефакты: `server-<platform>` (обычный), `server-<platform>-web` (режим Web Access, см. §5), `vscode-cli-<platform>` (CLI, качается вместе с `-web`);
  - `<commit>` — commit ID клиента (версии клиента и сервера должны совпадать);
  - скрипты скачивания встроены в `resolver.js` (bash: wget/curl с `--connect-timeout=7`; PowerShell: `irm` + TLS1.2, `TimeoutSec=20`, `OutFile=vscode-server.zip` / `vscode-cli-<commitId>.zip`); отдельный файл `out/install-script/scripts/linux-exec-server-installer.sh`.
- **Два режима загрузки** (настройка `remote.SSH.localServerDownload`: `auto` | `always` | `off`; есть недокументированное значение `manual-bootstrap`; легаси — `remote.SSH.allowLocalServerDownload`):
  - скачивание **на удалённом хосте** напрямую с CDN (wget/curl/irm);
  - скачивание **локально** и передача по SSH: local server (`out/localServer.js`, настройка `remote.SSH.useLocalServer`) отдаёт тарболл через localhost, удалённый хост забирает его через forwarded port (либо SCP-копирование).
- **Точка подмены URL (важно для корп. зеркала):** в коде есть `useOSS()` / `ossDownloadUrl()` — если задан OSS-URL, он подставляется в `UPDATE_URL` установочного скрипта вместо `https://update.code.visualstudio.com`. Это готовый крючок для внутреннего зеркала.
- **Прокси для загрузки:** в установочный скрипт пробрасываются `httpProxy`/`httpsProxy` (аргументы wget/curl); есть настройка `remote.SSH.useCurlAndWgetConfigurationFiles` — использование `.curlrc`/`.wgetrc` на удалённом хосте.
- **Прочее в установочном скрипте:** `remote.SSH.serverInstallPath` (пути установки per-host), `lockfilesInTmp`, `useFlock`, `useExecServer`, `enableRemoteCommand`, `remoteServerListenOnSocket`, `serverPickPortsFromRange`, `preferredLocalPortRange`, `bindHost`; маркеры прогресса `trigger_server_download`/`trigger_server_download_end`; коды ошибок `UnifiedStatusCode` / `InstallUnpackCode`.
- **Через установочный скрипт также передаются:**
  - `--install-extension <id>` из `remote.SSH.defaultExtensions` → сервер качает расширения с маркетплейса на удалённом хосте (см. §6);
  - `telemetryLevel` клиента → уровень телеметрии самого сервера.
- **Удаление сервера:** команды `cleanRemoteServer` / `cleanCurrentRemoteServer` / `cleanLocalServer` / `cleanDevBox` (скрипты в `out/localServer.js` — kill node + `rm -rf ~/.vscode-server[/bin]`).
- **Сам VS Code Server** (бинарник с CDN) — отдельный продукт со своей телеметрией и своим маркетплейсом (его `product.json`), в этот VSIX не входит.

## 2. Телеметрия — отключить

- `package.json`: поле **`aiKey`** (`0c6ae279ed8443289764825290e4f9e2-...`) + зависимость `@vscode/extension-telemetry`.
- Эндпоинты в бандлах: `mobile.events.data.microsoft.com`, `browser.events.data.microsoft.com` (1DS), `dc.services.visualstudio.com`, `dc-int.services.visualstudio.com`, `breeze.aimon.applicationinsights.io` (App Insights).
- `extension/telemetry.json` — декларации событий: `resolver` (тайминги `timing.*` — download/install/scp/serverstart/totalresolve, `resolveattempts`, `retries`, `askedpw`/`asked2fa`/`askedpassphrase`/`askedhostkey`, `didlocaldownload`, `dynamicforwarding`, `osreleaseid`, ...), `extraInfo` (`remotecloud`), `copilotChat`.
- `@vscode/extension-telemetry` уважает `telemetry.telemetryLevel`, но для «полностью отключено» надёжнее вырезать `aiKey`/заглушить `sendTelemetryEvent`.

## 3. Определение облачного провайдера (IMDS) — отключить

- Функция `guessRemoteCloud` в `extension.js`: при разрешении соединения шлёт HTTP-запросы к metadata-сервисам `http://169.254.169.254/metadata/instance` (Azure IMDS, заголовок `Metadata:true`), `http://169.254.169.254/metadata/v1/id` (DigitalOcean), `http://metadata.google.internal` (GCP); enum `RemoteCloud`: azure/aws/digitalocean/gcp.
- Результат уходит в телеметрию `extraInfo` (`reportExtraInfo`).

## 4. Copilot Chat — отключить

- Chat-участник `remote-ssh` (package.json `chatParticipants`), настройка **`remote.SSH.experimental.chat` — включена по умолчанию**; стейт: `config.remote.SSH.experimental.chat`.
- Внутри: `vscode.lm.selectChatModels` (вендор `copilot`, семейство `o1`), зависимости `@vscode/prompt-tsx`, `@vscode/chat-extension-utils`, статические хинты в `out/chat/static-hints/` (troubleshooting.md, tips-and-tricks.md), команда `clearAllSessionHints`.
- «Diagnose with Copilot»: устанавливает **`github.copilot-chat` с маркетплейса** (`workbench.extensions.installExtension`) и обрабатывает URI `/continueWithCopilot` (activation `onUri`).
- Телеметрия `ms-vscode-remote.remote-sshcopilotChat` (`conversationhashistory`, `isdiagnosewithcopilotsession`).

## 5. Web Access (веб-доступ к удалённому серверу) — отключить

- Скрытая настройка `remote.SSH.enableWebAccess` (не объявлена в `configuration`, читается напрямую в `resolver.js`).
- Когда включена: установочный скрипт качает артефакт `server-<platform>-web` + `vscode-cli`, на удалённом хосте поднимается веб-сервер (веб-версия VS Code).
- Команда `openWebUI` (package.json, меню status bar): открывает в браузере порт из `serverConfiguration.remoteListeningOn` (через SSH-форвардинг).

## 6. Установка расширений на удалённый хост — отключить

- `remote.SSH.defaultExtensions` и `defaultExtensionsIfInstalledLocally` (мигрирует в `remote.defaultExtensionsIfInstalledLocally`) → флаги `--install-extension "<id>"` для серверного CLI при установке → сервер сам обращается к маркетплейсу на удалённой стороне.

## 7. Extension Pack зависимости — убрать для изолированной сборки

- `package.json` → `extensionPack`: `ms-vscode-remote.remote-ssh-edit`, `ms-vscode.remote-explorer` — оба тянутся с маркетплейса при установке.

## 8. Внешние help-ссылки — опционально

- `remoteHelp` в package.json (`aka.ms/vscode-remote/ssh*`: getting-started, documentation, issues, provide-feedback); команды `getStarted`/`reportIssue`/`provideFeedback`/`getHelp`; ~7 вызовов `env.openExternal`. Сетевых вызовов из процесса нет — просто открытие браузера.

## 9. Оставить (ядро прямого подключения)

- SSH-соединение (ssh binary, `remote.SSH.configFile`/`path`/`connectTimeout`/`remotePlatform`), askpass (`out/askpass-main.js`, локальный IPC).
- Порт-форвардинг: `defaultForwardedPorts`, `enableDynamicForwarding` (SOCKS), `enableAgentForwarding`, `enableX11Forwarding`.
- Прокси-поддержка: `remote.SSH.httpProxy/httpsProxy`, `https-proxy-agent`, SOCKS5-клиент (пакет `socks`).
- Local server / SCP-передача сервера (часть §1).
- `preconnect` (локальный скрипт перед соединением — не сеть расширения).

## 10. Вне этого VSIX (не забыть в общей сборке)

- **Клиент VS Code:** маркетплейс, автообновления, телеметрия ядра — `product.json` (`extensionsGallery`, ...) и настройки (`extensions.autoUpdate`, `telemetry.telemetryLevel`, `update.mode`).
- **VS Code Server на удалённом хосте:** своя телеметрия (уровень приходит из установочного скрипта) и свой маркетплейс (`product.json` сервера).
- **Remote Tunnels / Dev Tunnels:** отдельное расширение (`ms-vscode.remote-server`); здесь `@microsoft/dev-tunnels-contracts` подключён только как типы, рантайм-обращений нет.

---

## Сводка эндпоинтов

| Домен | Назначение | Судьба |
|---|---|---|
| `update.code.visualstudio.com` | скачивание server/cli артефактов | оставить / зеркало (через `ossDownloadUrl`) |
| `mobile.events.data.microsoft.com`, `browser.events.data.microsoft.com` | телеметрия 1DS | отключить |
| `dc.services.visualstudio.com`, `dc-int...`, `breeze.aimon.applicationinsights.io` | App Insights | отключить |
| `169.254.169.254`, `metadata.google.internal` | IMDS-детект облака | отключить |
| маркетплейс (клиент/сервер/Copilot) | расширения | отключить по §4, §6, §10 |
| `aka.ms`, `github.com`, `learn.microsoft.com` и пр. | help-ссылки (`openExternal`) | опционально |
