# Сводка сессии: перенос семантики C# на встроенный `lsp` (форк opencode)

## Цель

Перевести семантические операции по C# с кастомного тула `roslyn` (`roslynq` CLI) на
встроенный инструмент `lsp` форка opencode. Обкатать в изолированной песочнице, **не
трогая глобальную установку** `~/.opencode/bin/opencode`.

## Итог (кратко)

- В форке `lsp`-тул расширен 8 символьными операциями: `structure/types/symbols/members/
  refs/callers/implementations/rename` (+ `rename --apply` через `textDocument/rename`).
- Найдены и исправлены 2 дефекта: слишком короткий дедлайн холодной загрузки и выбор
  probe-файла за пределами рабочего каталога.
- Найден и обойдён баг сборки: Bun `splitting: true` ломает скомпилированный бинарь.
- Проверено на реальном solution `nextorm` и на non-git scratch-проекте.
- Всё незакоммичено; глобальный `opencode` не изменялся.

## Ключевые решения

- **Target = встроенный `lsp`-тул форка**; `roslynq` — legacy-путь, выводится из употребления.
- Глобальный бинарь `~/.opencode/bin/opencode` (от 22.09) НЕ перезаписывать — он не содержит
  symbol-операций (коммит с фичей позже его сборки).
- Всё тестирование — в песочнице `~/sources/opencode-config-sandbox`.
- Багфикс splitting и feature symbol-операций — **разные** issue/PR.

## Два независимых пути (не путать)

| | Что | Где |
|---|---|---|
| LSP | `roslyn-language-server --stdio --autoLoadProjects` — диагностики + база для `lsp`-тула | `lsp.csharp` в `config/opencode.jsonc` |
| Custom tool | `roslyn` → dotnet-тул `roslynq` (отдельный демон) | `config/tools/roslyn.ts`, `config/tools/roslyn-cli/Program.cs` |

## Изменения в форке (uncommitted)

Ветка: `lsp-fix`, трекает `origin/dev`, `ahead 1`
(коммит `ef597fe856` «feat(lsp): symbol operations on the lsp tool»).

- `packages/opencode/src/tool/lsp.ts` — исправления этой сессии:
  1. **Probe-файл не выходит за пределы сессии.** Раньше `fs.glob` шёл от
     `instance.worktree`; для non-git каталога worktree уезжал наверх (в `/tmp`) и цеплял
     чужой `.cs`. Теперь кандидаты — `instance.directory`, затем `instance.worktree`, и файл
     обязан лежать внутри выбранного корня (`candidate.startsWith(probeRoot + path.sep)`).
  2. **Ожидание готовности LSP — 120 с вместо 20 с**, переопределяется
     `OPENCODE_LSP_READY_TIMEOUT_MS`.
- `packages/opencode/script/build.ts` — флаг `--no-splitting` (обход бага сборки).

Прочие ранее добавленные (в коммите) части фичи: `tool/lsp.txt`, `lsp/lsp.ts`
(`rename` на `textDocument/rename`, `searchSymbols`, `implementation`, `incomingCalls`),
`effect/runtime-flags.ts:45` + `tool/registry.ts:247` (гейт `OPENCODE_EXPERIMENTAL_LSP_TOOL`).

## Баг сборки и обход

Bun с `splitting: true` дублирует рантайм `effect` по чанкам → в скомпилированном бинаре
падает декодирование `Parameters`:
`undefined is not an object (evaluating 'node.name')`. Из исходников работает.
**Обход — `--no-splitting`.**

Рабочая команда сборки:

```bash
cd ~/sources/opencode
bun run packages/opencode/script/build.ts --single --skip-embed-web-ui --no-splitting --skip-install
cp packages/opencode/dist/opencode-linux-x64/bin/opencode \
   ~/sources/opencode-config-sandbox/bin/opencode-fork
```

## Изменения в песочнице (`opencode-config-sandbox`)

- `bin/oc-sandbox` — предпочитает `$SANDBOX/bin/opencode-fork`; экспортирует
  `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` и `OPENCODE_DISABLE_AUTOUPDATE=true`.
- `bin/opencode-fork` — собранный бинарь форка (gitignored).
- `config/opencode/opencode.jsonc` — блок `lsp.csharp`, `"autoupdate": false`,
  `"permission": { "lsp": "allow" }`.
- `.gitignore` — добавлен `bin/opencode-fork`.

## Фикс гонки демонов `roslynq` (uncommitted)

`config/tools/roslyn-cli/Program.cs`: `ServeAsync` берёт эксклюзивный OS-лок
(`FileShare.None`) на `<sock>.lock` до загрузки solution; `ConnectAsync` не спавнит дубль
при занятом локе (`LockPath`, `DaemonOwned`). Проверено: 6 параллельных первых вызовов → 1
демон; повторный вызов переиспользует PID; `stop` работает. Пакет пересобран и
переустановлен глобально.

## Замеры (nextorm, одна машина, шумные)

| | `roslynq` | нативный `lsp` (форк) |
|---|---|---|
| Процессов для семантики | отдельный демон + свои MSBuild-узлы | 0 новых (поверх уже запущенного LSP) |
| Память на solution | демон **~558 MB** | **+0**; сервер LSP ~717 MB + BuildHost ~22 MB уже нужен для диагностик |
| Тёплый вызов | ~150 ms (спавн клиента + roundtrip) | ~17–70 ms (в процессе) |
| Первый запрос | холодная загрузка (~80 s worst case) | ~7.5 s (в пределе та же загрузка, но общая с диагностикой) |

Вывод: **память — минус ~0.5 ГБ на solution; латентность — скорее быстрее; исчезает
дублирующий парс того же кода.** Выигрыш есть, только если LSP включён и прогрет.

## Что проверено

- Все 8 операций `lsp` на `nextorm.slnx` через `oc-sandbox debug lsp tool …`.
- `rename` dry-run + `--apply` (4 правки) на scratch git-репо.
- Non-git каталог `/tmp/lsp-nogit`: `types`, `rename` dry-run и `--apply` — после фикса
  probe-файла работает.
- `OPENCODE_LSP_READY_TIMEOUT_MS=60000` принимается.
- `debug config` подтверждает `autoupdate: false`.
- `bun run --cwd packages/opencode typecheck` (`tsgo --noEmit`) — чисто.

## Как тестировать самому

```bash
SB=~/sources/opencode-config-sandbox/bin/oc-sandbox
cd ~/sources/nextorm
export LSP_BENCH_DIR=$PWD          # иначе берётся cwd

$SB debug lsp tool structure --raw
$SB debug lsp tool types --query SqlBuilder --raw
$SB debug lsp tool symbols --symbol NextORM.Core.SqlBuilder --raw
$SB debug lsp tool members --symbol NextORM.Core.SqlBuilder --raw
$SB debug lsp tool refs --symbol NextORM.Core.SqlBuilder.ToCommand --raw
$SB debug lsp tool callers --symbol NextORM.Core.SqlBuilder.Kw --raw
$SB debug lsp tool implementations --symbol NextORM.Core.IDataContext --raw
$SB debug lsp tool rename --symbol NextORM.Core.SqlBuilder.Kw --newName Keyword   # dry-run
$SB debug lsp tool rename --symbol Foo --newName Bar --apply                      # пишет файлы!
```

`debug lsp tool` не спрашивает permission и не тратит модель; `--repeat 5` — тёплые тайминги.

Полный агентский путь: запустить `oc-sandbox` (TUI) и попросить агента использовать `lsp`.

## Открытые вопросы / далее

1. Решить судьбу фикса `--no-splitting`: оформить как отдельный bugfix PR (issue upstream).
2. Для feature symbol-операций — сначала issue с описанием/подходом и одобрение core-команды,
   потом PR.
3. Решить, выводится ли `roslynq`/`roslyn.ts` из конфига после закрепления `lsp` как основного пути.

## Контрибьют (справка)

- Гайд: `CONTRIBUTING.md` (+ style guide `AGENTS.md`).
- **Issue First Policy** (`CONTRIBUTING.md:180`): PR обязан ссылаться на issue.
- Новые фичи — только через design review (`:13`, `:251`).
- Issue заводятся **в апстриме** `anomalyco/opencode` (не в форке) и только по шаблонам
  (Bug report / Feature request / Question); при несоответствии — 2 часа на правку, потом
  авто-закрытие (`:257`–`:263`).
- PR-заголовки в conventional commits (`feat:`, `fix(opencode):`, …); запрещены
  «AI-generated walls of text» (`:204`).
- `upstream` remote локально **не обязателен** — Sync fork и PR можно делать через GitHub UI
  (target base — `anomalyco/opencode`, целевая ветка, скорее всего, `dev`).
