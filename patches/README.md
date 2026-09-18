# patches/ — реестр отличий форка от апстрима

Конвенция: dsh-config `PLUGIN-STACK-REWORK-2026-09-18.md` §1.0.
Один патч = один логический фикс = один файл (format-patch против апстрим-базы).
Ветка с патчами: `dsh-patches` (база: `6a37c052a` = release coding-agents v0.6.1).
`main` форка трекает upstream без изменений.

| Файл | Что делает | Симптом без него | Upstream | Проверка |
|---|---|---|---|---|
| `001-bank-missing-404.patch` | `HindsightClient` отличает «банк ещё не создан» (404 + `Bank … not found` в теле) от «сервер не умеет knowledge pages» (404/405/501 без такого тела). Точки: `tree()`, `getPage()`, seedPages POST/PATCH. Файл: `hindsight-integrations/coding-agents/src/core/hindsight.ts` | На первой сессии любого репо (банк создаётся лениво) `knowledgePagesSupported` лочился в `false` на весь процесс → все page-тулы отвечали `knowledge_pages_unavailable` до рестарта хоста | не отправлен (TODO: PR в vectorize-io/hindsight) | `grep -c bankMissingIn hindsight-integrations/coding-agents/dist/dsh.js` → 5 после `npm run build`; vitest `hindsight.pages.test.ts` зелёный |
| `002-installer-reinstall-empty-list.patch` | `install()` dsh-инсталлера считает leftover `[]` (его оставляет `uninstall()`, т.к. dsh требует top-level array) пустым patch-слоем, а не контентом. Файл: `hindsight-integrations/coding-agents/src/installer.ts` | Реинсталл (skillsync и т.п.) склеивал `[]` + marker-блок → скаляр + sequence = невалидный YAML → **dsh-web crash-loop на буте**. Прод-инцидент 18.09.2026 | не отправлен (TODO: PR в vectorize-io/hindsight) | vitest: тест "re-install after uninstall does not glue our block onto the leftover '[]'" зелёный |
| `003-installer-profile-bundle-skip.patch` | `install()` dsh-инсталлера сканирует `$DSH_HOME/profiles/*/package.json`; если пакет уже в `dsh.profile.bundles` — глобальный insert не пишется, stale-блок снимается, слой сходится к `[]` (или к своим патчам юзера). Файл: `hindsight-integrations/coding-agents/src/installer.ts` | Профиль с bundle + глобальный insert = один loader id дважды → `duplicate loader entry id: hindsight` → **dsh-web crash-loop на буте**. Прод-инцидент 18.09.2026 (повтор после 002 при skillsync) | не отправлен (TODO: PR в vectorize-io/hindsight) | vitest: тесты "defers to a profile bundle that already registers the plugin", "repairs a stale home block once a profile bundle owns registration", "still writes the home block when no profile bundles the plugin" зелёные; `npm test` 1021/1021, `npm run build`, прогон `node dist/installer.js install dsh` на живом `~/.dsh` → "profile web already bundles the plugin — home patch layer kept clear" |

## Merge-ритуал (синк с апстримом)

1. `git fetch upstream && git checkout dsh-patches && git rebase upstream/main` (или merge — по вкусу, но rebase держит патч-коммиты сверху).
2. Разрулить конфликты в `src/`.
3. Регенерировать патч-файлы: `git format-patch <новая база> -o patches/` + переименовать в `NNN-slug.patch`.
4. `npm ci && npm run build && npm test` в `hindsight-integrations/coding-agents`; прогнать колонку «Проверка».
5. Если апстрим принял фикс — патч удаляется из `patches/`, коммит дропается на rebase, запись помечается `UPSTREAMED in vX.Y.Z`.
