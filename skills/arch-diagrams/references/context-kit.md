# Context kit — свободная нотация context-диаграмм

Решение 2026-09-06: БЕЗ строгой C4-терминологии (Q11=B) — общий словарь
«кто с чем взаимодействует». Уровни C4 при желании выражаются этой же
нотацией: L1 (context) — человек и системы; L2 (containers) — те же
примитивы внутри зоны системы.

## Нотация

| Сущность | Канон |
|---|---|
| Человек/роль | Card `👤 Имя`, width 200 |
| Наша система | Card `⚙ Система` + note (стек/назначение), white3 в зоне |
| Внешняя система | Shape gray-preset `☁ Имя`; облачная — `basic-cloud` |
| Хранилище | Shape `bpmn-data-store` |
| Очередь/шина | Shape `bpmn-message` |
| Связь | Arrow + глагол («читает», «публикует», «вызывает»); solid=sync, dashed=async |

Границы: зона «Наша система» — Frame outline; внешние НЕ внутри зоны
(снаружи, gray). Легенда обязательна (builder из visual-kit.md §7).

## Скелет

```js
const root = zone("<Система> — context", 100, 100, 1400, 760)
const user = personCard(root, "Инженер", 32, 120)
const sys  = svcCard(root, "Core System", "назначение, стек", 320, 120)
const db   = dbNode(root, "Postgres", 320, 120 + sys.height + 40)
const ext  = externalNode(root, "3rd-party API", 720, 120)
const ext2 = externalNode(root, "Cloud Storage", 720, 280, true)

edge(user, sys, "работает в")
edge(sys, db, "SQL")
edge(sys, ext, "calls", {async:true})
edge(sys, ext2, "sync", {async:true})
legend(root, 720, 440)

const shot = await root.screenshot()
return {rootId: root.id, ids:[user.id, sys.id, db.id, ext.id, ext2.id]}
```

## Правила чтения борда зрителем

- Серое = не наше (зона ответственности вне нас), dashed = асинхронность.
- Глагол на стрелке важнее цвета; цвет стрелок — только подсветка критического
  пути (`blue9`/`red9`, ≤2).
- Много внешних систем (>5) — сгруппировать внешних в одну зону-колонку.
