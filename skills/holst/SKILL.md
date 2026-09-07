---
name: holst
description: Корпоративный вайтборд Holst (<HOLST_INSTANCE>) через MCP — борды, диаграммы, BPMN, flowchart, mind map, канбан, стикеры, скриншоты, экспорт. Use when the user says "борд", "доска", "Holst", "whiteboard", "нарисуй диаграмму/схему/флоу", "BPMN", "mind map", "воркшоп/ретро-борд", "стикеры", "визуализируй процесс" — or asks to read/screenshot/export an existing Holst board.
---

# Holst — вайтборд  через MCP

Holst — корпоративный вайтборд (on-prem holst.so, аналог Miro). MCP-сервер
`holst-mcp` живёт **в desktop-приложении** (`http://localhost:47641/mcp`) и
проксирует в его UI. Тулы `holst_*` уже подключены в opencode; этот скилл —
как ими пользоваться без граблей.

## Когда Holst, а когда другое

| Запрос | Инструмент |
|---|---|
| Persistent борд, воркшоп, ретро, collaborative-артефакт | **Holst** |
| Архитектурная схема: layered, context, dependency graph | **skill arch-diagrams** (поверх этого) |
| Быстрая схема в чате/доке, не требует правок людьми | Mermaid в markdown |
| Канонический BPMN-процесс  (поиск, метаданные) | Atlas (`rbank-atlas`), Holst — только визуализация |
| UI-дизайн, макеты | Figma |

## Предусловия

1. Desktop Holst запущен и залогинен (проверка: `lsof -iTCP:47641 -sTCP:LISTEN`).
2. Если недавно был веб-логин на <HOLST_INSTANCE> — MCP-состояние
   сброшено: первый вызов даст `OPERATION_CANCELLED "account changed"`,
   повторный вызов проходит.
3. Guest/read-only: доступны только `resolve_board`, `list_workspaces`,
   `get_guide`.

## Golden workflow

```
resolve_board(url?) → нет борда → create_board({name})
   → use_holst({boardId, code, description})  [1–6 атомарных вызовов]
   → [диаграммы] после стрелок — ГЕЙТ: линтер skills/arch-diagrams/
     references/qa-lint.md (scripts/arch_lint.js) → suggestions → повтор
     до 0 exact-hits / 0 overlaps / 0 labelHits
   → get_screenshot / node.screenshot() внутри use_holst
   → вернуть resultLink ровно один раз
```

- **До первой мутации** прочитай `docs/guide-holst-use-workflow.md`;
  полный API — `docs/holst-use-api.d.ts` (или `holst_get_guide`).
- Целься в **1–3 завершённых семантических региона** на вызов, не «весь борд разом».
- ID между вызовами не выдумывай: reacquire через `holst.getNodeById(id)`;
  поиск — `holst.board.findAll(n => ...)` с guard `"text" in n`.
- Возврат — plain JSON (`{createdIds:[...]}`); в финальном мутирующем вызове
  добавь `resultLink: holst.getResultLink()` (или `getResultLink(node.id)`
  для одного standalone-результата) и покажи ссылку пользователю один раз.

## Контракты facade (проверено live, v4.0.0)

- Высота `Text/Card/Task/Code` — **readonly**, выводится из контента.
  Порядок: контент → width → читаем `node.height` → позиционируем следующее.
- Дети держат координаты **локально относительно ближайшего Frame**;
  после `appendChild()` явно `node.update({x, y})`.
- Arrow: `holst.createArrow({start:{nodeId}, end:{nodeId}, arrowType:"elbow"|"straight"|"curved",
  labels:[{html:"<p>OK</p>", position:0.5}]})` — поля `start/end/arrowType`,
  НЕ `from/to/type`.
- Shape: `holst.createShape({kind:"bpmn-task", html:"...", x, y})`; у wrapped-фигур
  передай `width` + `fixedSize:false`, затем читай height.
- Неизвестные поля facade **молча игнорирует** (опечатка ≠ ошибка) —
  сверяй имена с d.ts, читай back свойство, если важно.
- `await` обязателен для `createLinkPreview`, `file.setDisplayMode`,
  `extractPages`, `loadFontAsync`, `node.screenshot()`.
- Атомарность: упавший вызов не публикует мутации и отбрасывает свои
  скриншоты. `COMMIT_OUTCOME_UNKNOWN` → сначала inspect фактического
  состояния новым вызовом, потом решай о ретрае.
- Ноды: frame, text, card, sticker, shape, arrow, drawing, table, image,
  group, task, kanban, mind-map-node, flip-card, link-preview, code, stamp,
  file, dice, spinner-wheel. BPMN-набор полный: `bpmn-task`, `bpmn-gateway`,
  события, pool, data-store (см. d.ts `HolstShapeKind`/`BpmnIcon`).
- Контент: `node.text` — plain-чтение; `node.richText` — html-чтение
  (readonly); запись — `node.setRichText(html)` (RichTextNode: text, card,
  task, sticker, shape…).
- Удаление объектов: `node.remove()` — удаляет ноду и её application-owned
  потомков (проверено live). Целого борда это не касается (см. Грабли).
- Точные размеры Shape: для компактных блоков (бейджи, метки) —
  `fixedSize:true` + явные width/height; при `fixedSize:false` measured
  height выходит больше ожидаемого — читай после создания, не угадывай.
- Границы скриншота: `frame.screenshot()` рендерит только поддерево фрейма —
  стрелки, созданные на board-корне (дефолт), в него не попадают. Ревью
  стрелок — `holst_get_screenshot` (root/rect).

## Проверенные скелеты

Frame с контентом (высоты измеряются, не угадываются):

```js
const frame = holst.createFrame({x:100, y:100, width:1200, height:800, name:"..."})
const title = holst.createText({html:"<p><strong>Заголовок</strong></p>", fontSize:32})
const card  = holst.createCard({html:"<p><strong>Пункт:</strong> текст</p>", width:420})
frame.appendChild(title); frame.appendChild(card)
title.update({x:48, y:40})
card.update({x:48, y:title.y + title.height + 24})
await frame.screenshot()
return {frameId:frame.id, ids:[title.id, card.id]}
```

BPMN-флоу (task → gateway → task):

```js
const t1 = holst.createShape({kind:"bpmn-task", html:"<p>Собрать данные</p>", x:100, y:900})
const g1 = holst.createShape({kind:"bpmn-gateway", x:340, y:900, width:80, height:80})
const t2 = holst.createShape({kind:"bpmn-task", html:"<p>Отправить</p>", x:500, y:900})
holst.createArrow({start:{nodeId:t1.id}, end:{nodeId:g1.id}, arrowType:"elbow"})
holst.createArrow({start:{nodeId:g1.id}, end:{nodeId:t2.id}, arrowType:"elbow",
                   labels:[{html:"<p>OK</p>", position:0.5}]})
return {ids:[t1.id, g1.id, t2.id]}
```

Картинка из upload (`holst_upload_assets` → POST файла на submitUrl → assetId):

```js
const img = holst.createImage({assetId:"<assetId>", x:640, y:1000, width:240})
return {imageId:img.id, w:img.width, h:img.height}
```

Rewrite существующего контента:

```js
const matches = holst.board.findAll(n => n.type === "card" && /draft/iu.test(n.text))
for (const card of matches) card.setRichText(card.richText.replace(/draft/giu, "ready"))
return {updatedIds: matches.map(n => n.id)}
```

## Ассеты, экспорт, бэкап

- `holst_upload_assets({boardId, count})` → POST файла (≤50 MiB, multipart
  `file`) на каждый `submitUrl` → `{assetId}` → `createImage`/`createFile`.
  URL single-use, живут 10 минут, не публиковать.
- `holst_download_assets({boardId, scope, format:"png"|"jpeg"|"svg"|"pdf"})` —
  рендер scope (`viewport`/`root{rootObjectId}`/`rect{rect}`) → download URL
  (10 минут, скачать сразу).
- `holst_export_holst` / `holst_import_holst` — нативный бэкап `.holst`
  (ZIP: data.json + PNG-ассеты). Import-POST — foreground-blocking,
  timeout ≥ `requestTimeoutMs` (600 s), не бэкграундить.

## Грабли

- MCP-тула удаления/трэша **целого борда** нет — через UI; объекты на борде
  удаляются `node.remove()` внутри use_holst.
- Порт 47641 — дефолт из бандла, не гарантирован между версиями десктопа.
- Веб-версия — просмотрщик: MCP-созданные борды появляются в Recent после
  первого открытия; прямые `/gapi`-вызовы без токена дают 401 (публичного
  API у Holst нет — не строить на нём автоматизацию).
- Центральный деплой MCP на gateway **невозможен и не нужен**: сервер —
  прокси над локальным UI десктопа (подробно — KB, services/collab/holst.md).

## Обслуживание скилла

- Доки снапшотятся из MCP: `python3 scripts/holst_client.py dump-docs docs`
  (нужен запущенный десктоп). При обновлении Holst — переснимок + правка дат.
- Сервисный контекст и web-API карта: `team-kb/services/collab/holst.md`.
