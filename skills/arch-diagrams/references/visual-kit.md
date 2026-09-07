# Visual kit — канон блоков арх-диаграмм на Holst

Проверенные builder-скелеты (live, v4.0.0, 2026-09-06). Палитра — см. SKILL.md.
Живой образец: `artifacts/visual-kit.holst` (import_holst) + скриншоты.

## 0. Константы сессии

```js
const P = {
  neutral: {fill:"white6", stroke:"gray8", text:"gray12"},   // компактная метка
  external:{fill:"gray4",  stroke:"gray10", text:"gray12"},  // внешняя система
  arrow:   "gray10", text:"gray12",
}
```

Значения использованы в кит-борде (live). white6 — fill Shape (не Card);
Card fill — токен, но белый/нейтральный вид даёт дефолт без указания fill.

## 1. Zone (зона-контейнер)

```js
function zone(name, x, y, w, h) {
  const f = holst.createFrame({x, y, width:w, height:h, name,
    backgroundStyle:"outline"})
  const t = holst.createText({html:`<p><strong>${name}</strong></p>`, fontSize:20})
  f.appendChild(t); t.update({x:32, y:24})
  return f
}
```

Тинт акцентной зоны: `backgroundStyle:"fill"` + `fill:"blue3"` (или green3/
violet3/orange3) — одна доминирующая зона, не все.

## 2. Node: сервис-карточка (внутри зоны)

```js
function svcCard(frame, name, note, x, y, w=280, icon="⚙") {
  const c = holst.createCard({
    html:`<p><strong>${icon} ${name}</strong></p>${note?`<p>${note}</p>`:""}`,
    width:w})
  frame.appendChild(c); c.update({x, y})
  return c  // height — измеренная; следующая нода: y + c.height + 24
}
```

## 3. Node: компактная метка (Shape)

```js
function tagNode(frame, name, x, y, w=200, preset=P.neutral) {
  const s = holst.createShape({kind:"roundedRectangle", html:`<p>${name}</p>`,
    x:0, y:0, width:w, fixedSize:false, fill:preset.fill,
    strokeColor:preset.stroke, textColor:preset.text})
  frame.appendChild(s); s.update({x, y})
  return s
}
```

Высота измеренная (адаптируется к тексту) — позиционируй соседей по
`s.height`. Для жёстко компактных блоков (бейджи) — иначе: `fixedSize:true`
+ явные width/height (§6).

## 4. Спец-сущности

```js
function dbNode(frame, name, x, y) {          // хранилище
  const s = holst.createShape({kind:"bpmn-data-store", html:`<p>${name}</p>`,
    fill:"white6", strokeColor:"gray8", textColor:"gray12"})
  frame.appendChild(s); s.update({x, y}); return s
}
function queueNode(frame, name, x, y) {       // очередь/шина
  const s = holst.createShape({kind:"bpmn-message", html:`<p>${name}</p>`,
    fill:"white6", strokeColor:"gray8", textColor:"gray12"})
  frame.appendChild(s); s.update({x, y}); return s
}
function externalNode(frame, name, x, y, cloud=false) {  // внешняя система
  const s = holst.createShape({kind: cloud ? "basic-cloud" : "roundedRectangle",
    html:`<p>☁ ${name}</p>`, width:220, fixedSize:!cloud,
    fill:"gray4", strokeColor:"gray10", textColor:"gray12"})
  frame.appendChild(s); s.update({x, y}); return s
}
function personCard(frame, name, x, y) {      // человек/роль
  const c = holst.createCard({html:`<p><strong>👤 ${name}</strong></p>`, width:200})
  frame.appendChild(c); c.update({x, y}); return c
}
```

## 5. Стрелки

```js
function edge(a, b, label, opts={}) {         // a,b — ноды (handles)
  return holst.createArrow({
    start:{nodeId:a.id}, end:{nodeId:b.id},
    arrowType: opts.type || "elbow",
    strokeStyle: opts.async ? "dashed" : "solid",
    strokeColor: opts.highlight || P.arrow,
    labels: label ? [{html:`<p>${label}</p>`, position:0.5}] : []})
}
```

- sync — solid `gray10`; async — **dashed** `gray10`; критический путь —
  `blue9` (opts.highlight), ≤2 на диаграмму.
- Подпись 1–4 слова; длиннее — Card рядом.
- После создания: `arrow.getLabelBounds()` → сравнить с endpoint'ами и
  другими стрелками; коллизия — сменить `position` лейбла (labels
  заменяются целиком), endpoint-`side` или расстояние между нодами.
- Бейджи — Shape с `fixedSize:true` + явные width/height (≈110×40):
  measured-height у Shape больше ожидаемого, два бейджа сливаются.
- Межзонные стрелки: маршрут через коридор между зонами —
  `controls:[{orientation:"vertical", x:<коридор>}]`.
- Ревью стрелок: frame-скриншот их НЕ показывает (стрелки на board-корне) —
  только get_screenshot rect/root.

## 6. Бейдж-статус

```js
function badge(frame, text, token, x, y) {    // ≤2 на зону; компактный
  const s = holst.createShape({kind:"roundedRectangle", html:`<p>${text}</p>`,
    width:110, height:40, fixedSize:true, fill:token,
    strokeColor:"gray8", textColor:"gray12"})
  frame.appendChild(s); s.update({x, y}); return s
}
// green6 active · blue6 planned · orange deprecated · red critical
// (orange/red — по гайду, в кит-борде не отрисованы; стикеры — НЕ бейджи)
```

## 7. Легенда и title-блок

```js
function legend(frame, x, y) {
  const c = holst.createCard({width:260, html:
    "<p><strong>Легенда</strong></p>"
    + "<p>— solid: sync-вызов</p>"
    + "<p>‑ ‑ dashed: async/событие</p>"
    + "<p>🗄 bpmn-data-store · 🔔 bpmn-message</p>"
    + "<p>gray4 = внешняя система</p>"})
  frame.appendChild(c); c.update({x, y}); return c
}
```

Title-блок (имя диаграммы + дата) — первая Card в корне: name + `2026-09-06`,
`gray12`. Card в корне борда рендерится с дефолтной рамкой и читается —
это единственное легитимное исключение из правила «содержательные ноды
в зонах». Provenance-блок в v1 не используется (решение 2026-09-06).

## 8. Композит: мини-пример

```js
const zone1 = zone("Our System", 100, 100, 720, 420)
const api   = svcCard(zone1, "API", "Next.js + tRPC", 32, 96)
const db    = dbNode(zone1, "Postgres", 32, 96 + api.height + 40)
const ext   = externalNode(zone1, "Payment API", 400, 96)
edge(api, db, "SQL")
edge(api, ext, "charges", {async:true})
legend(zone1, 400, 96 + api.height + 40)
// стрелки на board-корне: frame.screenshot() их НЕ покажет —
// ревью только holst_get_screenshot (rect/root)
return {zoneId:zone1.id, nodeIds:[api.id, db.id, ext.id],
        resultLink: holst.getResultLink()}
```
