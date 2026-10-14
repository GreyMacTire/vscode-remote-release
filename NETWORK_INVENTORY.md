# Инвентаризация сетевого функционала — Remote-SSH 0.128.0

**Цель проекта:** корпоративная сборка с полностью отключённым сетевым функционалом, кроме прямого подключения к другому хосту (SSH). Механизм загрузки vscode-server выделен как **отдельный функционал** (см. §1) — он нужен для работы подключения, но его эндпоинт/зеркало решается отдельно.

**Объект:** `orig_vsix/ms-vscode-remote.remote-ssh-0.128.0/` (распакованный VSIX; исходники расширения закрыты — модификация = патчинг бандлов `out/extension.js` (~1.1 МБ), `out/resolver.js` (~823 КБ) и `package.json`).

> **СТАТУС: правки ПРИМЕНЕНЫ** к сборке `ms-vscode-remote.remote-ssh-0.129.2026100915-patched.vsix` — см. раздел **«Применённые правки»** в конце документа (с инструкцией по переносу на будущие версии расширения). Это единственная поддерживаемая сборка; рабочие файлы 0.128.0 (включая её патченную версию) удалены.

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

---

# Применённые правки (корпоративная сборка)

**Применено к:** `ms-vscode-remote.remote-ssh-0.129.2026100915` (распаковано в `ms-vscode-remote.remote-ssh-0.129.2026100915/`, рядом лежит нетронутый оригинальный VSIX — эталон для сверки).
**Результат:** `ms-vscode-remote.remote-ssh-0.129.2026100915-patched.vsix`, версия внутри оставлена оригинальная.
**Подтверждено вживую:** Windows→Windows, macOS→Windows, macOS→macOS(arm), macOS→Linux.

## Сводка правок (P1–P10)

Если не указано иное, каждая правка вносится в **оба** бандла — `extension/out/extension.js` и `extension/out/resolver.js` (в каждом своя копия кода).

| # | Правка | Закрывает | Где |
|---|---|---|---|
| P1 | `localDownload()` → всегда `"manual-bootstrap"` | §1 | оба бандла |
| P2 | `useExecServer()` → `false` (без предупреждения) | §1 | оба бандла |
| P3 | Клиентский обработчик bootstrap: win32-запрет убран, имя файла от артефакта, платформо-зависимые pickup-каталоги | §1 | оба бандла |
| P4 | `$forceLocalDownload = $True` в классическом PowerShell-шаблоне | §1 | оба бандла (только классический шаблон!) |
| P5 | Детект архитектуры: `PROCESSOR_ARCHITEW6432` + однострочные if | §1 | оба бандла (оба PS-шаблона) |
| P6 | Телеметрия: заглушка `createTelemetryReporter`, удалён `aiKey` и зависимость | §2 | extension.js + package.json + встроенные копии манифеста |
| P7 | `guessRemoteCloud` → пустая функция | §3 | extension.js |
| P8 | Copilot-чат: гейт `false`, URI-обработчик — ранний return, участник и настройка удалены из манифеста | §4 | оба бандла + package.json |
| P9 | `extensionPack` удалён (манифест, бандлы, vsixmanifest) | §7 | везде |
| P10 | `getSshDefaultExtensions()` → `[]`, настройка удалена (`IfInstalledLocally` НЕ трогать!) | §6 | оба бандла + package.json |
| P11 | Из `package.json` удалены настройки `localServerDownload`, `useExecServer`, `allowLocalServerDownload` | — | package.json |

## Детали правок (якоря — по содержанию, не по именам)

**Важно про якоря:** минифицированные имена (`h`, `r`, `o`, `t`, …) меняются от сборки к сборке. Искать по устойчивым строковым литералам (имена настроек, сообщения, фрагменты скриптов). В 0.128.0 и 0.129.2026100915 зоны правок совпали байт-в-байт — на будущее это НЕ гарантия. **Встроенные копии package.json:** `getPackageJson()` в бандлах возвращает копию манифеста, вшитую в каждый бандл текстом — правки манифеста надо вносить в трёх местах (standalone + обе копии), удаление — brace-matching'ом с проверкой `json.loads`.

### P1. Режим загрузки сервера
Якорь: функция, читающая `"remote.SSH.allowLocalServerDownload"` и `"remote.SSH.localServerDownload"` и возвращающая `"off"|"always"|"manual-bootstrap"|"auto"`.
Замена: `function h(){return"manual-bootstrap"}` (тело функции — одна строка).

### P2. Exec-server
Якорь: `useExecServer=function(){return"manual-bootstrap"===<fn>()?(…showWarningMessage("Local download mode is configured as 'manual-bootstrap'…")…):…get("remote.SSH.useExecServer",!0)}`.
Замена: `useExecServer=function(){return!1}`.

### P3. Обработчик manual-bootstrap на клиенте
Якорь: `"vscode-remote-ssh-bootstrap"` (в исходнике — единственное вхождение на бандл). Итоговый вид (после всех итераций, включая смену каталогов):
```js
c=o.join("win32"===a.platform()?"C:\\Program Files\\vscode-server":"/opt/vscode-server","vscode-"+r.artifact+(r.artifact.includes("win32")?".zip":".tar.gz"));try{
```
(в extension.js деструктуризация `{downloadParams:r,productInfo:i}`, l10n — `s`; в resolver.js — `{downloadParams:r,productInfo:s}`, l10n — `i`; проверять по факту). Убран `throw "Windows remotes do not support local bootstrap"`.
Pickup-каталоги: Windows-клиент → `C:\Program Files\vscode-server\`, macOS/Linux-клиент → `/opt/vscode-server/`. Каталоги admin-only на запись — предусмотреть раздачу файлов ИТ.

### P4. PowerShell: без попытки CDN
Якорь: `$forceLocalDownload = ${"always"===<v>?"$True":"$False"}` → `$forceLocalDownload = $True`.
**Осторожно:** таких строк ДВЕ на бандл (классический server-шаблон и exec-CLI-шаблон). Патчить только классический — он определяется по контексту перед флагом: `$sFile = "$serverRoot\\bin\\$commitId\\bin\\${…}.cmd"`. CLI-шаблон не трогать (мёртвый код, exec выключен в P2).

### P5. Детект архитектуры (PowerShell-шаблоны)
Якорь: комментарий `# https://docs.microsoft.com/en-us/windows/win32/winprog64/wow64-implementation-details?redirectedfrom=MSDN` (2 вхождения на бандл: классический + CLI — править оба). В бандле шаблон хранится с **литеральными `\n`/`\t`** (двухсимвольными) внутри backtick-шаблона — патчить как байтовые последовательности.
Замена (в отрендеренном виде):
```powershell
$p = $env:PROCESSOR_ARCHITEW6432
if (-not $p) { $p = $env:PROCESSOR_ARCHITECTURE }
$arch = ''
if ($p -eq 'AMD64' -or $p -eq 'IA64') { $arch = 'x64' } elseif ($p -eq 'ARM64') { $arch = 'arm64' }
if ($arch -eq '') { "Unsupported architecture '$p'."; q ${X.UnifiedStatusCode.UnsupportedArch} }
```
`${X…}` — JS-интерполяция из окружающего кода, сохранить как есть. **Урок (был баг):** замена обязана поглотить закрывающий `\n}` исходного блока — иначе двойная скобка `}}` (лина не ломает JS-синтаксис, но калечит PowerShell). Контрольный маркер: `UnsupportedArch} }` без `\n}}` рядом.

### P6. Телеметрия
Якорь (только extension.js — в resolver.js репортера нет): `createTelemetryReporter` с `new <mod>.default(<pkg>.aiKey)`.
Замена: `t.createTelemetryReporter=async function(e){return{telemetryLevel:"off",sendTelemetryEvent(){},sendTelemetryErrorEvent(){},dispose(){}}}`.
Дополнительно: удалить `"aiKey":"0c6ae…-7255",` из package.json и из обеих встроенных копий; из standalone `package.json` — ещё и `"@vscode/extension-telemetry":"^0.9.0",` из dependencies.
Побочный эффект (нужный): геттер `telemetryLevel` кормит `--telemetry-level` сервера → сервер стартует с `off` независимо от настроек клиента.

### P7. IMDS-детект облака
Якорь (только extension.js): `t.guessRemoteCloud=async function(…){…Promise.allSettled…}`.
Замена: `t.guessRemoteCloud=async function(e,t,n){}`. Единственный потребитель — `reportExtraInfo` (телеметрия), `undefined` обрабатывает безопасно.

### P8. Copilot-чат — три входа
1. Гейт: `useExperimentalChat=function(){return …get("remote.SSH.experimental.chat")??!1}` → `useExperimentalChat=function(){return!1}` (оба бандла).
2. URI-обработчик (только extension.js): `async handleUri(e){const t=e.query.split("&").reduce` → вставить ранний `return;` сразу после `{`. Заодно недостижимой становится установка `github.copilot-chat` (`workbench.extensions.installExtension`).
3. Манифест (standalone + обе встроенные копии): удалить `contributes.chatParticipants[…]` и настройку `remote.SSH.experimental.chat`.
Инертные остатки: строка локализации `chatParticipants.description`, код `lm.selectChatModels`, файлы `out/chat/static-hints/`.

### P9. extensionPack
Удалить `"extensionPack":["ms-vscode-remote.remote-ssh-edit","ms-vscode.remote-explorer"],` из package.json + обеих встроенных копий; из `extension.vsixmanifest` — `<Property Id="Microsoft.VisualStudio.Code.ExtensionPack" … />`. Рантайм-ссылка на `ms-vscode.remote-explorer` в коде защищена `getExtension(…)&&` — правки не требует.

### P10. defaultExtensions
Якорь (оба бандла): `getSshDefaultExtensions=function(){…get("remote.SSH.defaultExtensions")…Array.isArray…filter…}`.
Замена: `t.getSshDefaultExtensions=function(){return[]}`. Настройку удалить из манифеста (standalone + встроенные копии). **`getDefaultExtensionsIfInstalledLocally` — НЕ трогать** (это безопасный механизм «поставить на удалённый то, что стоит локально», без сети на хосте).

## Процедура патчинга и пересборки

1. Распаковать новый VSIX (`unzip` в каталог), сверить якоря (grep по содержательным строкам, `count==1`).
2. Правки вносить python-скриптом: индексное `find` → byte-exact замена, `assert count==1`, контекстные проверки (классический vs CLI-шаблон — по `$sFile` рядом).
3. Синтакс-контроль бандлов (node на машине сборки нет): обернуть в невызываемую функцию и прогнать через JavaScriptCore:
   ```bash
   { printf 'function __never_called__(){\n'; cat bundle.js; printf '\n}\n'; } > /tmp/jxa.js
   osascript -l JavaScript /tmp/jxa.js
   ```
4. Манифест: brace-matching удаление + `json.loads` валидация.
5. Пересборка: из корня распакованного VSIX — `find . -name .DS_Store -delete; zip -r -D -X ../<имя>-patched.vsix .` (без записей-каталогов). Версию внутри НЕ менять (см. ниже).
6. Верификация: список файлов идентичен оригиналу (`unzip -l` diff), маркеры правок внутри VSIX (см. таблицу ниже), `unzip -t`.

**Политика версий:** версию внутри оставлять оригинальную — у semver `X.Y.Z-N` prerelease-суффикс НИЖЕ релизной, VS Code считал бы сборку устаревшей и предлагал «обновление» до маркетплейсной.

## Контрольные маркеры (grep по содержимому VSIX, ожидаемые количества)

| Маркер | Ожидание |
|---|---|
| `function h(){return"manual-bootstrap"}` | 2 |
| `useExecServer=function(){return!1}` | 2 |
| `"win32"===a.platform()?"C:\\Program Files\\vscode-server":"/opt/vscode-server"` | 2 |
| `Windows remotes do not support local bootstrap` | 0 |
| `$forceLocalDownload = $True` | 2 |
| `$forceLocalDownload = ${"always"===` (CLI-шаблон, нетронут) | 2 |
| `PROCESSOR_ARCHITEW6432` (4 патченных + 1 upstream-детект WOW64 на бандл) | 6 |
| `createTelemetryReporter=async function(e){return{telemetryLevel:"off"` | 1 |
| `guessRemoteCloud=async function(e,t,n){}` | 1 |
| `useExperimentalChat=function(){return!1}` | 2 |
| `async handleUri(e){return;` | 1 |
| `aiKey` | 0 |
| `extensionPack":[` | 0 |
| `getSshDefaultExtensions=function(){return[]}` | 2 |
| `get("remote.SSH.defaultExtensionsIfInstalledLocally")` (сохранён) | 2 |

## Поведение сборки (итог)

- Подключение: SSH → скрипт на хосте → сервер установлен? → запуск; иначе маркеры → клиент берёт архив из pickup-каталога → scp (архив + `.done`) → распаковка → запуск. **Сеть на хосте не используется ни для одной ОС.**
- Имена файлов в pickup-каталоге = CDN-имена артефактов: `vscode-server-linux-x64.tar.gz`, `vscode-server-win32-x64.zip` и т.д. **macOS-хосты:** CDN отдаёт `.zip` (`vscode-server-darwin[-arm64].zip`) — файл переименовать в `vscode-server-darwin[-arm64].tar.gz` (содержимое-зип распакуется bsdtar; так же делает и оригинальный скрипт). `armhf` для современных коммитов не собирается (404).
- Коммит архива = коммит подключающегося клиента (Help → About).
- Расширения на хост ставятся только передачей с клиента (UI удалённого окна, автосинхронизация, `IfInstalledLocally`, VSIX) — хост в маркетплейс не ходит.
- Пароль запрашивается на каждый новый SSH-процесс (install + scp = 2 раза); «бесконечный пароль» = симптом retry-цикла (до 30 попыток), реальная причина — в логе между запросами (обычно битый `~/.vscode-server/bin/<commit>` на хосте → удалить).

## Инертные остатки (сознательно не вырезаются)

URL CDN/телеметрии/IMDS, exec-server-механика и библиотека телеметрии — мёртвые строки в бандлах; `-web` артефакты (§5, настройка `enableWebAccess` скрыта и выключена по умолчанию — при включении сети нет, искомый файл был бы локальным); help-ссылки (§8, `openExternal`).

## Чек-лист внедрения (вне VSIX)

- на клиентах: `extensions.autoUpdate: off` (или корп-каталог расширений) — иначе маркетплейс заменит патч;
- процесс обновления: новый коммит клиента = новые артефакты в pickup-каталоги (раздаёт ИТ);
- смоук-тест после каждой пересборки: одна связка на Windows-хост + одна на Linux/macOS-хост, свежий коммит, без предустановленного сервера.
