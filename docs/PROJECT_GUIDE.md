# PROJECT_GUIDE — Nodal.AcadMcp

Гид по репозиторию для разработчика, который впервые открывает проект. Всё ниже основано на реальных файлах репозитория: README.md, deploy.ps1, Directory.Build.props, исходный код в src/ и тесты в tests/. Ничего не выдумано: цифры, имена классов, порты, версии соответствуют коду.

## 1. Суть проекта и стек технологий

**Что это.** Nodal.AcadMcp — это MCP-сервер, который позволяет языковой модели управлять AutoCAD 2027 и AutoCAD Architecture. Проект состоит из двух частей, которые живут в одном процессе AutoCAD (acad.exe) и одном отдельном процессе (сервер):

- **Nodal.AcadMcp.Addin** — адд-ин, загружаемый внутрь AutoCAD как автолоадер-бандл (ApplicationPlugins). В нём живут все 159 инструментов, HTTP-мост внутри AutoCAD, диспетчер вызовов и главная кнопка NODALMCPSTART.
- **Nodal.AcadMcp.Server** — тонкий MCP-прокси (stdio), который пересылает вызовы от MCP-клиента на HTTP-мост внутри AutoCAD. Сервер не владеет определениями инструментов; при запросе списка инструментов он спрашивает адд-ин и пересылает ответ.

**Стек технологий (используется в коде):**

| Компонент | Язык / фреймворк | Целевая платформа | Примечание |
|---|---|---|---|
| Адд-ин | C# | `net10.0-windows` | WPF (`UseWPF=true`) для окна статуса и ленты |
| MCP-сервер | C# | `net10.0` | `ModelContextProtocol` 2.2.0, `Microsoft.Extensions.Hosting` 10.0.0 |
| Контракты | C# | `net10.0` | Общая библиотека между адд-ин и сервер |
| Тесты | C# / xUnit | `net10.0` | Ссылаются только на `Nodal.AcadMcp.Contracts`, без AutoCAD |
| AutoCAD API | `AutoCAD.NET` NuGet | версия 26.0.0 | R26.0 = AutoCAD 2027 |
| MCP | ModelContextProtocol 2.2.0 | — | stdio-транспорт между клиентом и сервером |

**Версии.**
- .NET 10 SDK нужен для сборки (deploy.ps1 ищет его сам: PATH, потом `~\.dotnet`, потом Program Files; `-DotnetPath` переопределяет).
- `TargetFramework` адд-ина — `net10.0-windows`; сервер — `net10.0`.
- Пакет `AutoCAD.NET` версии `26.0.0` (кабл в `Directory.Build.props`: `AcadNetPackageVersion`). Это жесткий границ: AutoCAD 2026 и ранее работают на .NET 8, поэтому поддержка их — это не просто смена версии, а перецелевание.
- Манифест бандла задаёт `SeriesMin="R26.0"` / `SeriesMax="R26.0"` — только AutoCAD 2027.

**Как технически взаимодействует с AutoCAD.**

1. Адд-ин загружается в процесс `acad.exe` при старте AutoCAD (или через `NETLOAD`). В момент загрузки он **не открывает сокет** — регистрирует команды, ленту и выходит. Это безопасно: сокет открывается только после того, как пользователь нажмёт **Start** на ленте **Nodal MCP** или введёт `NODALMCPSTART`.
2. Когда мост стартует, адд-ин поднимает `TcpListener` на loopback (`127.0.0.1`) по порту `DEFAULT_PORT = 48885` (константа `AcadMcpProtocol.DefaultPort`). Если порт занят вторым AutoCAD, берётся эфемерный порт, и реальный номер пишется в файл экземпляра.
3. Генерируется сессионный токен (`RandomNumberGenerator.GetBytes(32)`, 32 байта в hex) и пишется в `%LOCALAPPDATA%\Nodal.AcadMcp\instances\<pid>.json` вместе с `pid`, портом и временем старта. Сервер читает этот файл, чтобы найти запущенный AutoCAD и аутентифицироваться заголовком `X-Nodal-Token`.
4. Любой вызов инструмента маршалится на главную ветку AutoCAD через `Application.Idle` (`MainThread.InvokeAsync` / `MainThread.InCommandContextAsync`). Коснуться AutoCAD API из рабочей ветки — это мгновенный критичный отказ (`eFatalError`), а не исключение, поэтому это не опционально.
5. Каждый изменяющий вызов выполняется внутри `doc.TransactionManager.StartTransaction()` и коммитится как ровно один шаг отмены (`Ctrl+Z`), через блокировку документа (`doc.LockDocument()`). Чтение выполняется с отменой транзакции.
6. Помимо основного HTTP-моста есть `LispBridge` для `run_lisp` (вызов через `acedDefun`, т.к. `Application.Invoke` не доходит до AutoLISP) и `CSharpBridge` для `run_csharp` — компиляция произвольного C# через встроенный в AutoCAD Roslyn (4.10.0.0) в collectible `AssemblyLoadContext`.

**Безопасность.** Мост принимает только запросы с токеном, только на `Host: 127.0.0.1:<port>` (или `localhost:<port>`), отклоняет любой `Origin`/`Referer`, требует `Content-Type: application/json` для POST. Максимальный размер тела запроса — `MaxBodyBytes = 16 * 1024 * 1024` (16 МБ). Ответ всегда HTTP 200 с конвертом `{"ok": true, "result": ...}` или `{"ok": false, "error": {"code", "message", "detail"}}`.

## 2. Карта репозитория

```
.
├── README.md                  # Описание проекта, установка, подключение, каталог из 159 инструментов, ограничения
├── deploy.ps1                 # Сборка и развёртывание обоих halves (PowerShell 5.1 / pwsh 7)
├── Directory.Build.props      # Общие свойства: .NET 10, C# latest, x64, AcadVersion=2027, AutoCAD.NET 26.0.0
├── Nodal.AcadMcp.slnx         # Решение Visual Studio / dotnet
├── docs/                      # Документация
│   ├── PROJECT_GUIDE.md       # этот файл
│   ├── aca-api-lessons.md     # Уроки по API AutoCAD Architecture
│   ├── drafting-lessons.md    # Уроки по 2D черчению и оформлению листов через run_csharp
│   ├── workflow-tools.md      # Восстанавливаемые инструменты рабочих процессов
│   ├── workflow-validation.md # Валидация рабочих процессов
│   └── images/status-window.png
├── skills/                    # Навыки (руководства для моделей)
│   └── aca-modeling/SKILL.md  # Навык моделирования ACA
├── src/                       # Исходный код
│   ├── Nodal.AcadMcp.Addin/               # Адд-ин (внутри acad.exe), net10.0-windows
│   │   ├── AcadMcpApp.cs                  # Точка входа адд-ина (IExtensionApplication, команды NODALMCPSTART/STOP/STATUS)
│   │   ├── Acad.cs                        # AcadContext и Acad.Read/Write — вход к документу, транзакции, отмена
│   │   ├── MainThread.cs                  # Очередь работ на Application.Idle, маршаллинг на главную ветку (DrainBudget = 8)
│   │   ├── HttpBridge.cs                  # HTTP/1.1 сервер на loopback, токен, файл экземпляра, защита (MaxBodyBytes = 16 МБ)
│   │   ├── Router.cs                      # Таблица имён инструментов → обработчиков, Register/Invoke, ToolEnvelope
│   │   ├── ToolRegistry.cs                # Регистрация всех модулей инструментов (RegisterAll, по одной лямбде на модуль)
│   │   ├── LispBridge.cs                  # Вызов AutoLISP (run_lisp)
│   │   ├── CSharpBridge.cs                # Компиляция и выполнение произвольного C# (run_csharp, через Roslyn AutoCAD)
│   │   ├── StatusWindow.cs, RibbonSupport.cs, AcaSupport.cs, WorkflowSupport.cs, Args.cs, AcadMcpException.cs
│   │   └── Tools/                         # Модули инструментов (каждый регистрирует свои команды через Router.Register)
│   │       ├── SessionTools.cs, EntityTools.cs, BlockTools.cs, AnnotationTools.cs
│   │       ├── LayoutTools.cs, SheetTools.cs, PlotTools.cs, EscapeTools.cs
│   │       ├── SolidTools.cs, WorkflowTools.cs, DraftingTools.cs, CSharpBridge.cs (self-register)
│   │       ├── AcaTools.cs, AcaOpeningTools.cs, AcaSpaceTools.cs, AcaSlabRoofTools.cs
│   │       ├── AcaMassTools.cs, AcaGridStructureTools.cs, AcaAssemblyTools.cs, AcaSectionTools.cs
│   │       ├── AcaFabricationTools.cs, AcaProjectTools.cs
│   │       └── ToolRegistry.cs            # единственное место, где все модули подключаются
│   ├── Nodal.AcadMcp.Contracts/           # Общий контракт (без AutoCAD), net10.0
│   │   ├── AcadMcpProtocol.cs             # DefaultPort = 48885, routes (_tools, _ping, _guide), токен, файлы экземпляров, ToolEnvelope, ToolDescriptor, UndoKind
│   │   ├── ToolSchema.cs                  # Схемы аргументов и валидация
│   │   ├── ToolArgs.cs                    # Разбор тела запроса
│   │   ├── HttpRequestGuard.cs            # Проверка входящих HTTP-запросов (токен, Host, Origin/Referer, Content-Type)
│   │   ├── AgentGuide.cs                  # Правила работы (отправляются клиенту как инструкции сервера)
│   │   ├── CommandLineState.cs, CommandText.cs, DraftingChecks.cs, DraftingSymbols.cs
│   │   ├── AnnotationSizing.cs, PdfRaster.cs, ScriptSource.cs, SheetMath.cs, ToolResultImage.cs, WorkflowMath.cs, BridgeCli.cs
│   │   └── Nodal.AcadMcp.Contracts.csproj
│   └── Nodal.AcadMcp.Server/              # MCP-сервер (отдельный процесс), net10.0
│       └── Program.cs                     # Тонкий прокси: ListTools/CallTool через HTTP, CLI-режим (call/tools/ping/guide/help)
├── tests/                       # Тесты
│   ├── Nodal.AcadMcp.Tests/               # Юнит-тесты, net10.0, только Contracts (без AutoCAD)
│   │   ├── AcaLoadSafetyTests.cs, AnnotationSizingTests.cs, BridgeCliTests.cs
│   │   ├── CommandLineStateTests.cs, CommandTextTests.cs, DraftingTests.cs
│   │   ├── HttpRequestGuardTests.cs, PdfRasterTests.cs, ScriptSourceTests.cs
│   │   ├── SheetMathTests.cs, ToolArgsTests.cs, ToolDescriptorTests.cs
│   │   ├── ToolResultImageTests.cs, WorkflowTests.cs
│   │   └── Nodal.AcadMcp.Tests.csproj (xUnit, Microsoft.CodeAnalysis.CSharp 4.10.0)
│   └── integration/                   # Нативные регрессионные тесты (требуют запущенный AutoCAD + мост)
│       ├── README.md
│       └── workflow_regression.py
```

**Ответственность модулей:**
- `AcadMcpApp.cs` — точка входа адд-ина; команды `NODALMCPSTART/STOP/STATUS`, запуск и остановка моста, отслеживание ошибок загрузки.
- `ToolRegistry.cs` — единственное место, где все модули инструментов подключаются (`RegisterAll`); каждая лямбда обёрнута в `Guard`, так что неудача одного модуля не ломает остальные.
- `Router.cs` — диспетчер: имя инструмента → дескриптор + обработчик; `Invoke` валидирует схему и обрабатывает исключения в конверт `ToolEnvelope`.
- `HttpBridge.cs` — HTTP-сервер внутри AutoCAD: accept loop, авторизация, диспатч на `Router`, конверты ответов, защита от браузерных атак.
- `MainThread.cs` — очередь работ на `Application.Idle`, бюджет `DrainBudget = 8` элементов на тик, `InvokeAsync`/`InCommandContextAsync`.
- `Acad.cs` — `Acad.Read`/`Acad.Write` (транзакции, блокировка документа, `ToObjectId` по handle).
- `Program.cs` (Server) — MCP-сервер: stdio-транспорт, `ListTools` запрашивает каталог у адд-ина, `CallTool` пересылает вызов, CLI-режим (`call`, `tools`, `ping`, `guide`, `help`).
- Контракты (`Contracts/`) — общие типы, схемы, защита запросов, правила (`AgentGuide`), без зависимости от AutoCAD.
- Тесты — только `Contracts`, запускаются где угодно (в т.ч. на CI без AutoCAD).

## 3. Жизненный цикл запроса

Пошаговая схема от вызова инструмента ИИ-агентом до выполнения внутри AutoCAD и возврата результата:

1. **MCP-клиент** вызывает инструмент через stdio-транспорт у `Nodal.AcadMcp.Server.exe` (сервер — тонкий прокси, у него своих определений инструментов нет).
2. Сервер находит запущенный AutoCAD: читает `%LOCALAPPDATA%\Nodal.AcadMcp\instances\<pid>.json` — берёт самую свежую запись, чей процесс ещё жив. Порт и токен берёт оттуда (может быть переопределен через `NODAL_ACAD_PORT`).
3. Сервер отправляет `POST http://127.0.0.1:<port>/<tool>` с телом JSON аргументов и заголовком `X-Nodal-Token`. `Host` должен быть loopback; `Origin`/`Referer` недопустимы; `Content-Type: application/json` обязателен.
4. **HttpBridge** (внутри `acad.exe`) принимает соединение, проверяет заголовок через `HttpRequestGuard.Admit` (токен, Host, отсутствие Origin/Referer, Content-Type, размер тела ≤ `MaxBodyBytes` = 16 МБ). При отказе — `403` / `413` / `415`, без раскрытия причины.
5. `HttpBridge.DispatchAsync` разбирает маршрут: `_tools` (список), `_ping` (живо), `_guide` (правила), иначе `POST /<tool>` → `Router.Invoke(route, args)`.
6. **Router** ищет инструмент по имени в таблице `Tools`. Если не найден — `unknown_tool`. Если найден — валидирует `InputSchema` через `ToolSchema.Validate`, затем вызывает обработчик `ToolHandler(JsonElement args)`.
7. Обработчик инструмента выполняет работу с AutoCAD через `Acad.Read` / `Acad.Write` (или `MainThread.InCommandContextAsync` для командного контекста), которые маршалируют делегат на главную ветку AutoCAD через `Application.Idle` (`MainThread.InvokeAsync`). Бюджет дренажа — `DrainBudget = 8` элементов за тик, чтобы burst не замораживал UI.
8. Внутри `Acad.Write` открывается транзакция документа (`doc.TransactionManager.StartTransaction`), документ блокируется (`doc.LockDocument`), выполняется код инструмента, транзакция коммитится — это один шаг отмены (`Ctrl+Z`). Для чтения транзакция отменяется.
9. Результат возвращается через `Router.Invoke`, который оборачивает его в `ToolEnvelope` (`{ ok: true, result }` или `{ ok: false, error: { code, message, detail } }`). Исключения AutoCAD (`Autodesk.AutoCAD.Runtime.Exception`) превращаются в `autocad_error` с `ErrorStatus`.
10. `HttpBridge` сериализует конверт в JSON, отвечает HTTP 200 с `Content-Type: application/json; charset=utf-8`.
11. Сервер читает ответ. Если это не ошибка и в теле есть изображение (`plot_preview`, `export_drawing_preview`), `ToolResultImage.TrySplit` отделяет текст от base64-картинки, и картинка передается MCP-клиенту как `image` контент. Иначе возвращается текст.
12. MCP-клиент отдаёт результат языковой модели.

## 4. Как устроены инструменты (Tools) и навыки (Skills)

**Где зарегистрированы 159 инструментов.** В `src/Nodal.AcadMcp.Addin/Tools/ToolRegistry.cs`, метод `RegisterAll()`. Каждый модуль подключается одной лямбдой (`() => Module.Register()`), обёрнутой в `AcadMcpApp.Guard(name, register)`, так что падение одного модуля ломает только свои инструменты, а не весь адд-ин.

Список модулей (в порядке регистрации):

Plain AutoCAD:
- `SessionTools` — сессия, документы, слои
- `EntityTools` — сущности (создание, запрос, модификация, трансформация, удаление)
- `BlockTools` — блоки, вставки, атрибуты, xref
- `AnnotationTools` — аннотации, размеры, мультилидеры, штриховки, таблицы
- `LayoutTools` — layouts, viewports, выбор, текущий выделенный набор
- `SheetTools` — листы, страницы, поля, свойства чертежа
- `PlotTools` — печать layout в PDF, превью
- `EscapeTools` — `run_csharp`, `run_lisp`, `send_command`, системные переменные, zoom, regen
- `SolidTools` — 3D тела (примитивы, булевы операции, экструзия и т.д.)
- `WorkflowTools` — `checkpoint_drawing`, `export_drawing_preview`, `validate_model`
- `DraftingTools` — чертежные символы, проверка черчения, чтение PDF

AutoCAD Architecture (79 инструментов, работают и на vanilla AutoCAD, возвращая `aca_unavailable`):
- `AcaTools` — объекты AEC, стены, двери, окна, пространства, наборы свойств
- `AcaOpeningTools` — проемы, стили дверей/окон/проемов
- `AcaSpaceTools` — пространства
- `AcaSlabRoofTools` — плиты, кровли
- `AcaMassTools` — массовые элементы
- `AcaGridStructureTools` — сетки колонн, layout-сетки, элементы конструкции
- `AcaAssemblyTools` — curtain wall, окна-сборки, ограждения, лестницы
- `AcaSectionTools` — разрезы и фасады
- `AcaFabricationTools` — изготовление, клоны стилей, профили элементов, MV-блоки
- `AcaProjectTools` — конструктивные проекты
- `LispBridge` — `run_lisp` (через `acedDefun`)

Каждый инструмент регистрируется через `Router.Register(name, description, mutating, [undo], [schema], handler)`. Обработчик — `ToolHandler(JsonElement args)` returning `Task<object?>`. Параметр `undo` задаёт обратимость (`single-step`, `tool-defined`, `not-undoable`, `none`), который сервер превращает в предупреждение для модели.

**Как добавить новый инструмент по аналогии:**
1. Создайте новый файл в `src/Nodal.AcadMcp.Addin/Tools/` (или добавьте в существующий модуль), например `MyTools.cs`, в пространстве имён `Nodal.AcadMcp.Addin.Tools`.
2. Реализуйте `internal static void Register()` и внутри вызовите `Router.Register("имя_инструмента", "описание", mutating: true/false, handler)`, где `handler` — `args => { ... }` returning `Task<object?>`. Работа с чертежом через `Acad.Read` / `Acad.Write` (они сами маршалируют на главную ветку).
3. Если нужна строгая схема аргументов — передайте `ToolSchema.Object(...)` (см. `WorkflowTools` как пример со строгими схемами, и `ToolSchema` в `Contracts/ToolSchema.cs`).
4. Если инструмент относится к ACA — оберните код ACA-типов в `#if ACA` и используйте явные псевдонимы (см. `AcaTools.cs`; символ `ACA` определяется, когда найдена `$(AcadAcaDir)\AecBaseMgd.dll`).
5. Добавьте строку `Module("MyTools", () => MyTools.Register());` в `ToolRegistry.RegisterAll()` — в блок plain AutoCAD или AutoCAD Architecture по смыслу.
6. Соберите: `dotnet build Nodal.AcadMcp.slnx -c Release`. Сервер менять не нужно — он запрашивает каталог инструментов у адд-ина при каждом списке.

**Навыки (Skills).** В папке `skills/` лежат руководства для языковых моделей, а не код. Например, `skills/aca-modeling/SKILL.md` — навык моделирования ACA. Они не регистрируются в коде и не влияют на список инструментов.

## 5. Сборка, запуск и отладка

**Что нужно (только Windows):**
- AutoCAD 2027 (R26.0) — жесткое требование (net10.0-windows + AutoCAD.NET 26.0.0 + R26.0 в манифесте).
- AutoCAD Architecture 2027 — для 79 ACA-инструментов (иначе они видны, но возвращают `aca_unavailable`).
- .NET 10 SDK (не только runtime). deploy.ps1 ищет SDK сам; `-DotnetPath` переопределяет.
- PowerShell 5.1 (встроен в Windows) или pwsh 7.
- Для сборки из исходников: клонировать репозиторий и запустить `powershell -ExecutionPolicy Bypass -File deploy.ps1`.

**Сборка:**
```bash
dotnet build Nodal.AcadMcp.slnx -c Release
```
Сборка адд-ина проходит без установленного AutoCAD (базовый API из NuGet), но ACA-инструменты и `run_csharp` компилируются только при наличии установки AutoCAD (ссылки берутся из `$(AcadInstallDir)` и `$(AcadAcaDir)`), иначе они выключаются (`ACA` / `ROSYLN` символы). Для проверки на машине без AutoCAD:
```bash
dotnet build src/Nodal.AcadMcp.Addin -p:AcadInstallDir="C:\NoSuchDir"
```

**Развёртывание (deploy.ps1):**
```powershell
powershell -ExecutionPolicy Bypass -File deploy.ps1
```
Что делает скрипт:
- Собирает адд-ин и публикует MCP-сервер в `%LOCALAPPDATA%\Nodal.AcadMcp\server`.
- Копирует DLL/PDB/JSON адд-ина в версионированную папку `%APPDATA%\Autodesk\ApplicationPlugins\Nodal.AcadMcp.bundle\Contents\<timestamp>\` (новую папку при каждом деплое, так что запущенный AutoCAD не блокирует пересборку).
- Пишет `PackageContents.xml` с `SeriesMin/Max="R26.0"`, `LoadOnAutoCADStartup="true"` (модуль загружается при старте, но мост не открывается — это безопасно).
- Опционально `-Package` формирует самодостаточный (`--self-contained`) выпуск в `artifacts/Nodal.AcadMcp-acad2027-<timestamp>.zip`.
- Ключи: `-NoBuild` (развернуть уже собранное), `-Package` (также сделать zip), `-Configuration Debug`, `-DotnetPath`.

После развёртывания: перезапустить AutoCAD, затем в AutoCAD нажать **Start** на ленте **Nodal MCP** или ввести `NODALMCPSTART`. Мост откроется на порту 48885 (или эфемерном, если занят) и напечатает число инструментов.

**Подключение к AutoCAD:**
- MCP-клиент: зарегистрировать `%LOCALAPPDATA%\Nodal.AcadMcp\server\Nodal.AcadMcp.Server.exe` как stdio-сервер (Claude Desktop `claude_desktop_config.json` или Claude Code `claude mcp add --scope user nodal-acad -- "<путь>"`).
- Без MCP-клиентов: `Nodal.AcadMcp.Server.exe call <инструмент> @args.json`, `tools` выводит список, `help` — справку, `guide` — правила работы.
- Raw HTTP: `POST http://127.0.0.1:<порт>/<инструмент>` с заголовком `X-Nodal-Token` из файла экземпляра `%LOCALAPPDATA%\Nodal.AcadMcp\instances\<pid>.json`.

**Запуск тестов:**
```bash
dotnet test tests/Nodal.AcadMcp.Tests
```
Тесты ссылаются только на `Nodal.AcadMcp.Contracts`, поэтому запускаются на любой машине, включая CI без AutoCAD. Они покрывают проверку входящих запросов (`HttpRequestGuard`) — если трогаете `HttpRequestGuard`, эти тесты — контракт.

Нативные регрессионные тесты (`tests/integration/workflow_regression.py`) требуют запущенного AutoCAD 2027 с загруженным адд-ином и открытым мостом; запускаются против свежего scratch DWG.

## 6. Точки входа для новичков

Чтобы быстро понять логику, прочтите в этом порядке:

1. **`src/Nodal.AcadMcp.Addin/Tools/ToolRegistry.cs`** — единое место регистрации всех 159 инструментов; показывает, как модули подключаются и как добавить новый.
2. **`src/Nodal.AcadMcp.Addin/Router.cs`** — диспетчер инструментов: таблица имён → обработчики, валидация схемы, конверт `ToolEnvelope`, обработка ошибок.
3. **`src/Nodal.AcadMcp.Addin/HttpBridge.cs`** — HTTP-мост внутри AutoCAD: порт 48885, токен, файл экземпляра, защита (MaxBodyBytes = 16 МБ), диспатч на `Router`.
4. **`src/Nodal.AcadMcp.Addin/AcadMcpApp.cs`** — точка входа адд-ина: команды `NODALMCPSTART/STOP/STATUS`, запуск моста, отслеживание ошибок загрузки.
5. **`src/Nodal.AcadMcp.Addin/Acad.cs`** — `Acad.Read`/`Acad.Write`: как код попадает на главную ветку AutoCAD и выполняется внутри одной транзакции (один `Ctrl+Z`).
6. **`src/Nodal.AcadMcp.Server/Program.cs`** — тонкий MCP-прокси: `ListTools` запрашивает каталог у адд-ина, `CallTool` пересылает вызов через HTTP.

Дополнительно, по месту: `Tools/EntityTools.cs` (пример типичного модуля инструментов), `Tools/CSharpBridge.cs` (арбитражный escape hatch `run_csharp`), `Tools/WorkflowTools.cs` (пример со строгими схемами `ToolSchema`), `Contracts/AcadMcpProtocol.cs` (порт 48885, routes, токен, файлы экземпляров), `Contracts/HttpRequestGuard.cs` (правила приёма запросов), `tests/Nodal.AcadMcp.Tests/HttpRequestGuardTests.cs` (контракт защиты).
