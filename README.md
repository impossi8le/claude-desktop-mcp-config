<div align="center">

# Claude Desktop MCP Config

**Готовая сборка для Claude Desktop на Windows: 14 MCP-серверов, два прокси-фикса, собственный orchestrator-MCP, LiteLLM-прокси и каталог из 42 скиллов.**

![MCP](https://img.shields.io/badge/MCP-14%20servers-6E56CF)
![Skills](https://img.shields.io/badge/skills-42-D97706)
![Windows](https://img.shields.io/badge/Windows-10%2F11-0078D4?logo=windows&logoColor=white)
![Node](https://img.shields.io/badge/Node.js-18%2B-339933?logo=node.js&logoColor=white)

![License MIT](https://img.shields.io/badge/license-MIT-green)
</div>

> **Коротко.** Настройка MCP на Windows — это десяток ручных шагов, где каждый сервер требует своего пакета, ключа и иногда обхода сломанной JSON-схемы. Здесь всё сведено в один конфиг с плейсхолдерами, скрипт генерации реального конфига и два маленьких прокси, которые чинят несовместимости конкретных серверов. В репозитории нет ключей — только имена переменных.

**Ссылки:** [перенос на новый ПК](INSTALL-NEW-PC.md) · [задание для агента](CLAUDE-SETUP-TASK.md) · [orchestrator-MCP](orchestrator-mcp/) · [навыки](skills/)

## Проблема

- Claude Desktop подключает MCP-серверы через JSON, и один неверный символ или несовместимая схема инструмента ломают запуск всего набора без внятной ошибки.
- Каждый сервер ставится по-своему: `npx`-пакет, Python-модуль, бинарь на Rust, Docker-образ, отдельный `.exe` из GitHub Releases.
- Часть серверов отдаёт JSON Schema draft-07, которую клиент не принимает; лечится не настройкой, а прокси между клиентом и сервером.
- Ключи легко случайно закоммитить. Конфиг с реальными токенами не должен попадать в git вообще.
- Скиллы и инструкции для агента живут в разных местах и не переносятся на новую машину одним действием.

## Что умеет

- Поднимает 14 MCP-серверов из одного конфига: память, Docker, файлы, GitHub, SSH, поиск, браузер, документация, оркестратор и другие.
- Генерирует реальный `claude_desktop_config.json` из шаблона: `setup-config.ps1` спрашивает ключи и пишет файл в `%APPDATA%\Claude\`, не оставляя их в репозитории.
- Проверяет окружение и ставит зависимости одной командой `install.bat`.
- Чинит два известных несовместимых сервера локальными прокси: MCP Memory (`memory-proxy.cjs`) и n8n (`n8n-mcp-fix-proxy.mjs`).
- Запускает собственный MCP-сервер оркестратора с шестью ролями: аналитик, архитектор, согласователь, разработчик, devops, QA.
- Подменяет модели Claude через LiteLLM-прокси на `routerai.ru`, если нужен другой маршрут запросов.
- Разворачивает каталог скиллов копированием в `%USERPROFILE%\.claude\skills` или как плагин через `.claude-plugin/marketplace.json`.
- Переносит всю сборку на новый компьютер по пошаговой инструкции, где ключи вводит владелец машины, а не агент.

## Стек

| Слой | Технологии |
|---|---|
| Клиент | Claude Desktop (Cowork / 3p), MCP по stdio |
| Серверы | Node.js-пакеты через `npx`, Python-модули, Docker, Rust-бинарь, `github-mcp-server.exe` |
| Прокси | Node.js: `memory-proxy.cjs`, `n8n-mcp-fix-proxy.mjs` |
| Свой сервер | `orchestrator-mcp` на `@modelcontextprotocol/sdk` ^1.30.0, ESM |
| Модельный прокси | LiteLLM + `routerai.ru`, конфиг `ClaudeConfigLiteLLM.yaml` |
| Скиллы | 42 каталога с `SKILL.md`, устанавливаются копированием или плагином |
| Установка | PowerShell (`setup-config.ps1`), `install.bat`, Windows 10/11 |

## Как это устроено

```text
Claude Desktop
  ├─ stdio → MCP-сервер (npx / python / бинарь / docker)
  ├─ stdio → memory-proxy.cjs ──→ MCP Memory     (срез $schema draft-07)
  ├─ stdio → n8n-mcp-fix-proxy.mjs ──→ n8n MCP   (починка схемы инструмента)
  └─ stdio → orchestrator-mcp (index.js) ──→ LLM API

Claude Desktop ── ANTHROPIC_BASE_URL=http://localhost:4000 ──→ LiteLLM ──→ routerai.ru
```

Конфиг из репозитория содержит 14 записей `mcpServers` и 9 разных плейсхолдеров `YOUR_*`: PAT для GitHub, ключи Tavily и RouterAI, параметры SSH, путь к профилю пользователя. Реальные значения подставляет `setup-config.ps1`; сам файл с ключами исключён через `.gitignore`.

Claude Desktop читает файл из `%APPDATA%\Claude\`, поэтому `install.bat` и `setup-config.ps1` работают именно с этим путём, а копия в репозитории остаётся шаблоном.

`orchestrator-mcp/index.js` — отдельный Node-процесс, который принимает MCP-запросы и превращает их в конвейер из шести ролей. Его `package.json` объявляет единственную зависимость и запускает тесты через `node --test`.

Каталог `skills/` устроен одинаково: каждый навык — папка с `SKILL.md` и, при необходимости, скриптами и шаблонами. Файл `.claude-plugin/marketplace.json` описывает все 42 навыка как один плагин `orchestrator-skills` с автором `impossi8le`; это второй способ установки помимо копирования.

## Состав MCP-серверов

| Сервер | Что делает | Что требует |
|---|---|---|
| `local-memory` | Долговременная память в виде графа знаний | `memory-proxy.cjs` |
| `mcp_docker` | Управление Docker | Docker Desktop |
| `filesystem` | Доступ к файлам | — |
| `sequential-thinking` | Пошаговые рассуждения | — |
| `github` | PR, issues, поиск кода | `GITHUB_PERSONAL_ACCESS_TOKEN`, `github-mcp-server.exe` |
| `n8n-local` | Автоматизация n8n | URL, Bearer-токен, `n8n-mcp-fix-proxy.mjs` |
| `curl-client` | HTTP-запросы | — |
| `alolite-ssh` | Выполнение команд по SSH | хост, логин, ключ или пароль |
| `tavily-mcp` | Веб-поиск | `TAVILY_API_KEY` |
| `tavily2-mc` | Второй инстанс Tavily | `TAVILY_API_KEY` |
| `playwright` | Браузерная автоматизация | — |
| `orchestrator-mcp` | Конвейер из шести ролей | `ROUTERAI_API_KEY`, `npm install` |
| `context7` | Документация библиотек | — |
| `lean-ctx` | Сжатие контекста кода | Rust-бинарь `lean-ctx` |

Два экземпляра Tavily заведены отдельными записями `tavily-mcp` и `tavily2-mc`: конфиг позволяет задать каждому свой ключ, не меняя остальные маршруты.

## Состав навыков

| Группа | Примеры | Сколько |
|---|---|---|
| Официальные Anthropic (с `LICENSE.txt`) | `docx`, `pdf`, `pptx`, `xlsx`, `mcp-builder`, `skill-creator`, `canvas-design`, `brand-guidelines` | 16 |
| Сторонний набор superpowers | `brainstorming`, `writing-plans`, `test-driven-development`, `systematic-debugging`, `using-superpowers`, `subagent-driven-development` | 15 |
| Набор Tavily | `tavily-search`, `tavily-extract`, `tavily-crawl`, `tavily-map`, `tavily-research` и ещё три | 8 |
| Собственные | `graphify`, `orchestrator`, `docs-orchestration` | 3 |

Итого 42 каталога с `SKILL.md`. Шестнадцать официальных навыков Anthropic несут их `LICENSE.txt`; условия у них заданы в этих файлах и отличаются от лицензии самого репозитория. У остальных отдельного файла лицензии в каталоге нет.

### Оркестратор: шесть ролей

`orchestrator-mcp` — не обёртка над чужим сервером, а собственный процесс на `@modelcontextprotocol/sdk`. Он проводит задачу через роли analyst → architect → approver → developer → devops → qa, хранит результаты этапов, позволяет вмешаться в конкретную роль, отредактировать результат и решить судьбу гейта (`retry`, `skip`, `stop`). Роли возвращают текст; менять файлы, гонять тесты и собирать проект они не умеют — это работа клиента, который их вызвал.

## Инженерные находки

- **Прокси вместо патча библиотеки.** MCP Memory отдаёт инструменты с `$schema` draft-07, который клиент не принимает. `memory-proxy.cjs` — маленький stdio-посредник, который вырезает поле на лету. Аналогично `n8n-mcp-fix-proxy.mjs` чинит схему n8n (в комментариях указан n8n#36361). Оба решения локальные: обновление upstream-пакета не ломает обход.
- **Плейсхолдеры как контракт безопасности.** В репозитории лежит конфиг только с `YOUR_*`; проверка коммита показывает отсутствие строк, похожих на реальные токены. Генерация реального файла вынесена в отдельный скрипт, а `.gitignore` закрывает `claude_desktop_config.json`, `.env` и ключи. Это защита от случайного коммита, а не от чтения файла на машине.
- **Тест сервера как чёрного ящика.** `orchestrator-mcp/test/orchestrator.test.js` поднимает локальный HTTP-сервер-заглушку, запускает настоящий `index.js` дочерним процессом и проверяет обмен по протоколу. Шесть тестов проверяют поведение, а не внутренние функции, — поэтому рефакторинг не ломает проверки.
- **Скиллы как устанавливаемый плагин.** `.claude-plugin/plugin.json` и `marketplace.json` превращают папку с навыками в плагин с манифестом и явным списком из 42 путей. Это снимает ручное копирование и делает состав каталога проверяемым.
- **Честное разделение своего и чужого.** 16 навыков несут `LICENSE.txt` Anthropic — это официальные скиллы из каталога Anthropic. Наборы `superpowers-*` и `tavily-*` — сторонние. Авторских в каталоге три — `graphify`, `orchestrator`, `docs-orchestration`: они описывают этот проект и его окружение, а не повторяют каталожные шаблоны. Считать все 42 своими нельзя.

## Быстрый старт

```bat
git clone https://github.com/impossi8le/claude-desktop-mcp-config.git
cd claude-desktop-mcp-config
install.bat
```

Сгенерировать конфиг с ключами (ключи вводятся локально, в git не попадают):

```powershell
powershell -ExecutionPolicy Bypass -File setup-config.ps1
```

Полностью перезапустить Claude Desktop (трей → Exit, затем запуск заново) — серверы подхватятся при старте.

Установить навыки:

```bat
xcopy /E /I /Y skills "%USERPROFILE%\.claude\skills"
```

Опционально — LiteLLM-прокси:

```bash
pip install "litellm[proxy]"
litellm --config ClaudeConfigLiteLLM.yaml --port 4000
```

и в конфиге Claude Desktop указать `ANTHROPIC_BASE_URL=http://localhost:4000` и токен прокси.

## Требования

| Компонент | Зачем | Проверка |
|---|---|---|
| Windows 10/11 | ОС, PowerShell и `.bat` | — |
| Node.js 18+ | MCP-серверы | `node --version` |
| Python 3.9+ в `PATH` | LiteLLM | `python --version` |
| Git | клонирование | `git --version` |
| Docker Desktop | `mcp_docker` | запущенный Docker |
| Rust + cargo | `lean-ctx` | `cargo --version` |
| `github-mcp-server.exe` | `github` | скачивается из GitHub Releases |
| Claude Desktop | клиент | Cowork / 3p |

Node.js и Python обязательны: без них `install.bat` останавливается. Остальное нужно только тем серверам, которые вы оставляете в конфиге.

## Если что-то не работает

| Симптом | Что проверить |
|---|---|
| MCP не появился | Валидность JSON и полный перезапуск Claude Desktop (трей → Exit) |
| `npx` долго стартует | Первый запуск качает пакет; дальше работает кэш |
| `github` не работает | Токен и права `repo`, `read:org` |
| `lean-ctx` не найден | `cargo install lean-ctx` и `.cargo\bin` в `PATH` |
| `orchestrator-mcp` падает | Не выполнен `npm install` в его каталоге |
| Нужны логи MCP | `%APPDATA%\Claude\logs\mcp-server-*.log` |

## Что не входит

- Сами MCP-серверы: репозиторий содержит конфигурацию, прокси и один собственный сервер, а не вендоренные исходники остальных.
- Секреты: ключи вводятся на машине и не версионируются.
- Кроссплатформенные скрипты: установка и запуск написаны под Windows.
- Автоматическая проверка конфига в CI.

## Статус и ограничения

- **Только Windows.** Скрипты — PowerShell и `.bat`; на macOS/Linux переносится вручную.
- **Нет CI.** Проверки — только локальные: `install.bat`, `setup-config.ps1` и `node --test` в `orchestrator-mcp/`.
- **Лицензия — MIT** (`LICENSE`), но она не покрывает навыки: 16 несут `LICENSE.txt` Anthropic, у остальных своих файлов лицензии нет.
- **42 скилла — не авторские.** Большинство собрано из каталога Anthropic и сторонних наборов; собственных три. Число 42 описывает содержимое папки, а не вклад автора.
- **Расхождение в описании оркестратора.** `package.json` называет его сервером «через DeepSeek API», README — «через RouterAI». Что используется фактически, определяется переменными окружения и настройкой, а не этими строками.
- **Внешние зависимости не ставятся автоматически.** `github-mcp-server.exe` скачивается отдельно, `lean-ctx` требует Rust, `mcp_docker` — запущенного Docker Desktop.
- **Документация ссылается на старое имя.** Команды клонирования указывают `CLaudeConfig`; GitHub перенаправляет на новое имя `claude-desktop-mcp-config`, но ссылки стоит обновить.
- **Конфиг с ключами живёт вне git.** В репозитории только плейсхолдеры; реальный файл генерируется на машине и не версионируется.
- **Два файла конфига в репозитории идентичны.** `claude_desktop_config.json` и `claude_desktop_config.example.json` — одна и та же заготовка с `YOUR_*`; имя без `.example` не означает готовый рабочий конфиг.
- **Проверка ключей ручная.** Ничто не мешает ввести неверный токен: ошибка проявится при первом вызове соответствующего сервера, а не при установке.
- **PowerShell требует обхода политики выполнения.** Документированные команды запускают скрипты с `-ExecutionPolicy Bypass`: без этого политика может заблокировать `setup-config.ps1` и `.ps1`-часть установки.

## Документация

- `INSTALL-NEW-PC.md` — пошаговый перенос сборки на другой компьютер.
- `CLAUDE-SETUP-TASK.md` — самодостаточный промпт для агента, который разворачивает окружение.
- `claude_desktop_config.example.json` — шаблон конфига с 14 серверами и плейсхолдерами.
- `setup-config.ps1` — генерация реального конфига в `%APPDATA%\Claude\`.
- `install.bat` — проверка окружения и установка npm/pip-зависимостей.
- `orchestrator-mcp/` — исходники MCP-сервера, `package.json` и тесты.
- `skills/` — 42 каталога навыков с `SKILL.md`.
- `.claude-plugin/` — манифест плагина и список навыков для установки.
- `ClaudeConfigLiteLLM.yaml` — конфиг LiteLLM для проксирования моделей.
