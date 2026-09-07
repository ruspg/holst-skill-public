---
name: arch-diagrams
description: Отрисовка архитектурных диаграмм на вайтборде Holst (<HOLST_INSTANCE>) — layered-схемы, context diagrams, dependency graphs, зоны и связи компонентов, легенды, шаблонные визуальные блоки. Use when the user says "архитектура", "арх-схема", "схема системы", "нарисуй архитектуру/систему", "context diagram", "dependency graph", "компоненты и связи", "C4" — работает поверх базового skill holst (MCP-механика), требует запущенного Holst desktop.
---

# Arch-diagrams — архитектурные диаграммы на Holst

Специализация базового skill [[holst]]: НЕ механика MCP (она там), а
архитектурная нотация, палитра по ролям элементов и генераторы диаграмм.
Перед первым бордом прочитай `references/visual-kit.md` — канон блоков;
живой образец: `artifacts/visual-kit.holst` + скриншоты рядом.
Пути `docs/` и `references/` — от корня репо holst-skill.

## Когда что использовать

| Запрос | Инструмент |
|---|---|
| Арх-схема на борде: layered, context, dependency graph | **этот скилл** |
| BPMN процесса, воркшоп, ретро, свободный борд | базовый skill holst |
| Одноразовая схема в чате/доке | Mermaid в markdown |
| Дизайн-макет, презентационный полиш | Figma (skill figma-tools) |
| Канонические данные о системах  | Atlas (`rbank-atlas`) — источник, Holst — визуализация |

Ground truth обязателен: узлы/связи — из слов пользователя, документов или
KB; не изобретай сущности и связи (см. docs/reference-diagrams-connectors.md).
Предположения и proposed-элементы помечай явно (badge или «proposed» в тексте).

## Палитра по ролям (канон, v2)

| Роль | Канон |
|---|---|
| Зона/группа | Frame `outline`, заголовок `gray12`; тинт (`blue3`/`green3`/`violet3`/`orange3`) — только акцент доминирующей зоны |
| Нода с описанием | Card `white3` **внутри зоны** (вне зоны Card имеет дефолтную рамку и читается — годится для title/легенды; содержательные ноды — в зонах) |
| Компактная метка | Shape + Neutral preset (fill `white6`, stroke `gray8`, text `gray12`) |
| Хранилище | Shape `bpmn-data-store`, Neutral preset |
| Очередь/шина | Shape `bpmn-message`, Neutral preset |
| Внешняя система | Shape gray-preset (fill `gray4`); облачная — `basic-cloud`; dashed stroke — опция |
| Человек | Card с emoji-префиксом `👤` |
| Стрелки | `gray10`; **solid = sync, dashed = async**; `blue9`/`red9` — подсветка критического пути, ≤2 на диаграмму |
| Текст | `gray12` везде; иерархия размером/весом, не цветом |
| Бейджи (≤2 на зону) | Shape `fixedSize:true` ≈110×40: `green6` active, `blue6` planned, `orange` deprecated, `red` critical (последние два — по гайду, в ките не отрисованы; стикеры — НЕ бейджи, слишком крупные) |

Иконки: emoji-префиксы в заголовках карточек (`👤 ⚙ 🗄 🔔 ☁ 🧠`), спец-шейпы
как иконки (`bpmn-data-store`, `bpmn-message`, `basic-cloud`). Универсальных
иконок в Holst нет — не пытайся вставить SVG-иконки.

Дисциплина геометрии: контент → `width` → читаем `height` → позиция
следующего. Высоты Text/Card/Task/Code — readonly. Стрелки:
`{start:{nodeId}, end:{nodeId}, arrowType:"elbow"|"straight"|"curved",
labels:[{html, position}], strokeStyle}` — поля start/end/arrowType, не from/to/type.

## Три генератора

### 1. Layered (слои/тиры)

Слои сверху вниз или слева направо; слой = зона; нода = Card white3.
Скелет — `references/patterns.md` §1. Данные: описание пользователя/документ.

### 2. Dependency graph (из ручного описания)

Шаг 1: нормализуй описание в `{nodes:[{id, kind, label, note?}],
edges:[{from, to, label?, async?}], groups?}` — покажи пользователю
нормализованную структуру для подтверждения, если узлов > 8.
Шаг 2: доминирующее направление потока (обычно слева направо), зоны по
группам, стрелки после всех нод. Скелет — `references/patterns.md` §2.

### 3. Context diagram (свободная нотация)

Person 👤, наша система — Card, внешняя — gray Shape (+`basic-cloud`),
хранилище — `bpmn-data-store`, очередь — `bpmn-message`; стрелки с глаголами
(«читает», «публикует»). Нотация и скелет — `references/context-kit.md`.

## Батчинг и контроль

- 1–3 семантических региона на use_holst-вызов; диаграмма из 10+ нод —
  3–6 вызовов: зоны+ноды → стрелки → легенда → скриншот-ревью.
- Стрелки создавать после всех нод (реальные nodeId); большие графы — по
  связным регионам, межрегиональные стрелки в конце.
- После стрелок с подписями: **обязательный гейт** — линтер `scripts/arch_lint.js`
  (тело файла → `code` вызова use_holst): отчёт + готовые `suggestions`
  (reroute/shift/label, каждый валидирован на чистоту коридора). Применяй
  `arrow.update(apply)` / `node.update(apply)`, повторяй линтер до
  0 exact-hits / 0 overlaps / 0 labelHits. `approxHits` — advisory (AUTO-маршруты),
  `crossings` линия×линия — допустимы. Детали — `references/qa-lint.md`.
- **Ревью стрелок — только root/rect-скриншотом**: `frame.screenshot()` не
  включает стрелки, parented к board (проверено в обкатке 2026-09-06).
- Коллизию лейбла чинить позицией: `arrow.update({labels:[{html, position}]})`
  (labels заменяются целиком); искать нужную стрелку — `board.findAll(n =>
  n.type === "arrow")` по тексту `n.labels` — id лейблов из getLabelBounds
  это id подписей, не стрелок.
- Коридоры между зонами — для межзонных маршрутов: `controls:[{orientation:
  "vertical", x: <между зонами>}]` уводит стрелку из зоны столкновений.
- Финал: полный root-скриншот + holistic вердикт; `resultLink` пользователю.
- Ошибки: use_holst атомарен; `COMMIT_OUTCOME_UNKNOWN` → инспекция борда
  новым вызовом до ретрая.

## Референс-борд

`artifacts/visual-kit.holst` — собранный кит (импорт через holst_import_holst).
Обновляй его при изменении канона: пересобрал блоки → export → перезапись
артефакта + скриншотов одним коммитом.
