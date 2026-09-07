# Patterns — архитектурные паттерны на Holst-фасаде

Перенос FigJam-паттернов (figma-tools references, июль 2026) на Holst.
Использует builders из visual-kit.md. Все размеры/позиции после measured
heights — не угадывать.

## 1. Layered / 3-tier

Слои — зоны вертикально; внутри слоя — grid карточек.

```js
const tiers = [
  {name:"Presentation", cards:[["Web App","React"],["Mobile App","RN"]]},
  {name:"Business",     cards:[["API Gateway","tRPC"],["Auth","OAuth"],["Orders","CRUD"]]},
  {name:"Data",         cards:[["PostgreSQL","main"],["Redis","cache"]]},
]
const W = 980, H = 340, GAP = 48
tiers.forEach((t, ti) => {
  const z = zone(t.name, 100, 100 + ti*(H+GAP), W, H)
  t.cards.forEach(([n, note], ci) => {
    const col = ci % 3, row = Math.floor(ci/3)
    svcCard(z, n, note, 32 + col*300, 96 + row*140)
  })
})
// межслойные стрелки — отдельным вызовом, после всех nodeId
```

Межслойные связи: собери handles через `holst.board.findAll(n => n.type==="card")`,
выбери по тексту, `edge()` — затем `getLabelBounds()`-ревью.

## 2. Dependency graph (из ручного описания)

Вход: описание пользователя. Нормализация (показать пользователю при >8 узлах):

```json
{"nodes":[{"id":"gateway","kind":"svc","label":"API Gateway"},
          {"id":"pg","kind":"db","label":"PostgreSQL"},
          {"id":"stripe","kind":"ext","label":"Stripe"}],
 "edges":[{"from":"gateway","to":"pg","label":"SQL"},
          {"from":"gateway","to":"stripe","label":"charges","async":true}],
 "groups":[{"name":"Core","nodes":["gateway","pg"]}]}
```

Layout: доминирующее направление — слева направо; группы = зоны-колонки;
x-колонка по топологической глубине, y — по порядку в группе. Асинхронные
связи — dashed. Циклы в графе: расположить петлевой узел ниже основного
потока, elbow-стрелки с разными endpoint-сторонами (`side:"top"/"bottom"`).

## 3. Pipeline / поток данных

```js
const steps = ["Source","Ingest","Transform","Store","Visualize"]
let prev = null
steps.forEach((name, i) => {
  const z = zone(name, 100 + i*420, 100, 360, 260)   // или tagNode без зон
  const n = svcCard(z, name, "", 32, 96)
  if (prev) edge(prev, n, "")
  prev = n
})
```

## 4. Hub / star (gateway → сервисы)

Частный случай layered: хаб слева, листья колонкой справа. Листья — tagNode
(компактные), хаб — svcCard. Стрелки хаб→лист без подписей, если очевидно.

## 5. Event-driven (очереди, async)

```js
const q = queueNode(z, "Kafka", 480, 120)
edge(producer, q, "publish", {async:true})
edge(q, consumer, "consume", {async:true})
```

Async — dashed независимо от цвета; шина — bpmn-message; топики — отдельные
`bpmn-message`-ноды, не подписи на стрелках.

## 6. Что НЕ переносить из FigJam-практики

- RGB-пресеты, fitShapeToText-костыль (Holst измеряет сам), connector-magnets
  (Arrow endpoints по nodeId), ENG_DATABASE/ENG_QUEUE (аналоги — спец-шейпы).
- Полный дамп диаграммы одним вызовом: Holst-атомарность выгоднее инкрементами.
